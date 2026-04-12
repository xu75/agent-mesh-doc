---
feature_ids: [F017]
related_features: [F001, F012]
topics: [plan-pool, oauth-proxy, claude-code-compatibility, streaming, fingerprint-normalization]
doc_kind: spec
created: 2026-04-11
layer: application
owner_module: plan-pool, plan-bridge-oauth
status: in-progress
phase: 2
depends_on:
  - { id: F017-phase-1.5, type: blocking, note: "Phase 1.5 的 team/credit/ledger/routing/circuit-breaker 全部复用" }
discussion: docs/discussions/F017-oauth-proxy-pivot.md
references:
  - { project: cc-gateway, url: "https://github.com/motiful/cc-gateway", use: "设备指纹归一化" }
  - { project: sub2api, url: "https://github.com/Wei-Shaw/sub2api", use: "号池管理 + session sticky + header 处理" }
  - { project: better-ccflare, url: "https://github.com/tombii/better-ccflare", use: "OAuth token 管理" }
---

# F017 Phase 2: OAuth Proxy Bridge — Claude Code 全功能兼容

> Status: **in-progress** | Owner: opus

## 1. 目标

让 Claude Code / Cursor / Cline 等工具通过 plan-pool 使用 Claude Max 订阅的**完整功能**，包括 tool use、streaming、extended thinking 等。

**非目标**：
- CLI bridge 改造（保留 chat-only，不动）
- API key bridge 真透传（太简单，需要时随时加）
- 对外商业代理服务（仅团队内部使用）

## 2. Bridge 类型

| bridge 类型 | 场景 | 机制 | Claude Code 兼容 |
|---|---|---|---|
| `cli-chat-bridge` | non-API 订阅，chat 场景 | `claude -p --tools ""` → 文本 | chat-only |
| `api-proxy-bridge` | API key 订阅 | HTTP 透传 | 完全兼容 |
| **`oauth-proxy-bridge`（新）** | **non-API 订阅，全功能** | **OAuth token 代理 → api.anthropic.com** | **完全兼容** |

## 3. 架构

```
Claude Code / Cursor / Cline
  │ ANTHROPIC_BASE_URL=https://pool.example.com
  │ ANTHROPIC_API_KEY=<team-api-key>
  ↓
plan-pool api-gateway (公网)
  ├─ 认证: x-api-key → member 身份
  ├─ 路由: session-bridge 硬绑定 → 选择 bridge
  ├─ 额度: mutual credit 检查
  ↓ WS 隧道 (NAT 穿透)
plan-bridge-oauth (owner 本地机器, NAT 后)
  ├─ OAuth token 管理 (读取 + 自动刷新)
  ├─ 请求转发: passthrough-with-normalization
  │   ├─ Header: 全新构建 (token 注入 + 指纹自生成)
  │   └─ Body: 仅替换 <env> block, 其余不碰
  ├─ SSE streaming: pipe 到 gateway → pipe 到 caller
  └─ Usage 提取: TransformStream tee → 上报 ledger
```

## 4. 接口规格

### 4.1 Gateway HTTP API

**认证**：支持两种 header（Claude Code 用 `x-api-key`）：
```
x-api-key: <team-api-key>        ← Claude Code 使用
Authorization: Bearer <team-key>  ← 兼容现有实现
```

**端点**：

| 方法 | 路径 | 说明 |
|---|---|---|
| POST | `/v1/messages` | Messages API（按 bridge 类型分发） |
| POST | `/v1/messages/count_tokens` | Token 计数（透传到 Anthropic） |

**分发逻辑**：
```typescript
// Gateway 层: strict passthrough — 不解析 body
if (bridge.type === "oauth-proxy" || bridge.type === "api-proxy") {
  return proxyPassthrough(req, reply, bridge);  // body 原样转发
} else {
  return handleCLIChatRequest(req, reply, bridge, auth);  // 降维为文本
}
```

