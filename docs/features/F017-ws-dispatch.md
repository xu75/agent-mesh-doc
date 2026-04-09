---
feature_ids: [F017]
related_features: [F001, F004, F013, F014]
topics: [mesh-hub, websocket, transport, nat-traversal, bridge-dispatch]
doc_kind: spec
created: 2026-04-08
layer: infrastructure
owner_module: mesh-hub
status: approved
depends_on:
  - { id: F017, type: blocking, note: "Phase 1.5 已完成的 bridge + pool-node 基础" }
  - { id: F001, type: blocking, note: "ED25519 identity + L1/L2 token 基础" }
  - { id: F004, type: related, note: "liveness/heartbeat 模型需适配 WS" }
review_history:
  - { date: 2026-04-08, reviewer: "砚砚/codex", result: "v1-v3 方案A review（5轮），暴露 pool层复制治理逻辑问题" }
  - { date: 2026-04-08, reviewer: "宪宪+砚砚+gpt52", result: "三猫全票推翻方案A，共识方案C（Hub dual-transport）" }
  - { date: 2026-04-08, reviewer: "CVO", result: "批准方案C，要求直接更新ADR而非打补丁" }
---

# F017-WS: Hub Dual-Transport（WebSocket 传输升级）

> Status: **approved** — CVO 2026-04-08 23:56 批准，待拆实施计划
> 背景：灰度测试时发现 SSH 隧道方案阻碍新成员上手，三猫架构讨论后确定 Hub 层传输升级

## 架构讨论记录

### 评估过的方案

| 方案 | 描述 | 结论 |
|------|------|------|
| A: Pool-node WS dispatch | pool-node 自建 WS 连接池，绕过 Hub 直连 bridge | **否决** — 在 pool 层复制一套 dispatch/timeout/cancel/revoke 治理逻辑，只解决 pool→bridge，通用节点仍需 SSH |
| B: Hub 全量 WS | Hub 强制所有节点改为 WS，移除 callback | **方向对但过激** — 一次性改动太大，不允许渐进迁移 |
| **C: Hub dual-transport** | Hub 传输层支持 WS + callback 并存，WS 优先 | **采纳** — 治理语义不变，传输层升级，所有节点受益，允许渐进迁移 |

### 关键洞察

- **方案 A 才是更大的 ADR 违反**（绕过 Hub），方案 C 只升级 Hub 传输层，保持 Hub Relay 语义（gpt52 提出）
- Hub 代码默认 host 是 `0.0.0.0`（`config.ts:110`），跨 NAT 的真正瓶颈是 callback 拓扑，不是 Hub 监听地址（gpt52 校正）
- pool-node 当前调用入口已经很干净（`MeshClient.invoke()`），做 Hub 层改动后 pool-node 零改动（三猫共识）

## 与 ADR 的关系

本 spec **正面更新** [Mesh Hub MVP 通讯架构决议](../decisions/2026-04-01-mesh-hub-mvp-communication-architecture.md)，不是打补丁加例外。

**更新内容**：
- Phase 1 约束从"Hub Relay via HTTP callback 是唯一 invoke 路径"
- 改为"**Hub Relay 是唯一 invoke 路径，传输层支持 HTTP callback 和 outbound WebSocket**"
- Hub 的治理职责不变：L2 token 校验、scope enforcement、jti replay 防护、revocation、audit trace
- WS 只替换"Hub 到目标节点的最后一跳"传输方式

## 问题

当前 invoke 调用链：

```
Caller → Hub /v1/invoke → Hub fetch(callbackUrl) → Target Node
```

Target Node 必须启动 HTTP callback server（MeshServer）并让 Hub 可达。当 Target 在 NAT 后面时，callback URL 不可达，需要 SSH 隧道。

问题：
- 每个跨 NAT 节点都需要手动建 SSH 隧道
- 隧道断连后 invoke 立刻失败，无自动恢复
- install.sh 无法自动化
- 不是正式产品该有的用户体验

## 方案：Hub Dual-Transport

### 核心思路

Hub 的 `/v1/invoke` 治理逻辑完全不变。仅把"最后一跳"的传输从 HTTP callback 扩展为**双传输**：

