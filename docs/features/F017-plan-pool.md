---
feature_ids: [F017]
related_features: [F001, F011, F012, F013]
topics: [plan-pool, quota-sharing, subscription, peak-shaving, cli-bridge, provider-verification]
doc_kind: spec
created: 2026-04-06
layer: application
owner_module: plan-pool
status: phase-1.5-complete
phase: 1.5
depends_on:
  - { id: F001, type: blocking }
  - { id: F012, type: blocking }
  - { id: F013, type: related, note: "F013-lite（open 注册模式）建议前置实现" }
  - { id: F011, type: related, note: "bridge 分层模式可复用" }
evidence:
  - { type: pr, ref: "#16", merged: "2026-04-07", title: "feat(F017): Plan Pool Phase 1.5 — team mutual-aid MVP" }
  - { type: pr, ref: "#17", merged: "2026-04-08", title: "feat(F017): grey release prep — bootstrap protection + admin/join pages + source install" }
  - { type: pr, ref: "#18", merged: "2026-04-08", title: "fix(F017): grey release hotfix — CLI verbose, response cleanup, ws:// drift, HTTP join fix" }
  - { type: pr, ref: "#30", merged: "2026-04-11", title: "feat(F017): P1 public API entry normalization + attestation re-approve" }
roundtable: docs/discussions/F017-plan-pool-roundtable.md
---

# F017: Plan Pool — 订阅计划池化与削峰填谷