### 4.2 Bridge 出站请求构建

**原则**：出站请求全新构建，不是入站请求的复制。参考 sub2api。

**Header 分类**：

| 类别 | Header | 来源 |
|---|---|---|
| **Caller 透传** | `anthropic-version`, `anthropic-beta`, `content-type`, `X-Claude-Code-Session-Id` | 从 caller 请求中取原值 |
| **Bridge 自生成** | `x-api-key`（注入 OAuth token）, `User-Agent`, `x-stainless-*` | Bridge 构造（与 owner 环境一致） |
| **丢弃** | `x-api-key`, `authorization`（caller 的）, `x-anthropic-billing-header` | Gateway 消费后不转发；billing header 是 fingerprint hash，必须去除 |

**Body 处理（passthrough-with-normalization）**：
- `system` 字段中的 `<env>...</env>` block → 替换为 owner 环境信息
- 其余 body 内容一律不碰（messages, tools, model, max_tokens, stream 等）
- 未找到 `<env>` block → 不修改 body

**`<env>` 定位**：
- 搜索范围：仅 `system` 字段（字符串或 `system[].text`）
- 匹配规则：正则 `<env>\s*\n[\s\S]*?</env>`
- **不搜索 `messages` 数组**（避免误改用户/工具内容）
- 实现参考 cc-gateway env 维度替换逻辑

### 4.3 OAuth Token 管理（D5: B+ 读为主 + 兜底刷新 — CVO 决策）

**核心决策**：Bridge 与 owner 的 Claude Code CLI 共享同一个 OAuth grant（Keychain 条目）。Bridge 以读为主，仅在 owner CLI 离线时兜底刷新。

**设计语义**：**eventual consistency**（高可用 + 有界收敛），不是零冲突保证。

```typescript
interface TokenProvider {
  getAccessToken(): Promise<string>;
  isHealthy(): boolean;
  refresh(): Promise<void>;
  onStatusChange(cb: (healthy: boolean) => void): void;
}
```

**Token 存储位置**（RT1 验证，与 owner CLI 共享）：

| 平台 | 存储 | 读取方式 |
|---|---|---|
| macOS | Keychain `Claude Code-credentials` | `security find-generic-password` |
| Linux/Windows | `~/.claude/.credentials.json` | 直接读文件 |

**刷新参数**（RT2/RT3 验证）：

| 参数 | 值 | 来源 |
|---|---|---|
| 刷新端点 | `POST https://platform.claude.com/v1/oauth/token` | 本机 CLI bundle 硬编码 |
| client_id | `9d1c250a-e61b-44d9-88ed-5944d1962f5e`（需可配置） | 本机 CLI bundle 硬编码 |
| Access token 有效期 | 8 小时（`expires_in: 28800`） | 本机验证 |
| Keychain 读取间隔 | 30 秒 | 与 CLI 自身 30s 读缓存窗对齐 |
| Grace period | 2 分钟 | token 过期后等待 CLI 刷新的宽限期 |
| 刷新重试间隔 | 30 秒 | 避免 429 |
| 连续失败→unhealthy | 3 次 | 容忍短暂网络波动 |

**B+ 四层护栏**：

```
1. READ-FIRST: 每 30s 从 Keychain 读最新 token
   → token 有效 → 直接用（CLI 管刷新，bridge 搭便车）

2. GRACE PERIOD: token 过期 → 等 2 分钟
   → 期间持续每 30s 读 Keychain（等 CLI 刷新）

3. LAST-RESORT REFRESH: 2 分钟后仍过期
   → 再读一次 Keychain（double-check CLI 是否刚刷过）
   → 确认仍过期 → bridge 自行刷新 → 写回 Keychain
   → CLI 下次读到新 token 也能用（eventual consistency）

4. 401 RECOVERY: 请求遇到 401/invalid_grant
   → 先重读 Keychain + 重试一次
   → 仍失败 → 标 unhealthy → 心跳上报 → 触发 session 解绑
```

