---
feature_ids: [F015]
related_features: [F010, F004]
topics: [test-nodes, infrastructure, systemd, echo-node, reviewer-node, persistence]
doc_kind: spec
created: 2026-04-05
---

# F015: Resident Test Nodes（常驻测试节点）

> **Status**: in-progress | **Owner**: 宪宪 | **Priority**: P1

## Why

当前 echo-node 和 reviewer-node 是实验脚本（`experiments/e2-mesh-bridge/run.ts`）临时注入的，Hub 重启后全部消失。原因有三：

1. 实验脚本 `run.ts` 退出时默认撤回 config 改动（节点从白名单消失）
2. Hub 的节点存活状态是纯内存 `Map<string, number>`（`lastHeartbeatByNode`），重启清零
3. 测试节点进程不是常驻服务，没有 systemd 管理

铲屎官原话：
> "测试 node 应该作为常驻服务长期存在，甚至正式上线了也应该保留给新加入的 Node 测试使用吧？"

## What

将 echo-node 和 reviewer-node 从"实验脚本临时注入"升级为**常驻基础设施**：

1. **Config 永久化**：echo-node 和 reviewer-node 的条目永久写入 `deploy/config.example.json` 和服务器 `/etc/mesh-hub/config.json`，不再依赖实验脚本注入
2. **systemd 常驻**：为两个节点编写 `mesh-echo.service` 和 `mesh-reviewer.service` unit file，开机自启、失败重启
3. **自动恢复**：`PartOf=mesh-hub.service` 让 systemd 在 Hub 重启时自动重启测试节点，节点重新 HELLO 获取新 L1 证书
4. **Identity 持久化**：节点 identity 保存在 `/opt/mesh-hub/.echo-identity/` 和 `.reviewer-identity/`，重启后复用同一密钥对
4. **文档更新**：明确标注这些是常驻测试节点，供新 Node 接入时验证用

## Acceptance Criteria

- [x] AC-1: `deploy/config.example.json` 包含 echo-node 和 reviewer-node 的永久条目
- [x] AC-2: 服务器 `/etc/mesh-hub/config.json` 同步包含两个测试节点（publicKey 已对齐持久化 identity）
- [x] AC-3: `mesh-echo.service` 和 `mesh-reviewer.service` 部署到服务器，`systemctl status` 显示 active
- [x] AC-4: Hub 重启后（`systemctl restart mesh-hub`），两个测试节点在 30s 内自动恢复 online 状态（`PartOf` + `ExecStartPre=sleep 2`）
- [x] AC-5: `/v1/directory` 能看到两个测试节点（重启前后均可见）
- [ ] AC-6: 相关文档更新，说明测试节点的常驻用途

## Dependencies

- **Evolved from**: F010（Mesh-Cat Cafe Integration）— echo-node 和 reviewer-node 在 F010 创建
- **Related**: F004（Node Liveness）— heartbeat 机制保障重连

## Risk

| 风险 | 缓解 |
|------|------|
| 测试节点占用服务器资源 | echo-node 和 reviewer-node 极轻量，空闲时几乎零 CPU |
| 密钥管理：测试节点密钥需要持久化 | Identity 保存在 `/opt/mesh-hub/.{echo,reviewer}-identity/`，权限 600，`loadIdentity()` 自动复用 |

## Key Decisions

| # | 决策 | 理由 | 日期 |
|---|------|------|------|
| KD-1 | 用 `PartOf=mesh-hub.service` 而非修改 heartbeat 代码 | 零代码改动解决 Hub 重启后 L1 证书失效问题；heartbeat 401 重连是更大的改进留给后续 | 2026-04-05 |
| KD-2 | `ExecStartPre=/bin/sleep 2` 延迟启动 | 确保 Hub 完全就绪后再 HELLO | 2026-04-05 |

## Timeline

| 日期 | 事件 |
|------|------|
| 2026-04-05 | 立项 |
| 2026-04-05 | 实施：config 永久化 + systemd PartOf + identity 持久化 + 服务器部署验证 |
