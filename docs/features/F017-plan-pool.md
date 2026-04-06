---
feature_ids: [F017]
related_features: [F001, F011, F012, F013]
topics: [plan-pool, quota-sharing, subscription, peak-shaving, cli-bridge, provider-verification]
doc_kind: spec
created: 2026-04-06
layer: application
owner_module: plan-pool
status: spec
phase: 1.5
depends_on:
  - { id: F001, type: blocking }
  - { id: F012, type: blocking }
  - { id: F013, type: related, note: "F013-lite（open 注册模式）建议前置实现" }
  - { id: F011, type: related, note: "bridge 分层模式可复用" }
evidence: []
roundtable: docs/discussions/F017-plan-pool-roundtable.md
---

# F017: Plan Pool — 订阅计划池化与削峰填谷

> Status: spec | Owner: TBD
> Roundtable: 2026-04-06（opus / codex / gemini / antig-opus）
> 铲屎官裁定：优先做 non-API Plan；只做 Chat 模式

## Why

AI 订阅计划（Claude Max、ChatGPT Plus、Kimi Pro 等）绑定到个人账号和设备，存在两个浪费场景：

1. **团队号池**（场景 A）：多人购买了订阅但使用节奏不同——有人在忙时额度不够用，有人休假时额度白白浪费。
2. **个人多设备**（场景 B）：同一人在办公室和实验室各有订阅，希望无感合并使用，不需要切换账号。

如果只是做 API key 转发，一个 Nginx 反代就够了。**Agent Mesh 的差异化价值在于桥接那些不提供 API 的订阅计划**——通过 CLI 封装将"只能在本地用的订阅"变成可池化的 A2A 节点。

**为什么只做 Chat 模式**：远端 Coding 意义不大——调用方如果在 Claude 等 IDE 中使用，本地就有 Coding 能力，不需要远端做。Chat 模式（`--tools ""`）同时也是安全隔离最强的模式。

铲屎官原话："把 API 直接给人有信任问题也不好控制量，用 Agent Mesh 的场景通过贡献额度来获取额度就比较公平。"

## What

### 核心概念

| 概念 | 定义 |
|------|------|
| **plan-bridge** | 运行在 Plan 订阅所在机器上的桥接器，将一个订阅封装为标准 A2A 节点。持有本地凭证（CLI 登录态 / API key），不暴露给任何外部调用方。**仅支持 Chat 模式**（纯对话，无工具能力）。 |
| **pool-node** | 虚拟路由节点，对上游暴露统一的 A2A card，对下游管理多个 plan-bridge。负责请求分发、failover、记账。可跑在任意可达的机器上（如 Hub 同机或 NAS）。 |
| **provider** | 实际提供 AI 能力的服务商（Claude / GPT / GLM / Kimi 等）。每个 plan-bridge 声明自己的 provider 类型。 |

### 部署拓扑

```
场景 B（个人两台电脑 + NAS）:

┌─ 办公室电脑 ──────────────┐   ┌─ 实验室电脑 ─────────────┐
│ plan-bridge-A              │   │ plan-bridge-B             │
│ provider: claude           │   │ provider: claude          │
│ maxConcurrency: 3          │   │ maxConcurrency: 3         │
│ (CLI authenticated here)   │   │ (CLI authenticated here)  │
└──────────┬─────────────────┘   └──────────┬───────────────┘
           │                                │
           └──────────┬─────────────────────┘
                      │ (via Hub Relay)
              ┌───────┴────────┐
              │  pool-node     │  ← NAS 或 Hub 同机
              │  (路由 + 记账)  │
              └───────┬────────┘
                      │
              ┌───────┴────────┐
              │  Mesh Hub      │
              └───────┬────────┘
                      │
              ┌───────┴────────┐
              │  Caller        │  ← 用户的 Agent / CLI
              └────────────────┘

数据流：Caller → Hub → pool-node → Hub → plan-bridge → Provider
```

