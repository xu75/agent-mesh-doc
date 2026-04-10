---
feature_ids: [F017]
topics: [plan-pool, oauth-proxy, claude-code-compatibility, subscription-to-api]
doc_kind: discussion
created: 2026-04-10
participants: [opus, gpt52, CVO]
status: draft
---

# F017 重大变更提案：Bridge 从 CLI 文本管道切换到 OAuth Proxy

> **触发**：CVO 提出 plan-pool 必须兼容 Claude Code CLI 的完整功能（包括 tool use），否则"多人组队削峰填谷连 Claude Code 都不能用就没意义了"。
> **结论**：CVO 批准走 OAuth proxy 路线（2026-04-10），要求参考成熟项目实现，先出规格。

## 1. 问题陈述

### 1.1 当前实现

plan-bridge-cli 使用 `claude -p --tools "" --output-format stream-json` 将 Claude CLI 订阅包装成 API。这条链路：

- 输入：结构化 Anthropic Messages API 请求 → **降维为纯文本 prompt**
- 输出：CLI stream-json 文本 → **重新包装为 Anthropic 响应**
- `--tools ""`：模型看不到任何工具定义，不会返回 `tool_use` blocks

### 1.2 核心矛盾

Claude Code 的工作流重度依赖 tool use（Read, Write, Bash, Grep 等几十个工具全靠 `tool_use` blocks 触发）。CLI 文本管道天生不支持 tool use → Claude Code 无法正常工作。

### 1.3 竞品验证

行业内 Claude Max subscription-to-API 的成熟实现（sub2api 11.5k stars, CLIProxyAPI 24.5k stars, ccflare, cc-gateway 等）**全部走 OAuth token 提取 + 直接代理 api.anthropic.com**，而非 CLI 文本包装。CLI 文本包装只在最早期/最简陋的实现中出现，成熟项目已全部弃用。

## 2. 提案：新增 plan-bridge-oauth 模式

### 2.1 新增 Bridge 类型

在现有两种 bridge 类型基础上新增第三种：

| bridge 类型 | 场景 | 机制 | Claude Code 兼容 |
|---|---|---|---|
| `cli-chat-bridge` | non-API 订阅，chat 场景 | `claude -p --tools ""` → 文本 | chat-only |
| `api-proxy-bridge` | API key 订阅 | HTTP 透传到 provider API | 完全兼容（需实现真透传） |
| **`oauth-proxy-bridge`（新）** | **non-API 订阅，全功能** | **OAuth token 代理到 api.anthropic.com** | **完全兼容** |

### 2.2 架构

```
Claude Code / Cursor / Cline
  ↓ ANTHROPIC_BASE_URL=https://pool.example.com
plan-pool api-gateway
  ├─ 认证（team API key → member 身份）
  ├─ 路由（session-sticky → 选择 bridge）
  ├─ 额度检查（mutual credit）
  ↓
plan-bridge-oauth（运行在订阅所有者的本地机器上）
  ├─ 从本地 Claude CLI session 获取 OAuth token
  ├─ 自动刷新（token 过期前 ~30min）
  ├─ 注入 token 到请求 Authorization header
  ├─ 原样转发完整请求 body 到 api.anthropic.com
  ├─ 原样返回响应（包括 SSE streaming）
  └─ 提取 usage 信息用于计费
```

### 2.3 关键设计决策

**D0: Streaming Transport 前置（架构前提）** _(gpt52 review 补充)_

当前 pool-node 到 bridge 的调用链是一次性 `client.invoke()` → 等完整 `payload.text` 返回（`index.ts:74`）。这条路径不支持流式。

OAuth proxy / API 透传模式需要 **端到端 streaming**：client ↔ gateway ↔ bridge ↔ api.anthropic.com 全程 SSE pipe。这是架构前提，不是实现细节。

**方案选项**（需在实施前确定）：

