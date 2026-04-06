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
> 铲屎官裁定：优先做 non-API Plan

## Why

AI 订阅计划（Claude Max、ChatGPT Plus、Kimi Pro 等）绑定到个人账号和设备，存在两个浪费场景：

1. **团队号池**（场景 A）：多人购买了订阅但使用节奏不同——有人在忙时额度不够用，有人休假时额度白白浪费。
2. **个人多设备**（场景 B）：同一人在办公室和实验室各有订阅，希望无感合并使用，不需要切换账号。

如果只是做 API key 转发，一个 Nginx 反代就够了。**Agent Mesh 的差异化价值在于桥接那些不提供 API 的订阅计划**——通过 CLI 封装将"只能在本地用的订阅"变成可池化的 A2A 节点。

铲屎官原话："把 API 直接给人有信任问题也不好控制量，用 Agent Mesh 的场景通过贡献额度来获取额度就比较公平。"

## What

### 核心概念

| 概念 | 定义 |
|------|------|
| **plan-bridge** | 运行在 Plan 订阅所在机器上的桥接器，将一个订阅封装为标准 A2A 节点。持有本地凭证（CLI 登录态 / API key），不暴露给任何外部调用方。 |
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

#### CLI 封装型（non-API Plan，MVP 优先）

适用于不提供 API 的订阅（如 Claude Max Coding Plan）。

```bash
# 标准编程调用模式
claude -p \
  --session-id "<caller-bound-uuid>" \
  --output-format stream-json \
  --permission-mode auto \
  "your prompt here"

# 恢复上下文
claude --resume <session-id>
```

**Session 管理**：每个调用方映射独立的 named session（session-id = UUID），天然隔离上下文。`--resume` 支持跨请求保持上下文连贯。

**并发控制**：每个 provider 的并发上限由 plan-bridge 安装时配置（`maxConcurrency`，默认 3）。超出并发上限的请求排队或返回 `RATE_LIMITED`。

#### API 代理型

适用于提供 API key 的订阅。HTTP 代理转发，bridge 持有 key 不暴露。

### Provider 验真机制

plan-bridge 注册时声明 provider 类型（如 `claude`、`gpt`、`glm`、`kimi`）。为防止模型套壳（声称是 GPT-4o 实际跑的是更便宜的模型），pool-node 需要验证 provider 真实性：

#### 验真策略

| 策略 | 原理 | 适用场景 |
|------|------|---------|
| **Response Header 检查** | 许多 API 返回 `x-model` / `model` 等 header 标识实际模型 | API 代理型 bridge |
| **CLI 输出解析** | Claude CLI 的 `stream-json` 输出包含 `model` 字段 | CLI 封装型 bridge |
| **Probe 指纹** | 定期发送已知 prompt，通过响应特征判断模型身份 | 通用（尤其无 header 的 provider） |
| **自报 + 抽检** | 注册时自报 provider，pool-node 定期抽检验证 | MVP 推荐 |

#### MVP 实现

```typescript
interface PlanBridgeConfig {
  provider: 'claude' | 'gpt' | 'glm' | 'kimi' | string  // 声明 provider
  providerModel?: string        // 具体模型（如 'claude-opus-4-6'）
  maxConcurrency: number        // 并发上限（默认 3，安装时配置）
  bridgeType: 'cli' | 'api'    // 桥接类型
}

// pool-node 在转发响应时提取并校验 provider 信息
interface ProviderVerification {
  claimed: string               // bridge 声明的 provider
  observed: string | null       // 从响应中观测到的 provider
  verified: boolean             // 是否一致
  lastChecked: number           // 最后校验时间
}
```