**bridge locality 原则**：plan-bridge 必须跑在 Plan 订阅所在的机器上。凭证（CLI 登录态 / API key）不离开本地。

### plan-bridge 类型

| bridge 类型 | A2A agentId 示例 | 能力 |
|-------------|-----------------|------|
| `cli-chat-bridge` | `claude-chat-office` | CLI 封装，纯对话（`--tools ""`） |
| `api-proxy-bridge` | `gpt-api-lab` | API key 代理转发 |

#### CLI 封装型（non-API Plan，MVP 优先）

适用于不提供 API 的订阅（如 Claude Max Coding Plan）。

```bash
# Chat 模式（唯一模式）
claude -p \
  --session-id "<caller-bound-uuid>" \
  --output-format stream-json \
  --setting-sources ""         \  # 跳过用户 hooks/插件
  --strict-mcp-config          \  # 跳过 MCP 服务器
  --tools ""                   \  # 禁用所有工具（纯对话）
  --max-budget-usd 0.5        \  # 每请求预算上限
  "your prompt here"

# 恢复上下文（prompt 缓存，节省 token）
claude --resume <session-id>
```

**Session 管理**：每个调用方映射独立的 named session（session-id = UUID），天然隔离上下文。`--resume` 支持跨请求保持上下文连贯。

**Session 回收**：固定 TTL，bridge 启动时可配置（`sessionTTLMinutes`，默认 30）。参考 magno73/claude-cli-bridge 的 30 分钟过期策略——这是业界最常见的做法。

**并发控制**：每个 provider 的并发上限由 plan-bridge 安装时配置（`maxConcurrency`，默认 3）。此为硬限制，不可被调用方覆盖。超出并发上限返回 `RATE_LIMITED`。

#### API 代理型

适用于提供 API key 的订阅。HTTP 代理转发，bridge 持有 key 不暴露。

### 安全设计

plan-bridge **仅支持 Chat 模式**（`--tools ""`），这是最强的安全隔离——即使 prompt 注入成功，也无法执行任何本地操作。

#### 进程安全

```typescript
// plan-bridge-cli: 安全进程创建模板
const child = spawn('claude', args, {
  shell: false,                       // 禁止 shell 解释（防命令注入）
  env: {                              // 最小化环境变量
    HOME: sandboxHome,                // 隔离的 home 目录
    PATH: '/usr/bin:/usr/local/bin',  // 最小 PATH
    LANG: 'en_US.UTF-8',
    // 显式不传：API keys, tokens, SSH keys, AWS creds...
  },
  cwd: isolatedWorkDir,              // 隔离工作目录
  timeout: 120_000,                   // 2 分钟硬超时
})
// 客户端断开时自动 kill 子进程（防止僵尸进程）
req.on('close', () => child.kill('SIGTERM'))
```

#### 安全纵深

| 防护层 | 措施 | 防护目标 |
|--------|------|---------|
| L0 能力隔离 | `--tools ""`（禁用所有工具） | 即使注入成功也无法执行任何本地操作 |
| L1 进程隔离 | `spawn(shell:false)` + 最小化 env + 隔离 cwd | 防命令注入 + 防凭证泄露 |
| L2 插件隔离 | `--setting-sources "" --strict-mcp-config` | 防恶意 hook/MCP 注入 |
| L3 额度熔断 | `--max-budget-usd` + maxConcurrency 硬限制 | 防无限循环烧钱 |
| L4 超时强杀 | configurable 超时 + 客户端断开时 SIGTERM | 防僵尸进程 |

### Provider 验真机制（ProviderAttestation）

plan-bridge 注册时声明 provider 类型（如 `claude`、`gpt`、`glm`、`kimi`）。为防止模型套壳（声称是 GPT-4o 实际跑的是更便宜的模型），采用多层验真：

#### 验真层次