**收敛预期**：CLI 有 30s Keychain 读缓存窗（gpt52 从 CLI bundle 验证）。在持续活跃的 CLI 会话中，通常会在一个缓存窗内收敛到新 token；极端 race 下可能出现一次短暂 401 或延迟收敛。

**行为要求**：
- Owner CLI 在线：bridge 只读不刷新，正常情况下 CLI 无感知（极端 race 除外）
- Owner CLI 离线：bridge 兜底刷新并写回，owner 下次开 CLI 正常
- 刷新后验证 scope 包含 `user:inference`（[#34785](https://github.com/anthropics/claude-code/issues/34785) 已知 scope 丢失风险）
- Token 不离开 bridge 进程（locality principle）
- 安装体验：一份 Auth，零额外授权步骤

### 4.4 Session-Bridge 硬绑定

**核心规则**：session 绑定 bridge 后不切换。这是正确性约束，不是优化。

**Session affinity key（优先级递降）**：
1. `X-Claude-Code-Session-Id`（Claude Code 自带）
2. `x-namespace`（兼容现有实现）
3. Gateway 自动生成的 sticky token（兜底，回传机制待定：Set-Cookie 或自定义 response header）

**绑定生命周期**：
```
首次请求 → 绑定 session → bridge-x
后续请求 → 强制走 bridge-x
解绑条件（仅以下四种）：
  1. bridge-x 心跳超时
  2. bridge-x 被 circuit breaker 熔断
  3. 绑定 TTL 过期（2h 无活动）
  4. bridge-x auth unhealthy（token 刷新失败 / 持续 401/403）
```

**Bridge 不可用时**：
```http
HTTP/1.1 503 Service Unavailable
X-Session-Expired: true
Content-Type: application/json

{
  "type": "error",
  "error": {
    "type": "session_expired",
    "message": "Service temporarily unavailable. Please retry with a new session."
  }
}
```

- 不偷偷切 bridge（切换 account 导致 prompt cache 失效 + 潜在调用失败）
- `error.type=session_expired` 是 gateway 扩展，非 Anthropic 标准
- 不识别的 client 按标准 retryable 503 处理

**`Pool.selectBridge()` 语义变更**：
- 现行为：binding 失效 → fallback round-robin（`pool.ts:96,110`）
- 新行为：binding 失效 → hard-fail 返回 503（不 fallback）
- 这是核心状态机反转

### 4.5 Streaming Transport（D6: 方案 C' — CVO 决策）

**方案**：Bridge 直连 Anthropic + WS 隧道回传（解决 bridge 无公网 IP）。Token 不离开 bridge 环境。

> ~~降级方案 A（gateway 代请求）~~：已排除——token 需传到 gateway，打破 locality principle。

```
Client → POST /v1/messages
  → Gateway
    → WS: {type: "proxy-request", streamId, body, headers}
      → Hub relay → Bridge
        → HTTP POST → api.anthropic.com
        ← SSE stream
      ← WS: {type: "proxy-sse-chunk", streamId, data} (逐 chunk)
      ← WS: {type: "proxy-end", streamId, usage}
    ← Hub relay → Gateway
  ← SSE events
Client
```

**流协议帧类型**：

| 帧类型 | 方向 | 说明 |
|---|---|---|
| `proxy-request` | gateway → bridge | 包含 streamId, body, headers |
| `proxy-sse-chunk` | bridge → gateway | SSE 事件数据 |
| `proxy-end` | bridge → gateway | 流正常结束，附带 usage 数据 |
| `proxy-error` | bridge → gateway | 流异常结束，附带 error |
| `proxy-abort` | gateway → bridge | client 断开，bridge 取消上游 fetch |

**协议约束**：

| 约束 | 规则 |
|---|---|
| streamId | 每个 proxy-request 唯一，后续帧都带此 ID |
| 多路复用 | 同一 WS 可并发多个 stream（externalMaxConcurrency=2） |
| terminal frame | 每个 stream 必须以 `proxy-end` 或 `proxy-error` 结束（exactly-once） |
| abort | client 断开 → gateway 发 `proxy-abort` → bridge 用 AbortController 取消 |
| backpressure | chunk 积压超 64 帧 → 暂停读取上游响应 |
| 首帧超时 | proxy-request 后 N 秒无首个 chunk → gateway 返回 504 |

**实施注意**（RT4 评估结论）：C' 是 mesh transport 基础设施升级，涉及 mesh-hub / mesh-node / pool 三层改造。现有 WS 是 unary INVOKE→RESULT，需扩展为 streaming session。backpressure 作为独立高风险子任务，不混入 MVP。

### 4.6 Usage 提取（计费）

**非流式**：从响应 body 的 `usage` 字段提取。

**流式**：从 SSE `message_delta` 事件中提取 usage（Anthropic 在流结束前发送）。使用 TransformStream tee——读取但不消费/修改事件流。

提取后通过旁路上报 pool-node ledger。

### 4.7 Bridge Capability 声明

Bridge 注册时声明能力，gateway 根据能力路由：

```typescript
// bridge register payload 新增字段
{
  bridgeId: string,
  provider: string,
  shareMode: "off" | "idle-only" | "capped",
  externalMaxConcurrency: number,
  capabilities: {
    supportsTools: boolean,             // oauth/api: true, cli: false
    supportsStreaming: boolean,          // oauth/api: true, cli: false
    supportsStructuredInput: boolean,    // oauth/api: true, cli: false — 结构化 system[]、thinking、非文本 content block
    bridgeType: "cli-chat" | "api-proxy" | "oauth-proxy"
  }
}
```

### 4.8 idle-only 策略参数

| 参数 | 旧值 | 新值 | 说明 |
|---|---|---|---|
| `ownerIdleTimeout` | 5min（运行时）/ 30min（测试） | **10min（统一）** | Owner 最后活跃后多久 bridge 对外开放 |
| `externalMaxConcurrency` | 1 | **2** | 允许同时代理的外部请求数 |

Bridge 检测到 429 rate limit → 心跳上报 capacity 降低 → pool 暂停路由。

## 5. 安全模型

### 5.1 安全层变化

| 安全层 | CLI bridge | OAuth bridge | 说明 |
|---|---|---|---|
| L0 能力 | `--tools ""` 禁用工具 | 透传（不限制） | Bridge 不执行工具，只代理 |
| L1 进程 | spawn + 隔离 | HTTP proxy（无子进程） | 简化 |
| L2 插件 | 禁用 hooks/MCP | 不适用 | 不运行 CLI |
| L3 额度 | --max-budget-usd | pool-node rate limit + 计费 | 改为上层控制 |
| L4 超时 | SIGTERM | AbortController | 等效 |

### 5.2 不变约束

- Bridge 运行在订阅所有者本地机器（locality principle）
- OAuth token 不离开 bridge 进程
- 调用方只有 team API key，无法直接访问 Anthropic
- 自路由禁止（AC-31）：`bridge.ownerMemberId === callerId → SELF_ROUTE_DENIED`
- Mutual credit / quota / routing 全部保留

## 6. 实施路线

### 6.0 前置 Research Tasks（已完成）

| ID | 任务 | 结论 | 报告 |
|---|---|---|---|
| RT1 | OAuth token 存储位置 | macOS Keychain `Claude Code-credentials` / Linux `~/.claude/.credentials.json` | `docs/discussions/F017-RT1-RT3-token-research.md` |
| RT2 | Token refresh 端点 | `POST https://platform.claude.com/v1/oauth/token`，client_id=`9d1c250a-...` | 同上（gpt52 从 CLI bundle 校正） |
| RT3 | Token 过期周期 | Access token 8h（`expires_in: 28800`），本机验证一致 | 同上 |
| RT4 | Streaming transport | C' 可行但是 mesh transport 升级（M-L），CVO 决策采用 C' | gpt52 评估 + 宪宪代码验证 |

**决策记录**：
- D5: B+（读为主 + 兜底刷新）— eventual consistency 语义，CVO 批准 2026-04-11
- D6: WS 隧道（方案 C'）— CVO 决策，token 不离开 bridge

### 6.1 Phase 1A: Transport + Proxy ✅

**目标**：单 bridge 能代理 Claude Code 请求到 Anthropic 并返回（含 streaming）。

**任务**：
- [x] 新增 `packages/plan-bridge-oauth/` 
  - `src/token-manager.ts` — TokenProvider 实现（B+ 四层护栏）
  - `src/proxy.ts` — HTTP reverse proxy + streaming pipe
  - `src/bridge.ts` — A2A 注册 + 心跳 + capability 声明
  - `src/env-normalizer.ts` — `<env>` block 替换（Phase 1C 引用）
- [x] TokenProvider: Keychain 读取 + grace period + 兜底刷新 + 写回 + 401 recovery
- [x] WS streaming 帧协议（D6 C'，mesh transport 升级）：
  - `mesh-hub/src/hub.ts` — 新增 streaming frame 处理（proxy-sse-chunk / proxy-end / proxy-error）
  - `mesh-hub/src/ws-registry.ts` — 从 unary PendingInvocation 扩展为 streaming session
  - `mesh-node/src/ws-channel.ts` — 新增 streaming frame 收发
  - `mesh-node/src/client.ts` — 新增 streaming invoke path（替代 unary `client.invoke()`）
- [x] Proxy: token 注入 + body passthrough + SSE streaming
- [x] Usage 提取: TransformStream tee
- [x] `/v1/messages/count_tokens` 透传

**验收标准**：
- [x] AC-P2-01: bridge 启动后自动读取本地 OAuth token
- [x] AC-P2-02: 非流式请求: Claude Code → bridge → Anthropic → 正确响应
- [x] AC-P2-03: 流式请求: SSE 全程 pipe，client 实时收到 chunk
- [x] AC-P2-04: tool_use blocks 正确透传（bridge 不解析/不执行）
- [x] AC-P2-05: usage 数据正确提取并上报
- [x] AC-P2-06: owner CLI 在线时 bridge 只读 token 不刷新，CLI 刷新后 bridge 30s 内拿到新 token
- [x] AC-P2-22: owner CLI 离线 + token 过期 + grace period 2min → bridge 兜底刷新并写回 Keychain
- [x] AC-P2-23: 请求遇 401 → 重读 Keychain + 重试一次 → 仍失败才标 unhealthy
- [x] AC-P2-16: `/v1/messages/count_tokens` 透传到 Anthropic，正确返回 token 计数
- [x] AC-P2-17: client 断开 → gateway 发送 `proxy-abort` → bridge 用 AbortController 取消上游 fetch
- [x] AC-P2-18: `proxy-request` 发出后 N 秒无首帧 → gateway 返回 504 Gateway Timeout

| 日期 | 事件 |
|---|---|
| 2026-04-11 | Phase 1A merged (PR #41) — WS transport + proxy bridge + gateway routing |
| 2026-04-12 | Phase 1A complete — AC-P2-06 (#43), AC-P2-22 (#44) token guardrail tests |
| 2026-04-12 | Phase 1B merged — capability routing, hard-fail, auth unhealthy, session affinity (PR #47) |
| 2026-04-12 | Phase 1C strategy freeze — fingerprint preflight/suspend/auto-refresh + local E2E baseline |
| 2026-04-12 | Phase 1C partial — env-normalizer, header/body fingerprint, LKG fallback, preflight guard (AC-P2-12/13/14); E2E + drift refresh pending |
| 2026-04-12 | Phase 1C complete — through-pool R/W/Bash sticky integration test + periodic refresh/drift detection + fingerprint route suspension |
| 2026-04-12 | Phase 1D kickoff — bootstrap CLI + real E2E validation + auto policy transition |

### 6.2 Phase 1B: Routing + Hard-fail

**目标**：pool 层集成，多 bridge 路由正确，session 硬绑定生效。

**任务**：
- [x] Gateway 改造: bridge 类型分发 + `x-api-key` 认证 + header 分类透传
- [x] Bridge capability 声明 + gateway 能力路由
- [x] `Pool.selectBridge()` hard-fail 语义（binding 失效 → 503，不 fallback）
- [x] 503 `session_expired` 错误响应

**验收标准**：
- [x] AC-P2-07: oauth-proxy bridge 注册时上报 capabilities
- [x] AC-P2-08: gateway 将 tool_use 请求路由到 oauth/api bridge，不路由到 cli bridge
- [x] AC-P2-09: session 硬绑定: 同一 session 的请求始终走同一 bridge
- [x] AC-P2-10: bridge 离线 → 503 session_expired（不 fallback 到其他 bridge）
- [x] AC-P2-11: 自路由禁止仍然生效
- [x] AC-P2-19: gateway 支持 `x-api-key` header 认证（Claude Code 默认方式）
- [x] AC-P2-20: bridge auth unhealthy（token 刷新失败 / 持续 401/403）→ 触发 session 解绑
- [x] AC-P2-21: 含 structured input（`system[]`、thinking、非文本 content block）的请求不路由到 cli bridge

### 6.3 Phase 1C: Fingerprint Normalization + E2E

**目标**：远端 Claude Code 通过 pool 完成完整工作流，指纹一致。

**任务**：
- [x] Header 层指纹: bridge 自生成 UA + x-stainless-*（参考 cc-gateway）
- [x] Body 层指纹: `<env>` block 替换为 owner 环境信息（参考 cc-gateway）
- [x] 指纹失败防护: preflight 校验 + 失败熔断（防高频异常上游请求）
- [x] 指纹漂移处理: 启动/定时自动刷新 + 漂移检测 + `last-known-good` 回滚
- [x] E2E 验证: through-pool Read + Write + Bash + streaming + multi-turn sticky 集成测试

**验收标准**：
- [x] AC-P2-12: 出站请求 User-Agent / x-stainless-* 与 owner 环境一致
- [x] AC-P2-13: 出站请求 body 中 `<env>` block 描述 owner 环境，非 caller 环境
- [x] AC-P2-14: 非 Claude Code caller（无 `<env>` block）请求不被修改
- [x] AC-P2-15: Claude Code E2E: Read + Write + Bash tool use 完整工作流通过（through-pool 集成测试）

**Phase 1C 策略冻结（2026-04-12, CVO 批准）**：

- owner 指纹来源：首次安装采集本机环境并固化配置；支持用户覆盖；源码仅保留模板，不写死机器特征。
- header canonical 字段：`user-agent`, `anthropic-version`, `x-stainless-lang`, `x-stainless-package-version`, `x-stainless-os`, `x-stainless-arch`, `x-stainless-runtime`, `x-stainless-runtime-version`。
- `<env>` 采用最小运行面整块替换（whole-block rewrite），不做逐行 patch；运行时必需集为 `workingDirectory/platform/shell/osVersion`。
- 缺失策略：`bootstrap-warn -> strict`；steady `strict` 下 T1 缺失返回 `503 bridge_fingerprint_unavailable`，且该请求不发上游。
- 风险防护：按 bridge 统计失败预算，超过阈值进入 `SUSPENDED_FINGERPRINT`，暂停新外部会话路由，待刷新探测恢复。
- 自动升级：启动时 + 定时（默认 6h）+ 漂移检测触发采集；候选配置经探测成功后晋升，失败回滚 `last-known-good`。
- 非 Claude caller 全程 passthrough（保持 AC-P2-14 不变）。
- E2E 运行环境：本地执行（Read + Write + Bash + multi-turn sticky）。

**采纳策略（执行必须遵守）**：

- 双层配置：`env`（完整采集快照）+ `prompt_env`（运行时最小必需集）。
- `<env>` 使用 whole-block rewrite（整块替换），不做逐行 patch。
- 指纹策略采用 `bootstrap-warn -> strict`；steady `strict` 下 T1 缺失直接 503 且不发上游。
- 仅允许 `captured profile` 或 `last-known-good` 参与运行时请求构造。
- 失败预算触发 `SUSPENDED_FINGERPRINT`，并暂停新外部会话路由。

**避免策略（禁止）**：

- 长期运行在 `warn` 模式（仅允许短时校准窗口）。
- 使用硬编码通用指纹默认值参与线上请求构造。
- 将 availability-first 的“无鉴权 fallback 转发”引入到本链路。
- 让 fallback 资产长期替代真实采集数据。

### 6.4 Phase 1D: Bootstrap & Production Validation

**目标**：从"测试通过"到"生产可用"——自动化指纹采集、真实 E2E 验证、策略自动升级。

**任务**：
- [ ] 指纹 profile bootstrap CLI: 采集当前环境生成 `fingerprint-profile.json`
- [ ] 真实 Claude Code E2E: pool + bridge 全栈部署，真实 API 请求验证
- [ ] bootstrap-warn → strict 自动策略转换

**验收标准**：
- [ ] AC-P2-23: `plan-bridge-oauth bootstrap` 采集当前环境生成有效 fingerprint profile
- [ ] AC-P2-24: 真实 Claude Code session 通过 pool + bridge 完成 Read/Write/Bash
- [ ] AC-P2-25: Bridge 首次 profile 验证成功后自动从 warn 升级到 strict

**详细计划**：见 `docs/plans/F017-phase2-1D-bootstrap-prod.md`

### 6.5 Phase 2: 增强（按需）

- [ ] 多账号 failover（bridge 内部管理多 token）
- [ ] CLI bridge 改进: Anthropic chat-subset 适配
- [ ] SDK query() 备选 bridge 实现

## 7. 代码影响

### 需改造

| 文件 | 改动 |
|---|---|
| `mesh-hub/src/hub.ts` | WS 端点新增 streaming frame 处理（D6 C'） |
| `mesh-hub/src/ws-registry.ts` | unary PendingInvocation → streaming session state（D6 C'） |
| `mesh-node/src/ws-channel.ts` | 新增 streaming frame 收发 + proxy-abort 处理（D6 C'） |
| `mesh-node/src/client.ts` | 新增 streaming invoke path（替代 unary `invoke()`）（D6 C'） |
| `plan-pool/src/pool.ts` | `selectBridge()` binding 失效 → hard-fail（状态机反转） |
| `plan-pool/src/api-gateway.ts` | bridge 类型分发 + 透传 + 新端点 + `x-api-key` 认证 + SSE relay |
| `plan-pool/src/index.ts` | streaming invokeBridge path（替代 unary `invokeBridge()`） |
| `plan-bridge-api/src/proxy.ts` | 真透传 + 真 streaming |
| `plan-bridge-api/src/bridge.ts` | 完整 body 传递 + capability 声明 |

### 需新增

| 包/文件 | 职责 |
|---|---|
| `packages/plan-bridge-oauth/` | 新 OAuth proxy bridge 包 |
| `src/token-manager.ts` | OAuth token 读取/刷新/健康检查 |
| `src/proxy.ts` | HTTP reverse proxy + streaming pipe |
| `src/bridge.ts` | A2A 注册 + 心跳 + capability |
| `src/env-normalizer.ts` | `<env>` block 定位 + 替换 |

### 不动

team.ts, credit.ts, ledger.ts, circuit-breaker.ts, format-anthropic.ts, format-openai.ts, plan-bridge-cli/