**MVP 行为**：
- 注册时自报 provider + model
- 每次响应解析实际 model（从 CLI stream-json 的 `model` 字段或 API response header）
- claimed vs observed 不一致时标记 `verified: false`，记录到 ledger，但 MVP 不自动禁用（告警留给管理者判断）

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

      // 记账
      this.ledger.record({
        sourceNodeId: req.sourceNodeId,
        bridgeId: bridge.id,
        provider: bridge.config.provider,
        observedModel,
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
}
```

### Ledger（记账）

```sql
CREATE TABLE invocations (
  id            INTEGER PRIMARY KEY AUTOINCREMENT,
  source_node   TEXT NOT NULL,       -- 调用方 nodeId
  bridge_id     TEXT NOT NULL,       -- 目标 plan-bridge
  provider      TEXT NOT NULL,       -- 声明的 provider
  observed_model TEXT,               -- 实际观测到的 model
  verified      BOOLEAN DEFAULT 1,   -- provider 验真结果
  status        TEXT NOT NULL,       -- success / error / rate_limited
  created_at    INTEGER NOT NULL     -- Unix timestamp ms
);

CREATE INDEX idx_source ON invocations(source_node, created_at);
CREATE INDEX idx_bridge ON invocations(bridge_id, created_at);
```

MVP 按请求次数记账。后续可扩展 `token_count` 字段（从 API 响应或 CLI 输出解析）。

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

- [ ] AC-1: CLI 封装型 plan-bridge 可通过 `claude -p --session-id --output-format stream-json` 封装 non-API Plan 为 A2A 节点。
- [ ] AC-2: 每个调用方映射独立的 named session（UUID），上下文隔离。
- [ ] AC-3: 并发上限由 `maxConcurrency` 配置（默认 3），超限返回 `RATE_LIMITED`。
- [ ] AC-4: API 代理型 plan-bridge 通过 HTTP 转发实现，key 不暴露给调用方。
- [ ] AC-5: plan-bridge 注册时声明 `provider` 和 `providerModel`。

### pool-node

- [ ] AC-6: pool-node 对调用方暴露统一 A2A card（如 `agentId: "claude-pool"`），调用方不感知底层多个 bridge。
- [ ] AC-7: 请求分发策略：round-robin + 429/SESSION_BUSY 自动 failover 到下一个 bridge。
- [ ] AC-8: 所有 bridge 不可用时返回 `RATE_LIMITED` 给调用方。
- [ ] AC-9: SQLite ledger 记录每次调用（source_node / bridge_id / provider / observed_model / status / timestamp）。
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
│   ├── ledger.ts             # SQLite 记账
│   ├── config.ts             # pool 配置加载
│   └── index.ts              # MeshClient 注册到 Hub
├── package.json
└── tsconfig.json

bridges/plan-bridge-cli/      # CLI 封装型 plan-bridge
├── src/
│   ├── bridge.ts             # A2A 协议层（复用 bridge-tts 模式）
│   ├── session_manager.ts    # 多 session 生命周期管理
│   └── cli_wrapper.ts        # Claude CLI pipe 封装 + 输出解析
├── package.json
└── tsconfig.json

packages/mesh-hub/            # F013-lite 改动
└── src/hub.ts                # HELLO handler 扩展（~50 行）
```

预计总代码量：~500 行

## MVP 不包含

- 贡献/消耗公平算法（Phase 2，场景 A 才需要）
- 浏览器自动化桥接（不稳定，CLI 优先）
- 跨网络 / 跨 Region 组网
- Hub 原生 pool 支持（留 Phase 2，pool-node 逻辑下沉到 Hub routing）
- 场景 A 的多人信任 / 鉴权体系
- `approval` 注册模式
- Token 级精确计量（先用请求次数）

## 演进路线

```
Phase 1.5 (F017 MVP — 当前):
  pool-node + plan-bridge(cli/api) + F013-lite + SQLite ledger
  → 验证场景 B（个人多设备无感合并）

Phase 2 (F017 扩展):
  contribution-based fairness + 场景 A（团队共享）
  + F013 full（approval 模式）+ token 级计量
  + provider 验真自动化（不一致自动降权/禁用）

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
| R1 | CLI 长时间运行的进程稳定性 | 中 | 按需启动 + TTL 回收 + 健康检查 |
| R2 | 重试导致重复消耗 | 中 | invocationId 幂等扣费 |
| R3 | 模型套壳影响额度计算 | 中 | provider 验真机制（MVP 告警，Phase 2 自动降权） |
| R4 | `enablePublicInvoke` 误开导致无鉴权访问 | 高 | 文档明确禁止 + 配置校验 |
| R5 | 双跳延迟（caller→hub→pool→hub→bridge） | 低 | 相比 LLM 推理可忽略；F014 后可优化 |
| R6 | 非 API Plan 的 TOS 合规性 | 中 | MVP 仅限个人多设备（自用），不跨账号 |

## Open Questions

1. CLI session 的 TTL 回收策略——多久没使用的 session 回收释放内存？建议 30 分钟。
2. provider 验真 probe 的 prompt 设计——是否有通用的模型指纹 prompt？
3. 场景 A（团队共享）的贡献权重公式设计——留 Phase 2 讨论。
4. 多 provider pool（同一个 pool-node 管理 Claude + GPT bridge）是否有意义？还是每个 provider 独立 pool？

## 参考

- 圆桌讨论记录：`docs/discussions/F017-plan-pool-roundtable.md`
- 业界参考：one-api（API key 池化）、new-api（加权路由）、oai-reverse-proxy（反代）
- 现有 bridge 模式：F011 bridge-tts（协议层 + 本地运行时包装层分层）