| 层次 | 策略 | 时机 | 适用场景 |
|------|------|------|---------|
| L1 | **二进制来源校验**：固定绝对路径 + SHA-256 hash + 版本探针 | 启动时 + 周期性（默认 1h） | CLI 封装型 |
| L2 | **响应校验**：从 stream-json `model` 字段或 API `x-model` header 提取实际模型 | 每次响应 | 全部 |
| L3 | **Probe 指纹**：定期发送已知 prompt，通过响应特征判断模型身份 | 周期性抽检 | Phase 2 |

#### MVP 实现

```typescript
interface ProviderAttestation {
  // L1: 二进制校验
  binaryPath: string           // /usr/local/bin/claude（固定绝对路径）
  binaryHash: string           // SHA-256 of binary
  version: string              // claude --version 输出
  // L2: 响应校验
  claimed: string              // bridge 声明的 provider
  observed: string | null      // 从响应中观测到的 provider
  verified: boolean            // 是否一致
  // 元数据
  fingerprint: string          // 组合指纹（path + hash + version）
  lastVerified: number         // 上次校验时间
  verifyInterval: number       // 校验间隔（默认 3600000ms = 1h）
}

interface PlanBridgeConfig {
  provider: 'claude' | 'gpt' | 'glm' | 'kimi' | string
  providerModel?: string
  maxConcurrency: number        // 硬限制，安装时配置，调用方不可覆盖
  bridgeType: 'cli' | 'api'
  sessionTTLMinutes: number     // Session 回收 TTL（默认 30，启动时可配置）
  maxBudgetPerRequest: number   // 每请求预算上限（USD，默认 0.5）
}
```

**MVP 行为**：
- 启动时校验二进制 hash + version（L1）
- 每次响应解析实际 model（L2）
- claimed vs observed 不一致时标记 `verified: false`，记录到 ledger 并告警
- MVP 不自动禁用（留给管理者判断），Phase 2 可配置自动降权

### pool-node 核心逻辑

```typescript
class PoolNode {
  private bridges: Map<string, BridgeEntry>
  private ledger: SQLiteLedger
  private roundRobinIndex: number = 0

  async invoke(req: InvokeRequest): Promise<InvokeResponse> {
    const available = this.getAvailableBridges()
    if (available.length === 0) {
      throw new MeshError('RATE_LIMITED', 'All bridges busy or unavailable')
    }

    // Round-robin 选择
    const bridge = available[this.roundRobinIndex++ % available.length]

    try {
      const result = await this.meshClient.invoke(bridge.agentId, req)

      // 验真：从响应中提取 provider 信息
      const observedModel = this.extractModel(result)
      this.verifyProvider(bridge, observedModel)

      // 从 stream-json 输出中提取 token 消耗
      const usage = this.extractUsage(result)

      // 记账（含 token 和金额消耗）
      this.ledger.record({
        sourceNodeId: req.sourceNodeId,
        bridgeId: bridge.id,
        provider: bridge.config.provider,
        observedModel,
        inputTokens: usage.inputTokens,
        outputTokens: usage.outputTokens,
        costUsd: usage.costUsd,
        timestamp: Date.now(),
      })

      return result
    } catch (e) {
      if (e.code === 'RATE_LIMITED' || e.code === 'SESSION_BUSY') {
        return this.failover(req, bridge)
      }
      throw e
    }
  }

  private getAvailableBridges(): BridgeEntry[] {
    return [...this.bridges.values()]
      .filter(b => b.activeSessions < b.config.maxConcurrency)
      .filter(b => b.healthy)
  }

  // 从 CLI stream-json 输出提取 token 消耗
  // 参考：Cat Cafe 每次调用 Claude 都记录了上下文数量和消耗金额
  private extractUsage(result: InvokeResponse): UsageInfo {
    // stream-json 最后一条消息包含 usage 信息
    // { inputTokens: number, outputTokens: number, costUsd?: number }
    return parseUsageFromStreamJson(result.body)
  }
}
```

### Ledger（记账）