> Status: **Phase 1.5 complete** (PR #16 merged 2026-04-07, PR #17 merged 2026-04-08) | Owner: TBD
> Roundtable: 2026-04-06（opus / codex / gemini / antig-opus）
> Follow-up: 2026-04-07（opus / codex — 铲屎官 6 点改进 + 竞品分析）
> Roundtable 2: 2026-04-07（opus / gpt52 / gemini / antig-opus — 团队互助额度机制）
> 铲屎官裁定：优先做 non-API Plan；只做 Chat 模式；**MVP 优先场景 A（团队互助）**
> **2026-04-07: Phase 1.5 implementation merged** (PR #16, 19 TDD tasks, 102 tests, 4 QG rounds + 3 review rounds)
> **2026-04-08: Grey release prep merged** (PR #17, bootstrap protection + admin/join pages + source install)
> **2026-04-08: Grey release hotfix merged** (PR #18, CLI verbose flag + response cleanup + ws:// drift + HTTP join fix)

## Why

AI 订阅计划（Claude Max、ChatGPT Plus、Kimi Pro 等）绑定到个人账号和设备，存在两个浪费场景：

1. **团队号池**（场景 A）：多人购买了订阅但使用节奏不同——有人在忙时额度不够用，有人休假时额度白白浪费。
2. **个人多设备**（场景 B）：同一人在办公室和实验室各有订阅，希望无感合并使用，不需要切换账号。

如果只是做 API key 转发，一个 Nginx 反代就够了。**Agent Mesh 的差异化价值在于桥接那些不提供 API 的订阅计划**——通过 CLI 封装将"只能在本地用的订阅"变成标准 API 可调用的资源，同时也注册为 A2A 节点供 Mesh 内部使用。

**为什么只做 Chat 模式**：远端 Coding 意义不大——调用方如果在 Claude 等 IDE 中使用，本地就有 Coding 能力，不需要远端做。Chat 模式（`--tools ""`）同时也是安全隔离最强的模式。

铲屎官原话："把 API 直接给人有信任问题也不好控制量，用 Agent Mesh 的场景通过贡献额度来获取额度就比较公平。"

## What

### 核心概念

| 概念 | 定义 |
|------|------|
| **team** | 一等实体。一个 team 对应一个 provider 类型（`provider_scope`），拥有独立的成员、邀请码、额度账本。同一个 pool-node 可托管多个 team（如 "Lab-Claude" 和 "Lab-GPT"）。team 之间账本隔离，不做跨 provider 额度映射。 |
| **member** | team 的成员。通过一次性邀请码加入，加入后获得 per-member API key。每个 member 有独立的 credit account（透支额度模型）。 |
| **plan-bridge** | 运行在 Plan 订阅所在机器上的桥接器，将一个订阅封装为标准 A2A 节点。持有本地凭证（CLI 登录态 / API key），不暴露给任何外部调用方。**仅支持 Chat 模式**（纯对话，无工具能力）。每个 bridge 归属一个 team + 一个 owner member。 |
| **pool-node** | 路由网关节点，**对外暴露 Provider-native HTTP API（主）和 A2A card（辅）**，托管多个 team，管理 bridge 注册、请求分发、failover、记账。可跑在任意可达的机器上（如 Hub 同机或 NAS）。 |
| **provider** | 实际提供 AI 能力的服务商（Claude / GPT / GLM / Kimi 等）。每个 team 的 `provider_scope` 限定该 team 只接受对应 provider 的 bridge 注册。 |
| **api-gateway** | pool-node 内置的 HTTP API 层。根据 team 的 `provider_scope` 自动暴露对应格式的 API：Claude team → Anthropic `/v1/messages`，GPT team → OpenAI `/v1/chat/completions`。**Caller 不需要是 Mesh 节点**，任何支持标准 API 的工具（Claude Code、Cursor、Continue 等）均可直接调用。 |

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
              │                │
              │  HTTP API:     │  ← 主入口（Provider-native）
              │  https://<pool-public-base>/v1/... │
              │  (标准 443；内部反代到 :3011)      │
              │  A2A card:     │  ← 辅入口（Mesh 内部）
              │  claude-pool   │
              └───────┬────────┘
                      │
              ┌───────┴────────┐
              │  Mesh Hub      │
              └───────┬────────┘
                      │
       ┌──────────────┼──────────────┐
       │              │              │
┌──────┴─────┐ ┌──────┴─────┐ ┌─────┴──────┐
│ Claude Code│ │ Cursor     │ │ Mesh Agent │
│ (HTTP API) │ │ (HTTP API) │ │ (A2A)      │
└────────────┘ └────────────┘ └────────────┘

主数据流（API）：Caller →[HTTP]→ pool-node →[A2A]→ Hub → plan-bridge → Provider
辅数据流（A2A）：Mesh Agent →[A2A]→ Hub → pool-node →[A2A]→ Hub → plan-bridge → Provider
```

**设计原则**：
- **API 为主，A2A 为辅**——Caller 不一定是 Mesh 节点，大多数 AI 工具只支持标准 HTTP API
- **Provider-native 格式**——底层是 Claude bridge 就对外暴露 Anthropic API，底层是 GPT bridge 就暴露 OpenAI API
- **对外统一入口、对内独立进程**——pool-node 内部保持独立监听（当前默认 `3011`），正式对外入口收敛到标准 `443` 反向代理，避免 caller 记特殊端口
- **bridge locality 原则**：plan-bridge 必须跑在 Plan 订阅所在的机器上，凭证不离开本地
- **灰度期连接约束（临时）**：若 Hub 仍只监听 `127.0.0.1`，桥接仍应保持在凭证机，本地 bridge 通过反向隧道/relay 满足 Hub 回调可达；同机部署仅允许紧急止血，不作为规范路径

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
| L1.5 | **环境探针**：校验 `BASE_URL` / auth source 是否指向官方 endpoint | 启动时 + 周期性 | 全部 |
| L2 | **响应校验**：从 stream-json `model` 字段或 API `x-model` header 提取实际模型 | 每次响应 | 全部 |
| L3 | **统计异常检测**：基于 ledger 数据，对延迟/token 消耗/输出风格进行统计分析 | 周期性 | Phase 2 |

#### MVP 实现

```typescript
interface ProviderAttestation {
  // L1: 二进制校验
  binaryPath: string           // /usr/local/bin/claude（固定绝对路径）
  binaryHash: string           // SHA-256 of binary
  version: string              // claude --version 输出
  // L1.5: 环境探针
  envProbe: {
    baseUrl: string | null       // 实际生效的 BASE_URL（null = 官方默认）
    isOfficialEndpoint: boolean  // 是否匹配已知官方域名白名单
    authMethod: 'oauth' | 'api-key' | 'unknown'  // 认证方式
    configSource: string         // 配置来源（env var / config file / default）
  }
  // L2: 响应校验
  claimed: string              // bridge 声明的 provider
  observed: string | null      // 从响应中观测到的 provider
  verified: boolean            // 是否一致
  // 元数据
  fingerprint: string          // 组合指纹（path + hash + version + envProbe）
  lastVerified: number         // 上次校验时间
  verifyInterval: number       // 校验间隔（默认 3600000ms = 1h）
}

// 官方 endpoint 白名单（L1.5 校验用）
const OFFICIAL_ENDPOINTS: Record<string, string[]> = {
  claude: ['api.anthropic.com'],
  gpt:    ['api.openai.com'],
  glm:    ['open.bigmodel.cn'],
  kimi:   ['api.moonshot.cn'],
  gemini: ['generativelanguage.googleapis.com'],
}

interface PlanBridgeConfig {
  provider: 'claude' | 'gpt' | 'glm' | 'kimi' | string
  providerModel?: string
  maxConcurrency: number        // 硬限制，安装时配置，调用方不可覆盖
  bridgeType: 'cli' | 'api'
  sessionTTLMinutes: number     // Session 回收 TTL（默认 30，启动时可配置）
  maxBudgetPerRequest: number   // 每请求预算上限（USD，默认 0.5）
  poolMode: 'self' | 'team'     // self=场景B个人自用, team=场景A团队共享
  endpointAllowlist?: string[]  // team 模式下允许的非官方 endpoint（默认空=仅官方）
}
```

**MVP 行为**：
- 启动时校验二进制 hash + version（L1）
- 启动时校验 BASE_URL 是否指向官方 endpoint（L1.5）——`self` 模式 warn-only，`team` 模式默认拒绝非官方（可配 allowlist 例外）
- 每次响应解析实际 model（L2）
- claimed vs observed 不一致时标记 `verified: false`，记录到 ledger 并告警
- HELLO 握手携带 attestation 字段（bridge version / binary hash / channel），MVP warn-only 写 ledger
- MVP 不自动禁用（留给管理者判断），Phase 2 可配置自动降权

### pool-node 核心逻辑

```typescript
// === Circuit Breaker（三态熔断器）===
type BreakerState = 'closed' | 'open' | 'half-open'
interface CircuitBreaker {
  state: BreakerState
  failureCount: number
  lastFailure: number
  // 不同错误类型走不同策略
  classify(error: MeshError): 'quota' | 'transient' | 'misconfigured'
}
// 错误分类策略：
//   429 → quota cooldown（等窗口重置，通常 5h）
//   5xx/timeout → transient（计数熔断，阈值 4 次，恢复探针 60s）
//   401/403 → misconfigured（长熔断，需人工修复）
// half-open 状态使用 single-flight 探针，避免恢复风暴

// === Session Binding（会话粘性）===
interface SessionBinding {
  bridgeId: string
  lastActive: number
  sessionNamespace: string  // callerId + provider + namespace 的组合键
}

class PoolNode {
  private bridges: Map<string, BridgeEntry>
  private breakers: Map<string, CircuitBreaker>
  private ledger: SQLiteLedger
  private sessionBindings: Map<string, SessionBinding> = new Map()
  private roundRobinIndex: number = 0

  async invoke(req: InvokeRequest): Promise<InvokeResponse> {
    // 生成 idempotency key（防重复扣费）
    const idempotencyKey = req.invocationId ?? generateUUID()
    
    // Session-sticky 路由：优先返回已绑定的 bridge
    const bridge = this.selectBridge(req)
    if (!bridge) {
      throw new MeshError('RATE_LIMITED', 'All bridges busy or unavailable')
    }

    // Lease-based 记账：预扣额度
    const leaseId = await this.ledger.acquireLease({
      idempotencyKey,
      sourceNodeId: req.sourceNodeId,
      bridgeId: bridge.id,
      provider: bridge.config.provider,
      leaseTTL: bridge.config.timeout + 30_000, // bridge timeout + 30s buffer
    })

    try {
      const result = await this.meshClient.invoke(bridge.agentId, req)

      // 验真：从响应中提取 provider 信息
      const observedModel = this.extractModel(result)
      this.verifyProvider(bridge, observedModel)

      // 从 stream-json 输出中提取 token 消耗
      const usage = this.extractUsage(result)

      // 结算 lease（预扣 → 实际）
      await this.ledger.settleLease(leaseId, {
        observedModel,
        inputTokens: usage.inputTokens,
        outputTokens: usage.outputTokens,
        costUsd: usage.costUsd,
        status: 'success',
      })

      // 更新 session binding
      this.updateSessionBinding(req, bridge)
      // 重置 circuit breaker 计数
      this.breakers.get(bridge.id)?.reset()

      return result
    } catch (e) {
      const errorType = this.breakers.get(bridge.id)?.classify(e)
      this.breakers.get(bridge.id)?.recordFailure(errorType)

      // 回滚 lease
      await this.ledger.rollbackLease(leaseId, { status: 'error', errorCode: e.code })

      if (e.code === 'RATE_LIMITED' || e.code === 'SESSION_BUSY') {
        return this.failover(req, bridge)
      }
      throw e
    }
  }

  // Session-sticky + round-robin fallback
  private selectBridge(req: InvokeRequest): BridgeEntry | null {
    const stickyKey = `${req.sourceNodeId}:${req.targetProvider ?? 'any'}`

    // 1. 优先返回已绑定的 bridge（如果健康且未熔断）
    const binding = this.sessionBindings.get(stickyKey)
    if (binding) {
      const bridge = this.bridges.get(binding.bridgeId)
      const breaker = this.breakers.get(binding.bridgeId)
      if (bridge && breaker?.state !== 'open'
          && bridge.activeSessions < bridge.config.maxConcurrency) {
        binding.lastActive = Date.now()
        return bridge
      }
      // 绑定的 bridge 不可用 → 解绑，走 round-robin
      this.sessionBindings.delete(stickyKey)
    }

    // 2. Round-robin fallback
    const available = this.getAvailableBridges()
    if (available.length === 0) return null
    return available[this.roundRobinIndex++ % available.length]
  }

  private getAvailableBridges(): BridgeEntry[] {
    return [...this.bridges.values()]
      .filter(b => b.activeSessions < b.config.maxConcurrency)
      .filter(b => this.breakers.get(b.id)?.state !== 'open')
  }

  private updateSessionBinding(req: InvokeRequest, bridge: BridgeEntry): void {
    const stickyKey = `${req.sourceNodeId}:${req.targetProvider ?? 'any'}`
    this.sessionBindings.set(stickyKey, {
      bridgeId: bridge.id,
      lastActive: Date.now(),
      sessionNamespace: stickyKey,
    })
  }

  // Session binding TTL 回收（默认 30 分钟无活动解绑）
  // 也支持 caller 显式 invalidate（发送 new-session 意图时立即解绑）
  private cleanupStaleBindings(): void {
    const ttl = 30 * 60 * 1000
    for (const [key, binding] of this.sessionBindings) {
      if (Date.now() - binding.lastActive > ttl) {
        this.sessionBindings.delete(key)
      }
    }
  }

  private extractUsage(result: InvokeResponse): UsageInfo {
    return parseUsageFromStreamJson(result.body)
  }
}
```

### API Gateway（Provider-native HTTP 层）

pool-node 内置 HTTP 服务器，根据底层 bridge 的 provider 类型自动暴露对应格式的 API 端点。

#### 端点映射

| bridge provider | 暴露的 HTTP 端点 | 格式标准 | 典型 caller |
|----------------|-----------------|---------|------------|
| `claude` | `POST /v1/messages` | Anthropic Messages API | Claude Code (`ANTHROPIC_BASE_URL`) |
| `gpt` | `POST /v1/chat/completions` | OpenAI Chat Completions API | Cursor, Continue, Cline |
| 混合 pool | 同时暴露两个端点，按端点路由到对应 provider 的 bridge | — | 任意 |

#### Caller 使用方式

```bash
# Claude Code 使用 pool 的 Claude bridge
ANTHROPIC_BASE_URL=https://<pool-public-base> claude code

# Cursor 使用 pool 的 GPT bridge
# 在 Cursor 设置中将 OpenAI endpoint 指向 https://<pool-public-base>
```

#### 认证

member 加入 team 后获得 per-member API key（不暴露底层 bridge 的凭证）：

```typescript
interface PoolApiConfig {
  listenPort: number             // 默认 3011（internal app port）
  publicBaseUrl: string          // 正式公共入口，例如 https://<pool-public-base>
  teams: TeamConfig[]            // pool 托管的 team 列表
}
```

认证流程：API key → 查询 `api_keys` 表 → 确定 `team_id + member_id` → 路由到该 team 的 bridge。

#### 格式转换

```typescript
// api-gateway.ts — 核心逻辑

class ApiGateway {
  constructor(private pool: PoolNode, private config: PoolApiConfig) {}

  // Anthropic 兼容端点
  async handleAnthropicMessages(req: Request, res: Response) {
    const callerId = this.authenticate(req)  // Bearer key → callerId
    const { messages, model, stream, max_tokens } = req.body

    // Anthropic Messages → pool-node InvokeRequest
    const invokeReq = {
      sourceNodeId: callerId,
      targetProvider: 'claude',
      body: { prompt: this.anthropicToPrompt(messages), model, max_tokens },
    }

    const result = await this.pool.invoke(invokeReq)

    // pool-node 响应 → Anthropic Messages 响应格式
    if (stream) {
      return this.streamAnthropicSSE(res, result)  // SSE: event: content_block_delta
    }
    res.json(this.toAnthropicResponse(result))
  }

  // OpenAI 兼容端点
  async handleOpenAIChatCompletions(req: Request, res: Response) {
    const callerId = this.authenticate(req)
    const { messages, model, stream, max_tokens } = req.body

    const invokeReq = {
      sourceNodeId: callerId,
      targetProvider: 'gpt',
      body: { prompt: this.openaiToPrompt(messages), model, max_tokens },
    }

    const result = await this.pool.invoke(invokeReq)

    if (stream) {
      return this.streamOpenAISSE(res, result)  // SSE: data: {"choices":[...]}
    }
    res.json(this.toOpenAIResponse(result))
  }

  // 路由表
  registerRoutes(app: Express) {
    app.post('/v1/messages', (req, res) => this.handleAnthropicMessages(req, res))
    app.post('/v1/chat/completions', (req, res) => this.handleOpenAIChatCompletions(req, res))
    // 模型列表（让 caller 发现可用模型）
    app.get('/v1/models', (req, res) => this.listAvailableModels(res))
  }
}
```

#### 流式输出

| provider | SSE 格式 | 关键字段 |
|----------|---------|---------|
| Anthropic | `event: content_block_delta\ndata: {...}` | `delta.text`, `usage` |
| OpenAI | `data: {"choices":[{"delta":{"content":"..."}}]}` | `choices[0].delta.content` |

bridge 内部输出是 `stream-json`（Claude CLI）或 provider 原生流。api-gateway 负责将其转译为 caller 期望的 SSE 格式。

### 团队管理（Team Management）

#### Team 数据模型

```sql
-- Team：一等实体，每个 team 绑定一个 provider 类型
CREATE TABLE teams (
  id                      TEXT PRIMARY KEY,
  name                    TEXT NOT NULL,           -- 团队名称（如 "Lab-Claude"）
  provider_scope          TEXT NOT NULL,           -- 该 team 只接受的 provider（claude / gpt / glm / kimi）
  owner_member_id         TEXT NOT NULL,           -- 管理员（创建者）
  default_borrow_limit_usd REAL NOT NULL DEFAULT 30.0, -- 新成员默认透支上限（USD）
  created_at              INTEGER NOT NULL
);

-- Member：团队成员
CREATE TABLE members (
  id                      TEXT PRIMARY KEY,        -- member 唯一 ID
  team_id                 TEXT NOT NULL REFERENCES teams(id),
  display_name            TEXT NOT NULL,
  public_key              TEXT NOT NULL,
  role                    TEXT NOT NULL DEFAULT 'member',  -- owner / admin / member
  status                  TEXT NOT NULL DEFAULT 'active',  -- active / suspended
  borrow_limit_override_usd REAL,                  -- 管理员为个人设置的透支上限（覆盖 team 默认值）
  joined_at               INTEGER NOT NULL,
  last_active_at          INTEGER NOT NULL
);

-- API Keys：per-member，用于 HTTP API 认证
CREATE TABLE api_keys (
  key_hash                TEXT PRIMARY KEY,        -- SHA-256(key)，不存明文
  team_id                 TEXT NOT NULL REFERENCES teams(id),
  member_id               TEXT NOT NULL REFERENCES members(id),
  status                  TEXT NOT NULL DEFAULT 'active',  -- active / revoked
  created_at              INTEGER NOT NULL
);

-- Invite Codes：一次性邀请码
CREATE TABLE invites (
  code_hash               TEXT PRIMARY KEY,        -- SHA-256(code)
  team_id                 TEXT NOT NULL REFERENCES teams(id),
  expires_at              INTEGER NOT NULL,         -- TTL
  max_uses                INTEGER NOT NULL DEFAULT 1, -- 最大使用次数
  used_count              INTEGER NOT NULL DEFAULT 0,
  created_by              TEXT NOT NULL REFERENCES members(id),
  created_at              INTEGER NOT NULL
);

-- Bridges：归属 team + owner
-- ADR 2026-04-09: id 改为 server-assigned UUID；client_name 存 client 提供的名称
-- UNIQUE(team_id, owner_member_id, client_name) 保证同 member 下名称唯一
CREATE TABLE bridges (
  id                      TEXT PRIMARY KEY,          -- server-generated UUID
  team_id                 TEXT NOT NULL REFERENCES teams(id),
  owner_member_id         TEXT NOT NULL REFERENCES members(id),
  client_name             TEXT NOT NULL,              -- client 提供的 bridgeId（原 PLAN_BRIDGE_NODE_ID）
  share_mode              TEXT NOT NULL DEFAULT 'idle-only', -- off / idle-only / capped
  external_max_concurrency INTEGER NOT NULL DEFAULT 1,
  last_owner_active_at    INTEGER,
  attestation_hash        TEXT,                    -- bridge 二进制 hash（首次 pin）
  attestation_version     TEXT,
  created_at              INTEGER NOT NULL,
  UNIQUE(team_id, owner_member_id, client_name)
);
```

#### 邀请码加入流程

```
1. Admin 创建邀请码 → POST /v1/admin/invites { teamId, maxUses, ttlHours }
   → 返回一次性 code（如 "lab-abc123"）
   
2. Admin 通过微信/飞书发送 code 给组员

3. 新成员加入 → POST /v1/join { code, displayName }
   → 验证 code 有效（TTL + maxUses + team 匹配）
   → TOFU：绑定 publicKey，注册为 member
   → 签发 per-member API key
   → 返回 { apiKey, teamName, borrowLimit }
   → code.used_count++，用完即失效

4. 成员挂载 bridge → bridge 启动时携带 API key + team_id
   → pool-node 验证 member 身份 + provider_scope 匹配
   → 首次注册：pin attestation（version/hash/channel）
   → 后续变更：暂停共享，admin 确认后恢复
```

#### 额度模型（Mutual Credit）

**核心语义**：不是"送你钱"，是"允许你借钱"。

```
member.balance = contributed_usd - consumed_usd
member.borrow_limit = member.borrow_limit_override_usd ?? team.default_borrow_limit_usd

可用额度 = balance + borrow_limit
当 可用额度 <= 0 → 返回 QUOTA_EXHAUSTED

contributed_usd = Σ cost_usd WHERE bridge.owner_member_id = member AND status = 'success'
consumed_usd    = Σ cost_usd WHERE source_member_id = member AND status = 'success'
```

**贡献/消耗规则**：
- A 用 B 的 bridge → A 消耗 -$X，B 贡献 +$X（零和，不凭空产生额度）
- A 用自己的 bridge → **禁止**（pool-node 硬规则：`bridge.owner_member_id !== caller.member_id`）
- A 本地直接用 CLI → 不经过 pool，不记账（自用无关团队）

**成本计算（normalized_usd）**：
```typescript
// 优先级：provider 上报 > token × 官方定价 > fixed weight
function calculateNormalizedCost(result: InvokeResult): number {
  // 1. provider 直接上报（stream-json 的 cost 字段）
  if (result.costUsd != null) return result.costUsd
  // 2. token × 官方 API 定价
  const pricing = OFFICIAL_PRICING[result.model]
  if (pricing && result.inputTokens != null) {
    return result.inputTokens * pricing.input + result.outputTokens * pricing.output
  }
  // 3. fallback: 固定请求权重（兜底）
  return 0.02  // ~Sonnet 单次 Chat 的名义成本
}

const OFFICIAL_PRICING: Record<string, { input: number, output: number }> = {
  'claude-sonnet-4':   { input: 3.0 / 1e6, output: 15.0 / 1e6 },
  'claude-opus-4':     { input: 15.0 / 1e6, output: 75.0 / 1e6 },
  'gpt-4o':            { input: 2.5 / 1e6, output: 10.0 / 1e6 },
  'gpt-4o-mini':       { input: 0.15 / 1e6, output: 0.6 / 1e6 },
}
```

**冷启动示例**（`default_borrow_limit_usd = 30.0`）：

```
新成员小明加入，balance = $0，borrow_limit = $30
可用额度 = $0 + $30 = $30（可以先用）

小明消耗了 $10 → balance = -$10，可用 = -$10 + $30 = $20（继续用）
小明的 bridge 为别人服务了 $20 → balance = -$10 + $20 = $10，可用 = $10 + $30 = $40
小明又消耗了 $40 → balance = $10 - $40 = -$30，可用 = -$30 + $30 = $0（触顶，需要贡献了）
管理员给小明加授信 → borrow_limit_override = $50 → 可用 = -$30 + $50 = $20（又能用了）
```

#### Owner-first（本机优先）

Bridge owner 对自己的 bridge 有优先权。MVP 做静态版：

```typescript
type ShareMode = 'off' | 'idle-only' | 'capped'

// bridge config
interface BridgeShareConfig {
  shareMode: ShareMode           // 默认 idle-only
  externalMaxConcurrency: number // 外部最大并发（默认 1）
  ownerIdleTimeout: number       // owner 多久不活跃算 idle（默认 30 分钟）
}

// pool-node 路由检查
function canAcceptExternalRequest(bridge: Bridge, callerMemberId: string): boolean {
  if (bridge.shareMode === 'off') return false
  if (bridge.shareMode === 'idle-only') {
    const ownerIdle = Date.now() - bridge.lastOwnerActiveAt > bridge.ownerIdleTimeout
    return ownerIdle && bridge.externalActiveSessions < bridge.externalMaxConcurrency
  }
  if (bridge.shareMode === 'capped') {
    return bridge.externalActiveSessions < bridge.externalMaxConcurrency
  }
  return false
}
```

#### 管理操作（MVP 最小集）

| 操作 | 端点 | 权限 |
|------|------|------|
| 创建 team | `POST /v1/admin/teams` | pool owner |
| 创建邀请码 | `POST /v1/admin/invites` | team owner/admin |
| 暂停成员 | `PUT /v1/admin/members/:id/suspend` | team owner/admin |
| 设置成员透支上限（含 owner 自己） | `PUT /v1/admin/members/:id/borrow-limit` | team owner/admin |
| 轮换成员 API key | `POST /v1/admin/members/:id/rotate-key` | team owner/admin |
| 查看团队用量 | `GET /v1/admin/usage` | team owner/admin |

### Ledger（记账）

```sql
CREATE TABLE invocations (
  id                INTEGER PRIMARY KEY AUTOINCREMENT,
  idempotency_key   TEXT NOT NULL UNIQUE, -- 幂等键（防重复扣费）
  team_id           TEXT NOT NULL,        -- 所属团队
  source_member_id  TEXT NOT NULL,        -- 调用方 member
  bridge_id         TEXT NOT NULL,        -- 目标 plan-bridge
  bridge_owner_id   TEXT NOT NULL,        -- bridge 所有者 member（贡献方）
  provider          TEXT NOT NULL,        -- 声明的 provider
  observed_model    TEXT,                 -- 实际观测到的 model
  verified          BOOLEAN DEFAULT 1,    -- provider 验真结果
  input_tokens      INTEGER DEFAULT 0,    -- 输入 token 数
  output_tokens     INTEGER DEFAULT 0,    -- 输出 token 数
  normalized_cost   REAL DEFAULT 0,       -- 名义成本（normalized_usd）
  cost_source       TEXT DEFAULT 'fallback', -- provider / token_pricing / fallback
  lease_status      TEXT NOT NULL DEFAULT 'pending',  -- pending / settled / rolled_back / expired
  lease_expires     INTEGER,              -- lease 到期时间（Unix ms）
  status            TEXT NOT NULL,        -- success / error / rate_limited
  created_at        INTEGER NOT NULL      -- Unix timestamp ms
);

CREATE INDEX idx_team ON invocations(team_id, created_at);
CREATE INDEX idx_source ON invocations(source_member_id, created_at);
CREATE INDEX idx_bridge_owner ON invocations(bridge_owner_id, created_at);
CREATE UNIQUE INDEX idx_idempotency ON invocations(idempotency_key);
CREATE INDEX idx_lease ON invocations(lease_status) WHERE lease_status = 'pending';

-- 余额视图（实时计算，不缓存）
CREATE VIEW member_balances AS
SELECT
  m.id AS member_id,
  m.team_id,
  m.display_name,
  COALESCE(c.contributed, 0) AS contributed_usd,
  COALESCE(d.consumed, 0) AS consumed_usd,
  COALESCE(c.contributed, 0) - COALESCE(d.consumed, 0) AS balance_usd,
  COALESCE(m.borrow_limit_override_usd, t.default_borrow_limit_usd) AS borrow_limit_usd,
  COALESCE(c.contributed, 0) - COALESCE(d.consumed, 0)
    + COALESCE(m.borrow_limit_override_usd, t.default_borrow_limit_usd) AS available_usd
FROM members m
JOIN teams t ON m.team_id = t.id
LEFT JOIN (
  SELECT bridge_owner_id, SUM(normalized_cost) AS contributed
  FROM invocations WHERE status = 'success' AND lease_status = 'settled'
  GROUP BY bridge_owner_id
) c ON c.bridge_owner_id = m.id
LEFT JOIN (
  SELECT source_member_id, SUM(normalized_cost) AS consumed
  FROM invocations WHERE status = 'success' AND lease_status = 'settled'
  GROUP BY source_member_id
) d ON d.source_member_id = m.id;
```

#### Lease-based 记账流程

```
请求进入 → acquireLease（预扣，status=pending）
  → 调用成功 → settleLease（结算，填入实际 token/cost）
  → 调用失败 → rollbackLease（回滚，记录错误码）
  → 超时未结算 → 定期扫描 lease_expires，按保守策略结算并标记异常
```

**设计要点**（借鉴 claude-code-hub 的 lease 模式）：
- `idempotency_key` 保证同一请求不会重复扣费（重试安全）
- `lease_expires = bridge.timeout + 30s buffer`，防止请求还在执行但 lease 已回收
- 悬挂 lease（超时未结算）按保守策略处理：保留预扣额度 + 标记 `anomaly` + 告警
- 每次调用都记录 token 消耗和金额，为 Phase 2 的贡献额度计算打基础
- CLI 的 `stream-json` 输出包含 `inputTokens` / `outputTokens` 信息
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

- [x] AC-1: CLI 封装型 plan-bridge 可通过 `claude -p --session-id --output-format stream-json --tools ""` 封装 non-API Plan 为纯对话 A2A 节点。
- [x] AC-2: 每个调用方映射独立的 named session（UUID），上下文隔离。
- [x] AC-3: 并发上限由 `maxConcurrency` 配置（默认 3），超限返回 `RATE_LIMITED`。此为硬限制，不可被调用方覆盖。
- [x] AC-4: API 代理型 plan-bridge 通过 HTTP 转发实现，key 不暴露给调用方。
- [x] AC-5: plan-bridge 注册时声明 `provider` 和 `providerModel`。
- [x] AC-19: Session TTL 在 bridge 启动时可配置（`sessionTTLMinutes`，默认 30），过期 session 自动回收。

### 安全

- [x] AC-14: `--tools ""` 生效——模型无法执行任何本地工具操作。
- [x] AC-15: 跨调用方 session 隔离——不同调用方的 session-id 不串线，上下文完全独立。
- [x] AC-16: 超预算（`--max-budget-usd`）或超并发（`maxConcurrency`）时返回 429 + `RATE_LIMITED`。
- [x] AC-17: CLI 进程使用 `spawn(shell: false)` + 最小化环境变量 + 隔离工作目录。
- [x] AC-18: provider 二进制 hash 校验在启动时执行，不一致时拒绝启动并记录告警。
- [x] AC-20: L1.5 环境探针——启动时校验 `BASE_URL` 是否指向官方 endpoint，非官方记录到 ledger；`team` 模式下默认拒绝非官方 endpoint，仅允许显式 `endpointAllowlist` 例外。`self` 模式 warn-only。
- [x] AC-21: HELLO 握手携带 attestation 字段（bridge version / binary hash / channel），MVP 阶段 warn-only 写 ledger，不做强校验（为 Phase 2 预留协议字段，避免破坏性升级）。

### pool-node

- [x] AC-6: pool-node **主入口为 Provider-native HTTP API**（Anthropic `/v1/messages` + OpenAI `/v1/chat/completions`），辅入口为 A2A card（如 `agentId: "claude-pool"`）。Caller 不需要是 Mesh 节点。
- [x] AC-7: 请求分发策略：**session-sticky**（key = callerId + provider + namespace）+ round-robin fallback。同一 caller 的连续请求优先路由到同一 bridge（保持 `--resume` 上下文连贯）。绑定失效条件：bridge 不健康 / TTL 过期（默认 30 分钟无活动） / caller 显式请求 new-session。**禁止 caller 指定目标 bridge**。
- [x] AC-8: 所有 bridge 不可用时返回 `RATE_LIMITED` 给调用方（HTTP 429 / A2A RATE_LIMITED）。
- [x] AC-9: SQLite ledger 采用 **lease-based 记账**——请求前预扣（acquireLease）、成功后结算（settleLease）、失败时回滚（rollbackLease）。每条记录含 idempotency_key / source_node / bridge_id / provider / observed_model / input_tokens / output_tokens / cost_usd / lease_status / status / timestamp。悬挂 lease 超时后按保守策略结算并告警。
- [x] AC-10: provider 验真——从响应中提取实际 model，与声明不一致时标记 `verified: false` 并记录。
- [x] AC-22: Circuit Breaker（三态熔断器）——按错误类型分策略：`429` → quota cooldown（等窗口重置）；`5xx/timeout` → transient 计数熔断（阈值 4 次，探针 60s）；`401/403` → misconfigured 长熔断（需人工恢复）。`half-open` 使用 single-flight 探针，避免恢复风暴。

### API Gateway

- [x] AC-23: 根据 team 的 `provider_scope` 自动暴露对应 HTTP 端点——Claude team → `POST /v1/messages`（Anthropic 兼容），GPT team → `POST /v1/chat/completions`（OpenAI 兼容）。多 team pool 同时暴露两个端点。
- [x] AC-24: per-member API key 认证——加入 team 后签发，key → `team_id + member_id`。不暴露底层 bridge 凭证。
- [x] AC-25: 流式输出——Anthropic 端点返回 Anthropic SSE 格式（`event: content_block_delta`），OpenAI 端点返回 OpenAI SSE 格式（`data: {"choices":[...]}`）。bridge 内部 stream-json 由 api-gateway 转译为 caller 期望的格式。
- [x] AC-26: `GET /v1/models` 返回当前 pool 中可用的 provider + model 列表。

### 团队与额度

- [x] AC-27: Team 为一等实体，每个 team 绑定一个 `provider_scope`，只接受对应 provider 的 bridge 注册。同一 pool-node 可托管多个 team。
- [x] AC-28: 一次性邀请码加入——`code + TTL + maxUses`，用完即失效。加入后 TOFU 绑定 publicKey，签发 per-member API key。
- [x] AC-29: 透支额度模型（Mutual Credit）——`available = contributed - consumed + borrowLimit`。`available <= 0` 时返回 `QUOTA_EXHAUSTED`（HTTP 402）。管理员可为个人设置 `borrow_limit_override`。
- [x] AC-30: 成本计算优先级——provider 上报 `costUsd` > `tokens × 官方定价` > fixed request weight（$0.02 fallback）。记录 `cost_source` 标注来源。
- [x] AC-31: 自路由禁止——pool-node 路由硬规则：`bridge.owner_member_id !== caller.member_id`。禁止将请求路由到 caller 自己的 bridge。
- [x] AC-32: Owner-first（本机优先）——bridge 配置 `shareMode`（`off | idle-only | capped`，默认 `idle-only`）。`idle-only` 模式下 owner 活跃时拒绝外部请求；`externalMaxConcurrency` 默认 1。
- [x] AC-33: 管理员操作——MVP 最小集：创建 team、创建邀请码、暂停成员、设置透支上限、轮换 API key、查看用量（`GET /v1/admin/usage` 返回 JSON）。其中“设置透支上限”适用于任意 member，包含 team owner 自己。

### F013-lite

- [x] AC-11: Hub 配置 `registrationMode: "open"` 后，新节点凭 nodeId + publicKey 可 HELLO 自注册。
- [x] AC-12: `whitelist` 模式行为不变（向后兼容）。
- [x] AC-13: 动态注册节点的 maxScopes 不超过 config 的 `defaultMaxScopes` 上限。

## 交付物

```
packages/plan-pool/           # pool-node 实现
├── src/
│   ├── pool.ts               # 核心：session-sticky + self-route-deny + owner-first + failover
│   ├── api-gateway.ts        # HTTP API 层：Anthropic /v1/messages + OpenAI /v1/chat/completions
│   ├── format-anthropic.ts   # Anthropic 格式转换（Messages ↔ InvokeRequest，SSE 流）
│   ├── format-openai.ts      # OpenAI 格式转换（ChatCompletions ↔ InvokeRequest，SSE 流）
│   ├── team.ts               # Team CRUD + invite 管理 + member 加入
│   ├── credit.ts             # Mutual Credit：余额计算 + 透支检查 + normalized_usd
│   ├── circuit-breaker.ts    # 三态熔断器（429/5xx/401 分类策略）
│   ├── ledger.ts             # SQLite 记账（lease-based + team/member 维度）
│   ├── admin-api.ts          # 管理端点（/v1/admin/*）
│   ├── config.ts             # pool 配置加载
│   └── index.ts              # HTTP 服务器 + MeshClient 注册到 Hub
├── package.json
└── tsconfig.json

bridges/plan-bridge-cli/      # CLI 封装型 plan-bridge（Chat only）
├── src/
│   ├── bridge.ts             # A2A 协议层（复用 bridge-tts 模式）
│   ├── session_manager.ts    # 多 session 生命周期管理（含 TTL 回收）
│   ├── cli_wrapper.ts        # Claude CLI pipe 封装 + 输出解析 + usage 提取
│   └── attestation.ts        # ProviderAttestation（L1 二进制 + L1.5 环境探针 + L2 响应）
├── package.json
└── tsconfig.json

packages/mesh-hub/            # F013-lite 改动
└── src/hub.ts                # HELLO handler 扩展（含 attestation 字段预留，~60 行）
```

预计总代码量：~1400 行（团队管理 + 额度 ~400 行）

## MVP 不包含

- Coding 模式（远端 Coding 意义不大，调用方本地有 Coding 能力）
- 浏览器自动化桥接（不稳定，CLI 优先）
- 跨网络 / 跨 Region 组网
- Hub 原生 pool 支持（留 Phase 2，pool-node 逻辑下沉到 Hub routing）
- `approval` 注册模式（MVP 用邀请码）
- Prompt 内容过滤（主防线是能力隔离 `--tools ""`，不是内容审查）
- 模型指纹 / Probe 验真（不稳定，改用 Phase 2 的 ledger 统计异常检测）
- 跨 provider 额度映射 / 倍率系统（MVP 每个 team 单 provider，不需要映射）
- 动态 Quota Overflow（基于 5h 窗口预测的智能释放，MVP 只做静态 owner-first）
- Bridge 防篡改强校验（MVP 仅 TOFU pin attestation，Phase 2 做签名/challenge）
- 管理 Dashboard UI（MVP 通过 `GET /v1/admin/usage` JSON 查看，Phase 2 做 UI）
- 副管理员 / 权限分级（MVP 只有 owner）

## 演进路线

```
Phase 1.5 (F017 MVP — 当前):
  **MVP 优先场景 A（团队互助）**
  Team 一等实体 + 邀请码加入 + per-member API key
  Mutual Credit 透支额度模型 + normalized_usd 记账
  Self-route deny + Owner-first（静态版：off/idle-only/capped）
  pool-node + plan-bridge(cli/api) + F013-lite + SQLite ledger
  Provider-native HTTP API（主）+ A2A card（辅）
  Chat only + ProviderAttestation（L1 + L1.5 + L2）
  Session-sticky 路由 + Circuit Breaker + Lease-based 记账
  管理端点（/v1/admin/*）JSON 查看
  → Caller 可以是 Claude Code / Cursor / 任意 HTTP 工具
  → 场景 B（个人多设备）作为场景 A 的子集自然支持

Phase 1.5 补丁 (grey release 暴露的运维缺口):
  P1: pool-node 启动从 DB 加载 bridge 到 router（重启后 bridge 不丢失）
  P1: bridge→pool 周期性重注册（pool 重启/网络闪断后自动恢复）
  P2: bridge child env 白名单/隔离（隔离 ANTHROPIC_BASE_URL 等 provider 变量，防污染）
  P2: attested binary = executed binary（resolvedBinary 传入 handler，消除 hash/spawn 不一致）
  ~~P2: admin attestation 审批 API（PUT /v1/admin/bridges/:id/approve-attestation）~~ → **Done** PR #30
  ~~P2: bridge ↔ pool-node 长连接/relay~~ → **Done**: Hub dual-transport (F017-WS) 已实现 outbound WS，SSH 隧道已消除

Phase 2 (F017 扩展):
  跨 provider 额度映射（倍率系统）
  + 动态 Quota Overflow（基于 5h 窗口 + 历史利用率的智能释放）
  + F013 full（approval 模式）
  + provider 验真自动化（不一致自动降权/禁用）
  + L3 统计异常检测（基于 ledger 数据）
  + Bridge 防篡改强校验（签名验证 / challenge-response）
  + 管理 Dashboard UI
  + 副管理员 / 权限分级

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
| R6 | 非 API Plan 的 TOS 合规性 | 中 | 团队成员各自运行自己的 bridge（凭证不离开本地），pool 只做路由 |
| R7 | Prompt 注入 | 低 | Chat only + `--tools ""` = 即使注入成功也无法执行本地操作 |
| R8 | BASE_URL 劫持（bridge 指向非官方 endpoint） | 中 | L1.5 环境探针 + team 模式默认拒绝 + allowlist 例外 |
| R9 | Lease 悬挂（请求超时但 lease 未结算） | 中 | lease_expires TTL + 定期扫描 + 保守结算 + 告警 |
| R10 | HTTP API 暴露扩大攻击面 | 中 | per-member API key 认证 + 透支额度限制 + 不暴露 bridge 凭证 |
| R11 | 白嫖（消耗不贡献） | 中 | 透支额度触顶 → QUOTA_EXHAUSTED；管理员可调整或暂停成员 |
| R12 | 邀请码泄露 | 低 | 一次性 + TTL + maxUses=1；泄露后只影响未使用的码，已加入成员不受影响 |
| R13 | 自路由假贡献 | 中 | pool-node 硬规则禁止 self-route，不依赖 availability 准确性 |

## Open Questions

1. ~~CLI session 的 TTL 回收策略~~ → 已定：固定 TTL，bridge 启动时可配置（默认 30 分钟）。
2. ~~provider 验真 probe 的 prompt 设计~~ → 已定：L3 改为基于 ledger 的统计异常检测，不做模型指纹（Phase 2）。
3. ~~场景 A 的贡献权重公式~~ → 已定：MVP 用 mutual credit（透支额度 + normalized_usd），跨 provider 倍率留 Phase 2。
4. ~~多 provider pool~~ → 已定：同一 pool-node 托管多个 team，每个 team 绑定单一 `provider_scope`。跨 provider 映射留 Phase 2。
5. ~~Coding 模式~~ → 铲屎官裁定不做。远端 Coding 意义不大。
6. **Bridge 防篡改的信任模型**（Phase 2）：MVP 做 TOFU pin（首次 attestation hash 钉住），后续变更暂停共享待 admin 确认。强校验（签名/challenge-response）留 Phase 2。
7. **动态 Quota Overflow**（Phase 2）：基于 5h 滚动窗口 + 历史利用率的智能释放。MVP 用静态 `shareMode`。
8. **Claude CLI 的 `costUsd` 稳定性**：CLI `stream-json` 是否稳定输出可用于结算的 cost 字段？如果不稳定，`tokens × 官方定价` 需要作为 MVP 的正式路径而非 fallback。
9. **跨 provider 倍率系统**（Phase 2）：不同 provider / model 如何换算为可比的 normalized_usd。候选：官方 API 定价倍率 / 社区定价共识 / admin 自定义。

## Grey Release Follow-ups (2026-04-08)

> 这批事项来自 2026-04-08 灰度 thread 的真实踩坑，已和铲屎官确认按 `5+1` 主线 + `Legacy` 追踪进入后续清单。

### P1 主线

1. **Public API entry normalization**  
   保留 pool-node 独立进程与内部 `3011` 监听；对外统一收敛到标准 `443` 反向代理入口。优先独立 API 域名/子域名，不与 mesh-hub control plane 同 origin 混布。

2. **Router durability**  
   pool 启动时从 DB 恢复 bridge 到 in-memory router；bridge 增加周期性重注册/心跳，避免 pool 重启后路由空窗。  
   ⚡ 部分进展：`diagnoseNoRoute()` + `NoRouteReason` 类型枚举已实现（路由失败可诊断具体原因：NO_BRIDGE_IN_TEAM / PROVIDER_MISMATCH / SELF_ROUTE_DENIED / CAPACITY_FULL 等）。DB 恢复 + bridge 心跳重注册仍待做。

3. **Bridge 运维完整性**  
   作为同一组 follow-up 处理，但实现时拆成两个子方向：
   - ~~**Durable bridge identity**~~  ✅ 已实现：Hub signing key 持久化（`signingKeyPath` config + file-based JWK 存储）；MeshClient cert auto-heal（heartbeat/caps/requestToken 遇 401 自动 re-HELLO）。
   - **Admin / runbook 闭环**：补 attestation re-approve 路径与 deploy/service runbook，把”suspend 之后如何恢复共享”收口成明确流程。

### P2 主线

4. **Pure-LLM runtime profile**  
   `--bare` 已从默认开启改为 `PLAN_BRIDGE_CLI_BARE=true` 可选（允许 bridge 在有 OAuth 登录的环境下免设 `ANTHROPIC_API_KEY`）。~~完整运行契约（child env allowlist、provider/base-url 隔离、chat-only profile）仍待定义。~~ ✅ env allowlist + cwd 隔离已实现（`buildCleanEnv` allowlist + `cwd: tmpdir()`）。残留：`--bare` 模式下可替换默认 system prompt（当前 `--system-prompt` 为追加）。

5. **Shared usage contract + provider-neutral consumption**  
   backlog 先作为单个 item 管理；实现阶段再拆成：
   - usage schema 契约（与 Cat Cafe/Clowder 对齐 model/tokens/cache/cost/contextWindow 语义）
   - admin/UI 语义（无 provider cost 时如何展示 observed cost / normalized cost / unavailable）

### ~~Legacy~~ ✅ 已完成

6. ~~**Response sanitization research**~~ ✅ **已实现**（2026-04-10）  
   remote-bridge system prompt 注入 + `<tool_call>/<tool_result>` regex 净化（含未闭合标签）+ bridge 只返回净化后 `text`（pool 侧无 raw fallback）。  
   commit: `5369c50 fix(F017): bridge output sanitization + env allowlist + cwd isolation`

### P3 记账

7. **`attested binary = executed binary`**  
   当前记为安全纵深增强，不进入这批灰度 follow-up 主线；等 P1/P2 收口后再评估是否前移。

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

### 竞品深度分析（2026-04-07 补充）

| 项目 | 定位 | 核心借鉴 | 不适用之处 |
|------|------|---------|-----------|
| **cc-switch** (farion1231) | 单用户多 provider 桌面切换器（Tauri/Rust+React） | **Circuit Breaker 三态模型**（Closed/Open/HalfOpen，per-provider）；FailoverSwitchManager 去重（HashSet 防并发切换风暴）；本地 HTTP proxy 拦截 | 单用户场景，不是多用户共享池 |
| **Cubence** | 商业 API 网关（BASE_URL 重定向 + 自发 API key） | **per-key 多维限额**（RPM/日/月）；proxy-gateway 模式验证了 BASE_URL 劫持风险（反面教材） | 可能违反 TOS（共享订阅凭证）；JSON 文件存储不可扩展 |
| **claude-code-hub** (ding113) | 多租户 API 反代（Next.js+Hono+PostgreSQL+Redis） | **Guard Pipeline**（auth→version→model→session→rate→provider→filter→forward）；**Lease-based quota**（预扣-结算-回滚，Redis Lua 原子操作）；**Session stickiness**（Redis 5min TTL）；Provider selector + weighted dispatch + circuit breaker | 中心化架构（key 集中），与我们的分布式 bridge locality 相反 |

**综合借鉴落地**：

| 借鉴项 | 来源 | 落地到 F017 | 阶段 |
|--------|------|------------|------|
| Circuit Breaker 三态 + 错误分类 | cc-switch + claude-code-hub | pool-node bridge 健康管理（AC-22） | MVP |
| Session stickiness | claude-code-hub | pool-node 路由策略（AC-7 改） | MVP |
| Lease-based quota（预扣-结算-回滚） | claude-code-hub | ledger 记账（AC-9 改） | MVP |
| Failover 去重 | cc-switch | pool-node failover 防风暴 | MVP |
| BASE_URL 校验 | Cubence 反面教材 | L1.5 环境探针（AC-20） | MVP |
| Guard Pipeline middleware chain | claude-code-hub | pool-node 请求处理链 | MVP |
| 多维限额（RPM/日/月） | Cubence + claude-code-hub | Phase 2 场景 A | Phase 2 |

### 消耗采集参考

Cat Cafe 中每次调用 Claude 都记录了上下文数量和消耗金额（收+发 token 统计）。F017 的 ledger 借鉴此模式，在 MVP 阶段就采集 `input_tokens` / `output_tokens` / `cost_usd`，为 Phase 2 贡献额度计算打基础。

### 现有 Mesh 架构

- F011 bridge-tts：协议层 + 本地运行时包装层分层模式
- Hub scope 拒绝链路：`hub.ts:365` / `hub.ts:466` 可承载权限分层
- `RATE_LIMITED=3003`：协议已定义，可标准化超限返回