```
之前: Caller → Hub /v1/invoke → Hub fetch(callbackUrl) → Target
之后: Caller → Hub /v1/invoke → Hub dispatch:
                                  WS 连接可用 → ws.send(INVOKE)     ← 新增
                                  WS 不可用   → fetch(callbackUrl)  ← 保留
```

节点通过 outbound WebSocket 连接 Hub，Hub 管理连接池。

```
                    ┌─── Hub (:3010) ───┐
                    │ /v1/invoke        │
                    │ L2/scope/audit    │
   Caller ─────────┤                   ├──── WS ←── Node A (NAT 后面也行)
   (e.g. pool-node)│ dispatch:         │
                    │  WS优先/callback备│──── WS ←── Node B
                    └───────────────────┘
```

### 连接生命周期

```
Node 启动
  │
  ├─ 1. loadOrCreateIdentity()
  ├─ 2. HELLO handshake (HTTP POST /v1/hello)  ← 不变
  │     获取 meshCertificate
  │
  ├─ 3. WebSocket connect to Hub
  │     ws(s)://<HUB_URL>/v1/ws
  │     连接建立后发送: { type: "WS_REGISTER", nodeId, meshCertificate }
  │     Hub 验证 meshCertificate 后回复: { type: "WS_REGISTERED", ok: true }
  │     Hub 将此连接注册到 transport registry
  │
  ├─ 4. 心跳保活
  │     node → Hub: WebSocket ping (每 30s)
  │     Hub → node: WebSocket pong
  │     Hub 侧超时 → liveness 降级（与现有 F004 heartbeat 对齐）
  │
  ├─ 5. 接收 INVOKE（Hub push）
  │     Hub → node: { type: "INVOKE", traceId, invocationToken, sourceNodeId, payload }
  │     node 本地验签 invocationToken（L2 复验，与现有 MeshServer 逻辑一致）
  │     node → Hub: { type: "RESULT", traceId, status, payload, durationMs }
  │
  └─ 6. 断线重连
        指数退避: 1s → 2s → 4s → 8s → ... → max 60s
        重连后重新 HELLO + WS_REGISTER
        例外: close code 4001 (credential_revoked) → 不重连
```

### 协议消息

所有消息为 JSON 文本帧。

**认证**：WS_REGISTER 帧携带 `meshCertificate`（已通过 HELLO 获取的 L1 凭证），Hub 验证后绑定连接。不引入新的认证机制。

#### Node → Hub（WS 帧）

```typescript
// 注册 WS 传输通道
{ type: "WS_REGISTER", nodeId: string, meshCertificate: string }

// INVOKE 执行结果
{ type: "RESULT", traceId: string, status: "success" | "error",
  payload: string, durationMs: number }
```

#### Hub → Node（WS 帧）

```typescript
// 注册确认
{ type: "WS_REGISTERED", ok: true }

// 注册拒绝
{ type: "WS_REGISTERED", ok: false, error: string }

// 调用请求（Hub 在完成 L2 校验后 push）
{ type: "INVOKE", traceId: string, invocationToken: string,
  sourceNodeId: string,
  payload: { task: string, context?: string, scope: string[] } }

// 取消执行
{ type: "CANCEL", traceId: string }
```

### Hub Invoke 转发逻辑变更

现有 Hub `/v1/invoke` handler（`hub.ts:417-570`）的变更点：

```typescript
// 现有代码（~L536）:
const res = await fetch(targetNode.callbackUrl, { ... });

// 改为:
if (wsRegistry.has(targetNodeId)) {
  // WS 优先：通过已建立的 WS 连接 push INVOKE
  const result = await wsRegistry.dispatch(targetNodeId, forwardPayload, timeoutMs);
  // result 通过 WS RESULT 帧异步返回
} else if (targetNode.callbackUrl) {
  // Fallback: HTTP callback（兼容未迁移的节点）
  const res = await fetch(targetNode.callbackUrl, { ... });
} else {
  return reply.code(502).send({ error: "target node has no registered transport" });
}
```

**L2 验签保留**：WS 模式下，INVOKE 帧携带 `invocationToken`，目标节点仍需本地验签（与现有 MeshServer 行为一致）。如果验签失败，返回 RESULT(error)。

### 超时与 in-flight 请求处理

Hub 已有 `invokeTimeoutMs`（默认 10s），WS dispatch 需对齐：