```sql
CREATE TABLE invocations (
  id              INTEGER PRIMARY KEY AUTOINCREMENT,
  source_node     TEXT NOT NULL,       -- 调用方 nodeId
  bridge_id       TEXT NOT NULL,       -- 目标 plan-bridge
  provider        TEXT NOT NULL,       -- 声明的 provider
  observed_model  TEXT,                -- 实际观测到的 model
  verified        BOOLEAN DEFAULT 1,   -- provider 验真结果
  input_tokens    INTEGER DEFAULT 0,   -- 输入 token 数
  output_tokens   INTEGER DEFAULT 0,   -- 输出 token 数
  cost_usd        REAL DEFAULT 0,      -- 本次消耗金额（USD）
  status          TEXT NOT NULL,       -- success / error / rate_limited
  created_at      INTEGER NOT NULL     -- Unix timestamp ms
);

CREATE INDEX idx_source ON invocations(source_node, created_at);
CREATE INDEX idx_bridge ON invocations(bridge_id, created_at);
```

**每次调用都记录 token 消耗和金额**，为 Phase 2 的贡献额度计算打基础：
- CLI 的 `stream-json` 输出包含 `inputTokens` / `outputTokens` 信息
- API 响应的 `usage` 字段包含 token 计数
- 金额可从 token 数 × 模型定价计算，或从 provider 返回的 cost 信息直接获取
- 即使 MVP 不做贡献权重公式，账本数据已经足够支撑未来计算

## F013-lite（前置最小切面）

当前 Hub 静态白名单制要求每加一个 plan-bridge 都要改 config + 重启。F017 的 plan-bridge 需要动态增减，因此建议前置 F013 的最小切面：

### 改动范围

1. **Hub config 新增 `registrationMode`**：`"whitelist"` (默认) | `"open"`
2. **HELLO handler 扩展**：`open` 模式下，未知 nodeId 自动注册（Trust-on-First-Use）
3. **HELLO body 新增可选 `capabilities` 字段**：节点自报能力
4. **安全约束**：TOFU 后 publicKey 绑定不可更改；动态注册节点 maxScopes 有上限

### 不做

- `approval` 注册模式（Phase 2）
- 节点信息持久化（重启后需重新注册，MVP 可接受）
- 管理审批 UX

预计改动：hub.ts HELLO handler + config schema，约 50 行。

## Acceptance Criteria

### plan-bridge

- [ ] AC-1: CLI 封装型 plan-bridge 可通过 `claude -p --session-id --output-format stream-json --tools ""` 封装 non-API Plan 为纯对话 A2A 节点。
- [ ] AC-2: 每个调用方映射独立的 named session（UUID），上下文隔离。
- [ ] AC-3: 并发上限由 `maxConcurrency` 配置（默认 3），超限返回 `RATE_LIMITED`。此为硬限制，不可被调用方覆盖。
- [ ] AC-4: API 代理型 plan-bridge 通过 HTTP 转发实现，key 不暴露给调用方。
- [ ] AC-5: plan-bridge 注册时声明 `provider` 和 `providerModel`。
- [ ] AC-19: Session TTL 在 bridge 启动时可配置（`sessionTTLMinutes`，默认 30），过期 session 自动回收。

### 安全

- [ ] AC-14: `--tools ""` 生效——模型无法执行任何本地工具操作。
- [ ] AC-15: 跨调用方 session 隔离——不同调用方的 session-id 不串线，上下文完全独立。
- [ ] AC-16: 超预算（`--max-budget-usd`）或超并发（`maxConcurrency`）时返回 429 + `RATE_LIMITED`。
- [ ] AC-17: CLI 进程使用 `spawn(shell: false)` + 最小化环境变量 + 隔离工作目录。
- [ ] AC-18: provider 二进制 hash 校验在启动时执行，不一致时拒绝启动并记录告警。

### pool-node