| 方案 | 说明 | 改动量 |
|---|---|---|
| A. HTTP direct backchannel | 透传 bridge 注册时上报 HTTP callback URL，gateway 直接 HTTP 请求 bridge（绕过 A2A invoke） | 中 — 需要 bridge 开 HTTP server |
| B. A2A streaming 扩展 | 在 Hub WS channel 上增加 streaming frame 类型，支持分块返回 | 大 — 需要改 Hub + A2A 协议 |
| C. Bridge 直连 Anthropic + 旁路上报 | Bridge 自己 reverse proxy 到 Anthropic，用量通过旁路上报 pool-node | 小 — bridge 独立工作，pool 只做管理/计费 |

**倾向方案 C'（C 的 NAT 修正版）**：Bridge 直连 Anthropic（保持 token locality），但通过现有 WS 通道做请求/响应隧道（解决 bridge 无公网 IP 的 NAT 穿透问题）。

> **为什么不能用 C 原版**：Bridge 运行在用户本地机器上（NAT 后），没有公网 IP。Gateway 无法发起 inbound HTTP 到 bridge。
>
> **C' 方案流程**：
> ```
> Claude Code → POST /v1/messages (SSE)
>   → api-gateway (公网)
>     → WS frame {type: "proxy-request", body, headers}
>       → Hub WS relay → bridge（现有 outbound WS，NAT 友好）
>         → outbound HTTP POST → api.anthropic.com
>         ← SSE stream
>       ← WS frames {type: "proxy-sse-chunk", data}（逐 chunk 回传）
>     ← Hub WS relay → gateway
>   ← SSE events
> Claude Code
> ```
>
> 本质：保留 C 的核心（bridge 直接持有 token、直接与 Anthropic 通信），但用已有的 WS 通道替代不可达的 HTTP 直连。WS 中转延迟相比 Anthropic API 本身的延迟（数百ms~数十s）可忽略。
>
> **为什么不选 A（Gateway 代请求）**：A 要求 OAuth token 从 bridge 传给 gateway，违反 locality principle。

**方案 A 仍可作为后备**：如果 WS 帧协议改造过于复杂，A 是最快落地的选择（代价是 token 需传到 gateway）。