1. **Per-invoke timeout**：Hub 为每个 WS dispatch 启动计时器。超时未收到 RESULT → 向 Caller 返回 504，向 node 发送 `{ type: "CANCEL", traceId }`。
2. **WS 断线时 pending invocation**：node 断线时，所有 in-flight INVOKE 立即失败（BRIDGE_DISCONNECTED）。
3. **CANCEL 到本地执行器的映射**（仅 bridge 相关）：
   - `plan-bridge-cli`：`child.kill('SIGTERM')` 终止 CLI 子进程
   - `plan-bridge-api`：`AbortController.abort()` 中止 fetch
   - 通用节点：handler 实现自行决定如何响应 CANCEL

### 安全考量

1. **认证复用**：WS_REGISTER 使用已有的 `meshCertificate`（L1 凭证），不引入新认证。Hub 验证 cert 有效性 + nodeId 匹配。
2. **L2 验签保留**：INVOKE 帧携带 `invocationToken`，target node 本地验签。治理不退化。
3. **wss:// 强制**：生产环境必须使用 wss://（TLS），灰度期可允许 ws://。
4. **凭证撤销 → 断链**：Hub 执行 cert revocation 时，立即关闭关联 WS 连接（close code 4001: credential_revoked）。
5. **连接身份绑定**：WS 连接建立时绑定 `{ nodeId, meshCertificate }`，dispatch 前校验身份一致。
6. **攻击面缩小**：使用 WS 的节点不需要开放任何端口，比 callback 模式攻击面更小。

### 变更范围

#### 1. mesh-hub（packages/mesh-hub）

| 文件 | 变更 |
|------|------|
| `hub.ts` | `/v1/invoke` handler 增加 WS dispatch 分支（WS 优先 → callback fallback）；新增 WS upgrade 路由 `/v1/ws`；WS_REGISTER 验证逻辑 |
| `ws-registry.ts` | **新增**：WS 连接注册表（nodeId → WebSocket mapping），dispatch + timeout + CANCEL + 断链清理 |
| `hub.test.ts` | 新增 WS dispatch 测试；现有 invoke 测试保持（callback 路径仍工作） |

#### 2. mesh-node（packages/mesh-node）

| 文件 | 变更 |
|------|------|
| `ws-channel.ts` | **新增**：outbound WS 连接管理（connect Hub、WS_REGISTER、接收 INVOKE、发送 RESULT、重连） |
| `server.ts` | MeshServer 保留（callback 兼容），新增 WsChannel 作为替代传输 |
| `client.ts` | MeshClient 不变（caller 侧不受影响） |
| `index.ts` | 导出 WsChannel |

#### 3. plan-bridge-cli（packages/plan-bridge-cli）

| 文件 | 变更 |
|------|------|
| `bridge.ts` | 移除 MeshServer（不再监听回调端口）；HELLO 后启用 WsChannel 连接 Hub；保留 identity、attestation、endpoint 检查、session 管理 |
| `bridge.test.ts` | 适配 WS 模式测试 |

#### 4. plan-bridge-api（packages/plan-bridge-api）

| 文件 | 变更 |
|------|------|
| `bridge.ts` | 同 bridge-cli：移除 MeshServer，改用 WsChannel |

#### 5. plan-pool（packages/plan-pool）— 不改

pool-node 仍走 `MeshClient.invoke()`，Hub 层透明处理传输。pool-node 零改动。

#### 6. install.sh / download.html

| 改动 | 说明 |
|------|------|
| `MESH_HUB_URL` | 保留（bridge 仍需连 Hub 做 HELLO + WS） |
| `POOL_NODE_URL` + `POOL_API_KEY` | 保留（bridge 注册到 pool 仍通过 HTTP） |
| 移除 SSH 隧道相关说明 | 不再需要 |
| 移除 `PLAN_BRIDGE_PORT` | 不再监听回调端口 |

### 不做什么

- **不做 WebSocket 流式响应**：INVOKE/RESULT 仍是完整 JSON 帧
- **不做多路复用**：一个 WS 连接对应一个 nodeId
- **不做 node-to-node 直连**：所有 invoke 仍经 Hub（Phase 2 scope）
- **不移除 HTTP callback**：保留作为 fallback，未迁移的节点继续工作
- **不改 MeshClient**：caller 侧接口不变

### AC（验收标准）