- [ ] AC-6: pool-node 对调用方暴露统一 A2A card（如 `agentId: "claude-pool"`），调用方不感知底层多个 bridge。
- [ ] AC-7: 请求分发策略：round-robin + 429/SESSION_BUSY 自动 failover 到下一个 bridge。
- [ ] AC-8: 所有 bridge 不可用时返回 `RATE_LIMITED` 给调用方。
- [ ] AC-9: SQLite ledger 记录每次调用，**含 token 消耗和金额**（source_node / bridge_id / provider / observed_model / input_tokens / output_tokens / cost_usd / status / timestamp）。
- [ ] AC-10: provider 验真——从响应中提取实际 model，与声明不一致时标记 `verified: false` 并记录。

### F013-lite

- [ ] AC-11: Hub 配置 `registrationMode: "open"` 后，新节点凭 nodeId + publicKey 可 HELLO 自注册。
- [ ] AC-12: `whitelist` 模式行为不变（向后兼容）。
- [ ] AC-13: 动态注册节点的 maxScopes 不超过 config 的 `defaultMaxScopes` 上限。

## 交付物

```
packages/plan-pool/           # pool-node 实现
├── src/
│   ├── pool.ts               # 核心：round-robin + failover + provider 验真
│   ├── ledger.ts             # SQLite 记账（含 token/cost 字段）
│   ├── config.ts             # pool 配置加载
│   └── index.ts              # MeshClient 注册到 Hub
├── package.json
└── tsconfig.json

bridges/plan-bridge-cli/      # CLI 封装型 plan-bridge（Chat only）
├── src/
│   ├── bridge.ts             # A2A 协议层（复用 bridge-tts 模式）
│   ├── session_manager.ts    # 多 session 生命周期管理（含 TTL 回收）
│   ├── cli_wrapper.ts        # Claude CLI pipe 封装 + 输出解析 + usage 提取
│   └── attestation.ts        # ProviderAttestation 子模块
├── package.json
└── tsconfig.json

packages/mesh-hub/            # F013-lite 改动
└── src/hub.ts                # HELLO handler 扩展（~50 行）
```

预计总代码量：~600 行

## MVP 不包含

- Coding 模式（远端 Coding 意义不大，调用方本地有 Coding 能力）
- 贡献/消耗公平算法（Phase 2，场景 A 才需要；但 token/cost 账本已在 MVP 采集）
- 浏览器自动化桥接（不稳定，CLI 优先）
- 跨网络 / 跨 Region 组网
- Hub 原生 pool 支持（留 Phase 2，pool-node 逻辑下沉到 Hub routing）
- 场景 A 的多人信任 / 鉴权体系
- `approval` 注册模式
- Prompt 内容过滤（主防线是能力隔离 `--tools ""`，不是内容审查）

## 演进路线

```
Phase 1.5 (F017 MVP — 当前):
  pool-node + plan-bridge(cli/api) + F013-lite + SQLite ledger
  Chat only + token/cost 采集 + ProviderAttestation
  → 验证场景 B（个人多设备无感合并）

Phase 2 (F017 扩展):
  contribution-based fairness（基于已采集的 token/cost 数据）
  + 场景 A（团队共享）+ F013 full（approval 模式）
  + provider 验真自动化（不一致自动降权/禁用）
  + Probe 指纹（L3 验真）

Phase 3 (远期):
  Hub 原生 pool 支持 + F014 直连优化
  + 跨 Region + 非 CLI Plan 桥接（browser automation）
```

## Dependencies

- F001（Security Foundation）— Ed25519 身份体系、L1/L2 token ✅
- F012（Public Discovery Layer）— CAPS / directory ✅
- F013（Open Node Registration）— 前置 F013-lite
- F011（Local TTS Bridge）— bridge 分层模式可参考（related）

## Risk

