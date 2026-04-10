---
feature_ids: [F014]
related_features: [F001, F012, F013]
topics: [routing, direct-url, relay, referral, invoke, performance, hub]
doc_kind: spec
created: 2026-04-05
layer: governance
owner_module: mesh-hub
status: complete
phase: 2
depends_on:
  - { id: F001, type: blocking }
  - { id: F012, type: blocking }
  - { id: F013, type: related }
evidence:
  - packages/mesh-hub/src/hybrid-routing.test.ts
  - packages/mesh-node/src/hybrid-routing.test.ts
adr_override: "F014 spec L87 放宽了 ADR Phase 2 门槛第1条（铲屎官判断），需正式更新 ADR"
---

# F014: Hybrid Routing — Direct URL + Hub Relay（混合路由）

> Status: complete | Owner: 砚砚
> Related ADR: `docs/decisions/2026-04-01-mesh-hub-mvp-communication-architecture.md`

## Why

当前所有 INVOKE 调用都经过 Hub 中转（relay）：

```
Caller → Hub（全量 payload 在内存中转）→ Target
Target → Hub（结果在内存中转）→ Caller
```

MVP 阶段这样做是正确的（ADR 决议：可治理性优先）。但随着服务增多和 payload 变大（音频、视频、大文件），Hub 中转成为不必要的瓶颈。

铲屎官指出：**如果服务有独立 URL（公网 IP / 域名），应该直接暴露给调用方发现和调用；没有独立 URL 的服务才通过 Hub 中转。**

## What

Hub 支持**混合路由模式**：节点注册时声明是否有独立可达 URL，调用方根据路由信息选择直连或走 Hub relay。

### 路由决策

| 场景 | 路由方式 | 数据流 |
|------|---------|--------|
| 目标节点有 `directUrl` | Direct（referral） | Caller → Target（直连，Hub 只签发 token） |
| 目标节点无 `directUrl` | Relay（当前行为） | Caller → Hub → Target |
| 直连失败 | Fallback to Relay | Caller → Hub → Target（自动回退） |

### 核心改动

1. **节点注册声明 `directUrl`**：
   - Config `allowedNodes` 新增可选 `directUrl` 字段（与 `callbackUrl` 独立）
   - `callbackUrl` 是 Hub 内部转发地址（可能是 127.0.0.1）
   - `directUrl` 是外部可达地址（公网 IP / 域名）

2. **CAPS / Directory 暴露路由信息**：
   - CAPS 响应新增 `directUrl` 字段（仅对已认证节点可见）
   - `/v1/directory` 响应新增 `routingMode: "direct" | "relay"`（不暴露具体 URL）

3. **MeshClient 支持直连调用**：
   - `client.invoke()` 检查 CAPS 返回的 `directUrl`
   - 有 directUrl → 先请求 L2 token → 直接调用目标节点
   - 直连失败 → 自动回退到 Hub relay
   - 无 directUrl → 走现有 Hub relay 路径

4. **审计对齐**：
   - 直连调用完成后，target node 向 Hub 上报调用结果摘要（traceId + status + durationMs）
   - Hub 审计日志同时覆盖 relay 和 direct 路径

### 安全约束

- 直连模式下 L2 token 验证**仍在目标节点执行**（MeshServer 已支持 `jwtVerify`）
- Hub 签发 L2 token 时不关心调用方式（relay/direct），token 结构不变
- `directUrl` 不暴露在匿名端点（`/v1/directory` 只返回 `routingMode`，不返回 URL）
- scope enforcement 不变：token 粒度的 scope 控制仍然由 Hub 决定

## Acceptance Criteria

- [x] AC-1: 节点配置 `directUrl` 后，CAPS 返回该字段（仅已认证节点可见）。
- [x] AC-2: `/v1/directory` 返回 `routingMode: "direct"` 或 `"relay"`（不暴露 URL）。
- [x] AC-3: `MeshClient.invoke()` 对有 `directUrl` 的目标自动走直连，跳过 Hub 数据中转。
- [x] AC-4: 直连失败时自动回退到 Hub relay，调用方无感知。
- [x] AC-5: 直连模式下 L2 token 在目标节点本地验证通过（与 relay 模式安全等级一致）。
- [x] AC-6: 直连调用完成后 Hub 审计日志有记录（traceId + 调用结果摘要）。
- [x] AC-7: 无 `directUrl` 的节点行为不变（纯 relay，向后兼容）。

## Dependencies

- F001（Security Foundation）— L2 token + Node B 本地验证 ✅
- F012（Public Discovery Layer）— CAPS / directory 端点 ✅
- F013（Open Node Registration）— 动态注册时可声明 directUrl（松耦合，不硬依赖）

## ADR Phase 2 启动门槛（参考）

> **⚠️ ADR Override Needed**: 本 feature 的设计与原 ADR 第1条门槛存在分歧。
> 铲屎官判断"有公网 URL 就应该能直连"，但原 ADR 要求"真实流量触发瓶颈"。
> 在实施前必须正式更新 ADR（`docs/decisions/2026-04-01-mesh-hub-mvp-communication-architecture.md`），
> 将铲屎官决定记录为门槛修订，而非在 feature spec 中私自放宽。

原 ADR 定义了 Phase 2（直连）的启动条件。本 feature 的设计已考虑这些约束：

1. ~~真实流量下 Hub Relay 触发瓶颈~~ → 铲屎官判断：不需要等瓶颈，有公网 URL 就应该能直连
2. 审计覆盖 100% → AC-6 要求直连路径也有审计
3. 撤销传播不退化 → L2 token 有 expiry，Hub 签发时控制
4. 直连失败回退有测试 → AC-4 要求自动回退

## Risk

| 风险 | 缓解 |
|------|------|
| 审计覆盖不一致 | AC-6 强制要求直连路径上报审计摘要 |
| 直连 URL 暴露给未授权方 | directUrl 仅在 CAPS 中返回（需 L1 认证） |
| 回退路径增加延迟 | 首次直连超时后缓存 fallback 标记，后续请求直接走 relay |
| 直连绕过 Hub scope 检查 | L2 token 在 Hub 签发，scope 已嵌入；Node 本地验证 |

## Open Questions

1. 直连调用的审计上报是同步还是异步？同步增加延迟，异步可能丢失。
2. `directUrl` 的健康检查由谁做？Hub 主动探测还是依赖 heartbeat？
3. 直连超时阈值应该多少？是否可配置？

## Needs Checklist

- [ ] 是否清晰区分了 directUrl（外部可达）和 callbackUrl（Hub 内部转发）？
- [ ] AC 是否覆盖了 relay/direct/fallback 三条路径？
- [ ] 安全等级是否与纯 relay 一致？
- [ ] 审计对齐是否有明确验收条目？