**D1: 请求转发策略（两层模型）** _(gpt52 R3 review #1#2: 术语拆分 + header 规则统一)_

请求转发分两层，术语不混用：

| 层 | 策略 | 说明 |
|---|---|---|
| **Gateway** | **Strict passthrough** | 完全不解析、不修改请求 body 和 response body，仅做认证+路由 |
| **OAuth Bridge** | **Passthrough-with-normalization** | Body 中仅修改 `<env>` block（指纹归一化），其余内容不碰；Header 全新构建 |

**Gateway 层**：认证 caller 身份 → 选择 bridge → 原样转发请求 body，不解析任何内容。

**Bridge 出站请求**：全新构建（参考 sub2api），不是入站请求的复制。

Header 分为两类：

| 类别 | Header | 来源 | 说明 |
|---|---|---|---|
| **Caller 透传** | `anthropic-version` | caller 原值 | 协议版本 |
| | `anthropic-beta` | caller 原值 | Beta 功能标记 |
| | `content-type` | caller 原值 | 固定 `application/json` |
| | `X-Claude-Code-Session-Id` | caller 原值 | Session 追踪（也用于内部路由） |
| **Bridge 自生成** | `Authorization` | bridge 注入 | `Bearer <oauth-token>` |
| | `User-Agent` | bridge 生成 | 与 owner 环境一致（参考 cc-gateway） |
| | `x-stainless-*` | bridge 生成 | 与 owner 环境一致（参考 sub2api `applyClaudeOAuthHeaderDefaults()`） |
| **丢弃** | `x-api-key` / `authorization` | caller 原值 | gateway 消费后丢弃，不转发 |

> `User-Agent` 和 `x-stainless-*` **不从 caller 透传**——它们包含 caller 的平台信息，会导致指纹不一致。由 bridge 自生成与 owner 环境匹配的值。

**Body 层（指纹归一化）** _(CVO R3: 学习成熟项目，不自己造轮子)_

远端 caller 通过 bridge 访问时，请求 body 中系统 prompt 的 `<env>` block 描述的是 **caller 的环境**，与 OAuth token 所属 owner 不一致：

```
Token: owner@mac 的 OAuth
<env> block: Platform: linux, /home/caller_name  ← 不匹配
```

**处理方案**：bridge 在转发前替换 `<env>` block 为 owner 的环境信息。参考 cc-gateway 已验证的实现。

**`<env>` block 定位策略** _(gpt52 R3 review #3)_：
- Claude Code 将 `<env>` block 注入在请求 body 的 `system` 字段中（字符串或 `system[].text`）
- 匹配规则：正则 `<env>\s*\n[\s\S]*?</env>` 定位 `<env>...</env>` 标签对
- 仅在 `system` 字段中搜索替换，**不搜索 `messages` 数组**（避免误改用户/工具内容）
- 如果未找到 `<env>` block，不修改 body（caller 可能不是 Claude Code）
- 具体实现参考 cc-gateway 的 env 维度替换逻辑，不自造

**注意**：除 `<env>` block 外，body 中所有内容（messages、tools、model、max_tokens 等）一律不碰。tool_use、thinking、image content 等功能自动支持。

**D2: OAuth Token 抽象接口** _(gpt52 review: 降级为抽象接口)_

Token 管理定义为接口，平台具体实现为 research task，不写死到架构决策。

```typescript
interface TokenProvider {
  /** 获取当前有效的 access token */
  getAccessToken(): Promise<string>;
  /** token 是否即将过期或已失效 */
  isHealthy(): boolean;
  /** 主动刷新 token（由调度层触发） */
  refresh(): Promise<void>;
  /** 监听 token 状态变化 */
  onStatusChange(cb: (healthy: boolean) => void): void;
}

interface RefreshStrategy {
  /** 过期前多久触发刷新（实现决定） */
  refreshBuffer: number;
  /** 刷新端点（平台相关，research task 确定） */
  refreshEndpoint: string;
}
```

**Research tasks**（实施前需完成）：
- RT1: 各平台 Claude CLI OAuth token 存储位置（macOS Keychain / Linux keyring / config dir）
- RT2: platform.claude.com token refresh 端点格式（参考 Meridian `tokenRefresh.ts`、better-ccflare 实现）
- RT3: token 过期周期（社区报告 ~8h，需验证）

**D3: 计费信息提取**

透传模式下，bridge 需要从响应中提取 usage 数据用于 mutual credit 计费：

- **非流式**：从响应 body 的 `usage` 字段提取 `input_tokens`, `output_tokens`
- **流式**：从 SSE 的 `message_delta` 事件中提取 `usage` 字段（Anthropic 在流式结束前会发送 usage 信息）
- 提取使用 `TransformStream` tee 方式——读取 usage 但不消费/修改事件流
- 提取后上报给 pool-node 的 ledger

### 2.4 对 gateway 的改造

当前 gateway 的 `/v1/messages` 端点会调用 `anthropicToPrompt()` 降维请求。需要改造为：

**按 bridge 类型分发**：

```typescript
app.post("/v1/messages", async (req, reply) => {
  const auth = authenticate(req);
  const bridge = router.selectBridge({ ... });

  if (bridge.type === "oauth-proxy" || bridge.type === "api-proxy") {
    // 透传模式：body 不动，直接转发
    return proxyPassthrough(req, reply, bridge);
  } else {
    // CLI 适配模式：降维为文本（保留现有逻辑）
    return handleCLIChatRequest(req, reply, bridge, auth);
  }
});
```

**新增端点**（Claude Code gateway spec 要求）：
- `POST /v1/messages/count_tokens` — 透传到 api.anthropic.com
- 支持 `x-api-key` 认证头（Claude CLI 使用）

**Header 透传**：
- `anthropic-beta` → 透传到 api.anthropic.com
- `anthropic-version` → 透传（不再硬编码）

**Session affinity key（优先级递降）** _(gpt52 review: 不依赖单一 header)_：
1. `X-Claude-Code-Session-Id`（Claude Code 自带）
2. `x-namespace`（显式客户端 session header，兼容现有实现）
3. gateway 自动生成的 sticky token（Set-Cookie 或 response header 回传，对 Cursor/Cline 等不发 session header 的客户端兜底）

**D4: Session-Bridge 硬绑定** _(CVO review 2026-04-11)_

> 背景：CVO 实测，Cat Cafe 中切换 auth 后会出现调用失败。原因是 Anthropic 服务端存在与 account 绑定的状态（prompt cache / session 关联 / beta 权限等）。**切换 bridge = 切换 account = session 可能中断。这是正确性问题，不是性能问题。**

**Session-sticky 从"路由优化"升级为"硬约束"**：

```
session 首次请求 → 绑定到 bridge-x
后续请求 → 强制走 bridge-x（不做负载均衡重分配）
解绑条件（仅以下四种）：
  1. bridge-x 心跳超时（离线）
  2. bridge-x 被 circuit breaker 熔断
  3. 绑定 TTL 过期（如 2h 无活动）
  4. bridge-x auth unhealthy（OAuth token 刷新失败或 Anthropic 持续返回 401/403）
```

**Bridge 离线时的处理**：
- Gateway 返回 **HTTP 503** + 明确告知 caller 必须开新 session：
  ```json
  {
    "type": "error",
    "error": {
      "type": "session_expired",
      "message": "Service temporarily unavailable. Please retry with a new session."
    }
  }
  ```
- **不偷偷切到其他 bridge**——切换 account 会导致 prompt cache 失效和潜在的调用失败
- Response header 附带 `X-Session-Expired: true`，方便 caller 程序化判断
- Caller 收到此错误后应丢弃当前 session，用新 session ID 重新开始
- **协议说明** _(gpt52 R2 review #4)_：`error.type=session_expired` 是 gateway 扩展，非 Anthropic 标准错误类型。不保证所有 Anthropic-compatible clients 能自动识别。Gateway 承诺此字段稳定；不识别的 client 会按标准 retryable 503 处理（重试同一 session），这种情况下 caller 需自行处理 session 过期逻辑

**自路由禁止（AC-31）不受影响**：
- `pool.ts:215-216`: `bridge.ownerMemberId === callerId → SELF_ROUTE_DENIED`
- Owner 的请求永远不路由到自己的 bridge，Owner 用自己的订阅时直接本地使用（不经过 pool）
- 因此**不存在"Owner 要回自己 bridge"的场景**——bridge 只给其他团队成员用

**Owner 在用 → Bridge 自动挂 busy（idle-only 策略，保留并调参）**：
- 现有机制：`shareMode: "idle-only"` + `ownerIdleTimeout` + `externalMaxConcurrency`（`pool.ts:226-233`）
- Owner 最后一次活跃后，需等待 idle timeout 到期，bridge 才对外开放
- OAuth proxy 下此机制更重要：owner 直接使用 + bridge 代理走同一 Anthropic account → rate limit 竞争
- **CVO 决定调参**（2026-04-11）：
  - `ownerIdleTimeout`: 当前 5min（运行时）/ 30min（测试）→ **统一为 10min**
  - `externalMaxConcurrency`: 当前 1 → **调为 2**（允许同时代理 2 个外部请求）
- 额外保护：bridge 检测到 429 rate limit → 心跳上报 capacity 降低 → pool 暂停路由新请求

**竞品参考（sub2api session-sticky 实现）**：
- 三级 session key：`X-Claude-Code-Session-Id`（最高优先）→ prompt cache content hash → 完整请求摘要
- Redis 存储 `sessionHash → accountID` 绑定（TTL-based）
- 切换 account 时设 `ForceCacheBilling = true`（将 input_tokens 归类为 cache_read，反映 cache 冷启动成本）

**Session ID 透传说明**：
- `X-Claude-Code-Session-Id` 在 header 白名单中，**原样透传**到 Anthropic
- 由于 session 与 bridge 硬绑定，同一 session ID 始终对应同一 account，不存在跨 account session ID 冲突
- **多人不同 Session ID 无冲突**：不同用户的请求独立处理，可以同时通过同一 bridge
- Session-sticky 的额外收益：Anthropic 端 **prompt cache** 命中率更高

### 2.5 真 Streaming 实现

**非流式**（`stream: false`）：
```
gateway → bridge → fetch(api.anthropic.com, {body}) → await resp.json() → 返回
```

**流式**（`stream: true`）：
```
gateway → bridge → fetch(api.anthropic.com, {body, stream: true})
                   → resp.body (ReadableStream)
                   → pipe 回 gateway → pipe 回 client
```

流式响应中，bridge 需要：
1. 读取 SSE 事件流中的 `message_delta` 事件提取 usage（用于计费）
2. 同时将所有事件原样转发给调用方
3. 使用 `TransformStream` 或类似机制实现"读取但不消费"的 tee

## 3. 安全模型变更

### 3.1 与 F017 原安全模型的差异

| 安全层 | 原 CLI bridge | 新 OAuth bridge | 变化 |
|---|---|---|---|
| L0 能力隔离 | `--tools ""` 禁用所有工具 | **无（透传模式不限制 tools）** | ⚠️ Bridge 本身不执行工具，但不再阻止模型返回 tool_use |
| L1 进程隔离 | spawn + 最小 env + 隔离 cwd | **HTTP proxy（无子进程）** | 简化——没有子进程就没有进程隔离问题 |
| L2 插件隔离 | 禁用 hooks/MCP | **不适用** | 简化——不运行 Claude CLI |
| L3 额度熔断 | --max-budget-usd | **通过 rate limit / token 计费实现** | 改为 pool-node 层面控制 |
| L4 超时强杀 | 配置超时 + SIGTERM | **HTTP 请求超时 + AbortController** | 等效 |

### 3.2 风险评估

**降低的风险**：
- 不再有子进程注入风险（没有 spawn）
- 不再有 CLI 插件/hook 攻击面

**新增的风险**：
- OAuth token 存储在 bridge 进程内存中（需确保进程安全）
- Token refresh 需要访问 platform.claude.com（需出站网络）

**不变的约束**：
- Bridge 运行在订阅所有者的本地机器上（locality principle 不变）
- 凭证不离开 bridge 机器（OAuth token 不暴露给 gateway 或调用方）
- 调用方只有 team API key，无法直接访问 Anthropic
- Pool-node 的 mutual credit / quota / routing 全部保留

### 3.3 关于 tool use 的安全说明

在 OAuth proxy 模式下，模型可能返回 `tool_use` blocks。但这些 tool **在调用方本地执行，不在 bridge 上执行**。Bridge 只是一个 HTTP 代理——它不理解也不执行 tool calls。

这意味着安全边界从"bridge 不允许任何工具"变成"bridge 不执行任何工具"——本质上更安全，因为 bridge 甚至不运行 Claude CLI 进程。

## 4. 竞品参考实现

### 4.1 推荐参考

| 项目 | 参考什么 | 链接 |
|---|---|---|
| **better-ccflare** | OAuth token 管理（health check + auto refresh + failover） | https://github.com/tombii/better-ccflare |
| **cc-gateway** | 设备指纹归一化（40+ env 维度替换） | https://github.com/motiful/cc-gateway |
| **sub2api** | 号池管理 + session sticky + credential 注入 | https://github.com/Wei-Shaw/sub2api |
| **Meridian** | SDK query() + passthrough tools（备选方案参考） | https://github.com/rynfar/meridian |

### 4.2 我们的差异化

| 竞品做法 | 我们的差异 |
|---|---|
| 中心化号池（所有 token 在一台机器上） | **分布式 bridge（token 在账号所有者的本地机器上）** |
| 简单 round-robin / 软 sticky | **session-bridge 硬绑定 + 自路由禁止 + circuit breaker** |
| 切换 account 时 ForceCacheBilling 补偿 | **不切换——bridge 离线时返回 503，由 caller 决定** |
| 简单计费（按 token 收费） | **mutual credit（贡献额度获取额度）** |
| 单 provider | **多 team / 多 provider（已支持）** |

## 5. 实施路线

> API key bridge 真透传（plan-bridge-api）太简单，不单独立 phase，需要时随时加。

### 5.1 Phase 1: OAuth proxy bridge（核心能力）

Phase 1 拆为三个里程碑，便于定位问题。_(gpt52 R3 review #4: scope 过宽，拆里程碑隔离风险)_

**Phase 1 前置：Research tasks**
- [ ] RT1: 各平台 Claude CLI OAuth token 存储位置
- [ ] RT2: platform.claude.com token refresh 端点格式
- [ ] RT3: token 过期周期验证
- [ ] RT4: D0 streaming transport 方案确认（倾向方案 C'：WS 隧道 + bridge 直连 Anthropic）

**Phase 1A: Transport + Proxy（能跑通单 bridge 代理）**
- [ ] 新增 `plan-bridge-oauth` package
- [ ] TokenProvider 实现（读取 + 刷新 + 健康检查）
- [ ] HTTP reverse proxy（token 注入 + passthrough-with-normalization）
- [ ] 真 SSE streaming（WS 隧道或直连，取决于 RT4）
- [ ] 流式 usage 提取（TransformStream tee）
- [ ] `/v1/messages/count_tokens` 透传端点
- **验收**：单 bridge 能代理 Claude Code 请求到 Anthropic 并返回（含 streaming）

**Phase 1B: Routing + Hard-fail（pool 层集成）**
- [ ] Gateway 改造：按 bridge 类型分发 + `x-api-key` 认证 + header 透传
- [ ] Bridge 注册时声明 capabilities（supportsTools, supportsStreaming）
- [ ] Gateway 根据 capabilities 路由
- [ ] `Pool.selectBridge()` hard-fail 语义改造（D4，binding 失效返回 503，不 fallback）
- **验收**：多 bridge pool 环境下路由正确、session 硬绑定生效、bridge 离线返回 503

**Phase 1C: Fingerprint Normalization + E2E（上线前）**
- [ ] 设备指纹归一化：Header 层（bridge 自生成 UA/stainless）+ Body 层（`<env>` block 替换）— 参考 cc-gateway，不自造轮子
- [ ] Claude Code CLI 端到端验证（tool use、streaming、multi-turn、session resume）
- **验收**：远端 Claude Code 通过 pool 完成完整工作流，指纹一致

### 5.2 Phase 2: 增强（按需）

- [ ] 多账号 failover（bridge 内部管理多 token）
- [ ] CLI bridge 改进：更精细的 Anthropic chat-subset 适配
- [ ] SDK query() 作为备选 bridge 实现

## 6. 对现有代码的影响

### 6.1 保留不动

- `packages/plan-pool/src/team.ts` — 团队管理
- `packages/plan-pool/src/credit.ts` — 额度计算
- `packages/plan-pool/src/ledger.ts` — 记账
- `packages/plan-pool/src/circuit-breaker.ts` — 熔断器
- `packages/plan-pool/src/format-anthropic.ts` — CLI bridge 仍需使用
- `packages/plan-pool/src/format-openai.ts` — OpenAI 兼容仍需使用
- `packages/plan-bridge-cli/` — 保留作为 chat-only bridge

### 6.2 需要改造

- `packages/plan-pool/src/pool.ts` — ⚠️ `selectBridge()` binding 失效从 fallback 改为 hard-fail（D4 核心状态机反转）_(gpt52 R2 review #1)_
- `packages/plan-pool/src/api-gateway.ts` — 按 bridge 类型分发 + 透传模式 + 新端点 + 新认证头
- `packages/plan-pool/src/index.ts` — `invokeBridge()` 需支持透传请求 body
- `packages/plan-bridge-api/src/proxy.ts` — 真透传 + 真 streaming
- `packages/plan-bridge-api/src/bridge.ts` — 完整 body 解析

### 6.3 需要新增

- `packages/plan-bridge-oauth/` — 新的 OAuth proxy bridge 包
  - `src/token-manager.ts` — OAuth token 读取/刷新/健康检查
  - `src/proxy.ts` — HTTP reverse proxy（token 注入 + streaming pipe）
  - `src/bridge.ts` — A2A 注册 + 心跳 + capability 声明

## 7. Open Questions

### Research Tasks（D2 TokenProvider 实现前置）

- **RT1**: 各平台 Claude CLI OAuth token 存储位置（macOS Keychain / Linux keyring / `~/.config/claude/` / other）— 参考 cc-gateway、Meridian 源码
- **RT2**: platform.claude.com token refresh 端点格式 — 参考 Meridian `tokenRefresh.ts`、better-ccflare
- **RT3**: token 过期周期验证（社区报告 ~8h）

### Architecture Decisions（需在实施前确定）

1. **Streaming transport（D0）**：倾向 C'（WS 隧道 + bridge 直连 Anthropic）。C 原版因 bridge 无公网 IP 不可行，C' 通过现有 WS 通道解决 NAT 穿透。方案 A（gateway 代请求）为后备。
2. **Bridge capability 注册协议**：如何让 pool-node 区分 bridge 能力（`supportsTools`, `supportsStreaming`, `supportsStructuredInput`）？建议在 bridge register 时声明。
3. **两层转发策略**：Gateway = strict passthrough（不碰 body），OAuth Bridge = passthrough-with-normalization（仅替换 `<env>` block + header 自生成）。指纹归一化参考 cc-gateway，Phase 1C 做。_(CVO R3 + gpt52 R3 #1#2)_

### Open Questions

1. ~~**D0 方案 C 下的路由**~~ → 已明确：C' 方案通过现有 WS 通道做请求隧道，gateway 不需要直连 bridge HTTP 端口
2. **多客户端 session affinity**：Cursor/Cline 不发 `X-Claude-Code-Session-Id`，gateway 生成的 sticky token 用什么机制回传？（Cookie? 自定义 response header?）
3. **计费精度**：流式响应中 tee 提取 usage 的时机——是每个 SSE 事件都检查，还是只检查 `message_delta` 和 `message_stop` 事件？
4. **C' WS 帧协议设计**：需要定义 `proxy-request` / `proxy-sse-chunk` / `proxy-end` / `proxy-error` 帧格式，评估对现有 WsChannel 的改动量。_(gpt52 R2 review: 补充流生命周期协议最小约束，见下)_

### C' WS 隧道流协议最小约束 _(gpt52 R2 review #2 补充)_

快乐路径之外，必须在实施前定义以下协议要素，否则断连/并发/大流量场景会挂：

| 约束 | 说明 |
|---|---|
| **streamId** | 每个 proxy-request 分配唯一 streamId，后续所有 chunk/end/error 都带此 ID。同一 WS 连接上可能并发多个 stream（externalMaxConcurrency=2）|
| **abort** | client 断开（gateway 检测到 HTTP 连接关闭）→ gateway 发送 `{type: "proxy-abort", streamId}` → bridge 用 AbortController 取消上游 Anthropic fetch |
| **terminal frame** | 每个 stream 必须以 `proxy-end` 或 `proxy-error` 结束（exactly-once）。Gateway 用此判断 stream 生命周期结束，释放资源 |
| **bounded buffering** | bridge 侧 SSE chunk 积压超过阈值（如 64 个未确认帧）→ 暂停读取上游 Anthropic 响应（backpressure）。Gateway 侧同理：client 消费慢 → 通过 WS flow control 反压 |
| **超时** | gateway 发出 proxy-request 后，若 N 秒内未收到首个 proxy-sse-chunk → 视为 bridge 无响应 → 返回 504 给 client |