| # | 风险 | 严重度 | 缓解 |
|---|------|--------|------|
| R1 | CLI 长时间运行的进程稳定性 | 中 | 按需启动 + configurable TTL 回收 + 健康检查 |
| R2 | 重试导致重复消耗 | 中 | invocationId 幂等扣费 |
| R3 | 模型套壳影响额度计算 | 中 | ProviderAttestation 多层验真（L1 二进制 hash + L2 响应校验） |
| R4 | `enablePublicInvoke` 误开导致无鉴权访问 | 高 | 文档明确禁止 + 配置校验 |
| R5 | 双跳延迟（caller→hub→pool→hub→bridge） | 低 | 相比 LLM 推理可忽略；F014 后可优化 |
| R6 | 非 API Plan 的 TOS 合规性 | 中 | MVP 仅限个人多设备（自用），不跨账号 |
| R7 | Prompt 注入 | 低 | Chat only + `--tools ""` = 即使注入成功也无法执行本地操作 |

## Open Questions

1. ~~CLI session 的 TTL 回收策略~~ → 已定：固定 TTL，bridge 启动时可配置（默认 30 分钟）。
2. provider 验真 probe 的 prompt 设计——是否有通用的模型指纹 prompt？（Phase 2）
3. 场景 A（团队共享）的贡献权重公式设计——留 Phase 2，基于 MVP 采集的 token/cost 数据。
4. 多 provider pool（同一个 pool-node 管理 Claude + GPT bridge）是否有意义？还是每个 provider 独立 pool？
5. ~~Coding 模式~~ → 铲屎官裁定不做。远端 Coding 意义不大。

## 参考

### 圆桌讨论

- 讨论记录：`docs/discussions/F017-plan-pool-roundtable.md`

### 业界号池项目调研（2026-04-06）

| 项目 | Stars | 技术路线 | 关键借鉴 |
|------|-------|---------|---------|
| OCP (daktil888) | - | CLI 子进程 | 8 并发共享 + CLAUDE_ALLOWED_TOOLS 白名单 |
| SenZhangAI/claude-cli-proxy | - | CLI 子进程 | **安全隔离黄金参数**：`--setting-sources "" --strict-mcp-config --tools "" --max-budget-usd` |
| mariklolik/claude-max-proxy | - | Agent SDK | FIFO 队列 + 背压 + 503 Retry-After + 5h 滚动窗口配额追踪 |
| magno73/claude-cli-bridge | - | CLI 子进程 | **`--resume` prompt 缓存**：首次完整上下文，后续增量，session 30min 过期 |
| letta-claude-proxy | - | Agent SDK | **Token 池 + Round-Robin 自动轮换** + 限制窗口重置检测 |
| AIClient-2-API (justlovemaki) | 6.6k | 模拟客户端 | 多账号轮询 + 智能故障转移 + TLS Sidecar |
| claude-code-hub (zsio) | 163 | CLI 子进程 | RPM + 日 USD 限额 + 权重分发 |
| ProxyLLM (zhalice2011) | 395 | Web Session 劫持 | Electron + CDP + 多站点适配（高合规风险，仅参考架构） |

**关键结论**：
1. 所有 CLI 路线项目均使用 `claude -p`（`--print`），这是非交互模式的必要入口
2. 安全隔离最佳实践来自 SenZhangAI：`--setting-sources "" --strict-mcp-config --tools ""`
3. 进程创建必须用 `spawn(shell: false)` 而非 `exec()`，防止命令注入
4. **几乎所有项目都没有 prompt 注入防护**——主防线必须是能力隔离，不能依赖内容过滤
5. 会话复用推荐 magno73 的 `--resume` 方案（30min TTL）

### 消耗采集参考

Cat Cafe 中每次调用 Claude 都记录了上下文数量和消耗金额（收+发 token 统计）。F017 的 ledger 借鉴此模式，在 MVP 阶段就采集 `input_tokens` / `output_tokens` / `cost_usd`，为 Phase 2 贡献额度计算打基础。

### 现有 Mesh 架构

- F011 bridge-tts：协议层 + 本地运行时包装层分层模式
- Hub scope 拒绝链路：`hub.ts:365` / `hub.ts:466` 可承载权限分层
- `RATE_LIMITED=3003`：协议已定义，可标准化超限返回
