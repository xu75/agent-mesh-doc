---
feature_ids: [F001, F002]
topics: [mesh-hub, mvp, communication, topology, relay, p2p, governance]
doc_kind: decision
created: 2026-04-01
---

# Mesh Hub MVP 通讯架构决议（2026-04-01）

## 决议

MVP 阶段采用 **A: 纯 Hub Relay** 作为唯一通讯路径。  
中期演进目标保留 **B: Hub 治理 + 可选 P2P 数据通道**。  
不采用 **C: 纯 P2P 组织拓扑** 作为起步方案。

## Why

1. MVP 核心目标是先跑通可治理闭环，不是先做吞吐优化。
2. Hub Relay 天然满足统一审计、统一授权、统一撤销传播。
3. 当前代码和测试资产已经围绕 Hub Relay 建立，继续复用性最高。
4. 纯 P2P 的工程成本和治理复杂度在 6-8 周窗口内不可控。

## 目标形态

### Phase 1（当前 / MVP）

- 控制面与数据面都通过 Hub。
- 调用链固定为 `Node A -> Hub -> Node B`。
- 强制执行：L2 token 校验、scope enforcement、jti replay、cert revocation、audit trace。

### Phase 2（仅在门槛满足后）

- 控制面仍由 Hub 负责（Identity/Policy/Audit/Registry）。
- 数据面可按场景启用直连（P2P/直通传输），Hub 退化为控制与审计汇聚中心。
- 失败回退必须可用：任何直连失败均可自动退回 Hub Relay。

## Phase 2 启动门槛（全部满足）

1. 真实流量下，Hub Relay 连续两轮压测触发明确瓶颈（延迟/吞吐超目标）。
2. 审计覆盖保持 100%，且直连路径与 Relay 路径审计字段一致。
3. 撤销传播语义不退化（不允许出现“已撤销但仍可调用”的窗口扩大）。
4. 直连失败回退路径有自动化测试覆盖并通过回归门禁。

## 明确不做（MVP Scope 外）

- 纯 P2P 网络组织拓扑
- Hub 水平扩展与多 Region 治理
- 流式大包优化通道
- NAT 穿透工程化（STUN/TURN/QUIC）

## 现状确认（代码事实）

- `MeshClient.invoke()` 当前固定请求 Hub `/v1/invoke`。
- Hub `/v1/invoke` 校验后转发到目标 `callbackUrl`。
- Node Server 语义为”接收 Hub 转发调用并本地验签”。

这与本决议的 Phase 1 目标完全一致，无需新增实现即可作为 MVP 通讯基线。

---

## Amendment 1: Hub 传输层升级（2026-04-08）

> 批准人：CVO | 参与者：宪宪(opus)、砚砚(codex)、gpt52 | 全票通过
> 实施 spec：[F017-WS: Hub Dual-Transport](../features/F017-ws-dispatch.md)

### 修订内容

Phase 1 约束修订：

- **保持不变**：Hub Relay 是唯一 invoke 路径。调用链仍为 `Node A → Hub → Node B`。L2 token 校验、scope enforcement、jti replay、cert revocation、audit trace 全部保留。
- **新增**：Hub 到目标节点的”最后一跳”传输层，从仅支持 HTTP callback 扩展为支持 **outbound WebSocket + HTTP callback 双传输**。Hub dispatch 时 WS 优先，fallback callback。

### 修订原因

灰度测试暴露：HTTP callback 模型要求目标节点（如 plan-bridge）开放端口并对 Hub 可达。跨 NAT 场景（用户家庭网络、办公 WiFi）下 callback 不可达，需要 SSH 隧道 — 阻碍新成员上手，不可作为产品级方案。

三猫独立评估后一致认为：在 Hub 层升级传输（而非在应用层如 pool-node 绕过 Hub）是更正确的方案，因为：
1. 保持 Hub 治理中心地位，不在应用层复制治理逻辑
2. 所有 mesh 节点受益，不只是 bridge
3. 是 Hub Relay 的自然演进，不是 Phase 2 的直连

### 不影响的约束

- Phase 2 门槛不变 — 通用 node-to-node 直连（绕过 Hub）仍需满足 4 项门槛
- 目标节点 L2 本地验签不变 — WS 模式下 INVOKE 帧仍携带 invocationToken
- MeshClient 接口不变 — caller 侧零改动