- [ ] AC-1: Hub 新增 `/v1/ws` WebSocket endpoint，接受节点 outbound 连接
- [ ] AC-2: 节点通过 WS_REGISTER（携带 meshCertificate）注册传输通道
- [ ] AC-3: Hub `/v1/invoke` 优先通过 WS dispatch，WS 不可用时 fallback callbackUrl
- [ ] AC-4: WS INVOKE 帧携带 invocationToken，目标节点本地验签（L2 复验不退化）
- [ ] AC-5: 节点断线后自动重连（指数退避），close code 4001 除外
- [ ] AC-6: Hub 侧 WS 连接状态与 liveness 联动（F004 heartbeat 对齐）
- [ ] AC-7: INVOKE 超时 → Hub 返回 504 + 向节点发送 CANCEL
- [ ] AC-8: CANCEL 到达 bridge 后终止本地执行（CLI: SIGTERM / API: abort）
- [ ] AC-9: WS 断线时 in-flight 请求立即失败
- [ ] AC-10: cert revocation → Hub 立即关闭关联 WS 连接（close code 4001）
- [ ] AC-11: plan-bridge-cli 使用 WsChannel 连接 Hub（不再启动 MeshServer）
- [ ] AC-12: plan-bridge-api 同上
- [ ] AC-13: 端到端测试：Consumer → pool-node → Hub → [WS] → bridge → CLI → response
- [ ] AC-14: 现有 HTTP callback 路径仍工作（未迁移节点兼容）
- [ ] AC-15: install.sh 和 download.html 移除 SSH 隧道说明

### 实施策略

分步实施（CVO 2026-04-08 批准）：

**C1（Hub 传输层）**：
1. Hub 增加 WS endpoint `/v1/ws` + ws-registry
2. Hub invoke handler 增加 WS dispatch 分支（WS 优先/callback fallback）
3. mesh-node 新增 WsChannel

**C2（Bridge 迁移）**：
4. plan-bridge-cli 移除 MeshServer，改用 WsChannel
5. plan-bridge-api 同上
6. install.sh / download.html 更新

**C3（文档收尾）**：
7. ADR 正面更新
8. 架构文档、部署指南、拓扑图同步

### 文档同步清单

| 文档 | 更新内容 | 优先级 |
|------|---------|--------|
| `docs/decisions/2026-04-01-mesh-hub-mvp-communication-architecture.md` | **正面更新**：Phase 1 传输层支持 WS + callback | **必须** |
| `docs/features/F017-plan-pool.md` 部署拓扑段 | 更新数据流图：bridge → Hub 改为 WS 直连 | **必须** |
| `docs/features/F017-deployment-guide.md` | 移除 SSH/tunnel 说明，更新 bridge 部署拓扑 | **必须** |
| `docs/ARCHITECTURE.md` + `ARCHITECTURE.zh-CN.md` | Phase 1 传输层描述更新 | **必须** |
| `packages/plan-pool/public/download.html` | 更新安装说明，移除 SSH | **必须** |
| `packages/plan-pool/public/download/install.sh` | 移除 SSH 相关，更新 .env 模板 | **必须** |
| `README.md` | 更新通讯架构描述 | 中 |
| `docs/discussions/2026-04-01-mvp-practical-runbook.md` | 更新序列图 | 中 |
| `docs/plans/F017-plan-pool-plan.md` | 检查与新架构冲突 | 中 |

### 工作量预估

| 组件 | 预估 |
|------|------|
| ws-registry.ts（mesh-hub 新增） | ~200 行（连接池 + dispatch + timeout + CANCEL + 清理） |
| hub.ts 改动（WS endpoint + invoke 双传输） | ~120 行 |
| hub.test.ts 新增测试 | ~250 行 |
| ws-channel.ts（mesh-node 新增） | ~180 行（outbound WS + WS_REGISTER + INVOKE 接收 + 重连） |
| server.ts 适配 | ~30 行 |
| bridge.ts 重构（bridge-cli） | ~60 行（移除 MeshServer，接入 WsChannel + CANCEL→SIGTERM） |
| bridge.ts 重构（bridge-api） | ~50 行（同上 + CANCEL→abort） |
| bridge 测试适配 | ~100 行 |
| install.sh + download.html | ~30 行 |
| 文档同步（ADR + 架构 + 拓扑 + 部署指南） | ~120 行 |
| **合计** | **~1140 行** |
