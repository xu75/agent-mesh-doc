---
feature_ids: [F013]
related_features: [F001, F012]
topics: [registration, onboarding, hub, dynamic, whitelist, open-registration]
doc_kind: spec
created: 2026-04-05
layer: governance
owner_module: mesh-hub
status: complete
phase: 2
depends_on:
  - { id: F001, type: blocking }
  - { id: F012, type: blocking }
evidence: []
---

# F013: Open Node Registration（动态节点注册）

> Status: complete | Owner: 宪宪 | Completed: 2026-04-10
> Evolved from: F012 (Phase 2 "Self-service 准入")

## Why

当前 Hub 的节点注册是**静态白名单制**：每个节点的 nodeId、publicKey、capabilities、maxScopes 必须预先写在 `config.json` 的 `allowedNodes` 里。新增任何服务（如 F011 的 tts-bridge）都需要：

1. 编辑 Hub 的 `config.json`
2. 重启 Hub 进程

这与"Hub 可以挂各种服务"的设计意图矛盾。一个 mesh hub 应该让新服务自己来注册，而不是每次都要管理员改配置重启。

铲屎官原话："为何挂一个新的服务，需要改 server 的代码并重启呢？"

## What

Hub 支持**可配置的注册策略**，新节点可以运行时自注册，不需要修改配置或重启 Hub。

### 注册模式

| 模式 | 行为 | 适用场景 |
|------|------|---------|
| `whitelist` | 当前行为，只接受预配置节点（默认） | 高安全环境 |
| `open` | 任何持有有效 Ed25519 密钥的节点可直接注册 | 开发/测试/开放 mesh |
| `approval` | 节点发送注册请求，管理员审批后生效（Phase 2） | 生产环境 |

### 核心改动

1. **HELLO 路径扩展**：`/v1/hello` 在 `open` 模式下，如果 nodeId 不在白名单，自动创建条目（Trust-on-First-Use）
2. **注册时声明 capabilities**：HELLO body 新增可选字段 `capabilities` 和 `maxScopes`，open 模式下 Hub 接受节点自报能力
3. **Config 新增 `registrationMode`**：控制注册策略，默认 `whitelist` 保持向后兼容
4. **`/v1/discovery` 联动**：`registration.mode` 从 hardcoded `"whitelist"` 改为读取实际配置

### 安全约束

- `open` 模式下首次 publicKey 绑定后不可更改（防止身份劫持）
- 动态注册的节点 `maxScopes` 有全局上限（config 配置 `defaultMaxScopes`，默认 `["read"]`）
- `whitelist` 模式保持当前行为，零回退风险
- 审计事件：每次动态注册记录 `NODE_REGISTERED` 事件

## Acceptance Criteria

- [x] AC-1: Hub 配置 `registrationMode: "open"` 后，新节点仅凭 nodeId + publicKey + proof 即可 HELLO 成功注册。
- [x] AC-2: 动态注册的节点在 CAPS 和 /v1/directory 中可见，skills 与注册时声明一致（HELLO body 中提供 `capabilities` 字段即可）。
- [x] AC-3: `whitelist` 模式行为不变（向后兼容），未知节点仍返回 403。
- [x] AC-4: 重复 nodeId 注册（不同 publicKey）被拒绝，返回语义化错误码。
- [x] AC-5: `/v1/discovery` 的 `registration.mode` 反映实际配置值。
- [x] AC-6: 动态注册的节点 maxScopes 不超过 config 的 `defaultMaxScopes` 上限（请求超限 scope 时 TOKEN 返回 403）。
- [x] AC-7: 注册事件可审计（`NODE_REGISTERED` 事件包含 nodeId、publicKey、capabilities、timestamp，仅首次注册触发）。

## Dependencies

- F001（Security Foundation）— Ed25519 身份体系 ✅
- F012（Public Discovery Layer）— `/v1/discovery` 的 `registration.mode` 字段 ✅

## Risk

| 风险 | 缓解 |
|------|------|
| `open` 模式下恶意节点泛滥 | defaultMaxScopes 限制权限上限；可随时切回 whitelist |
| publicKey 冲突/劫持 | TOFU 后 publicKey 绑定不可更改 |
| 内存膨胀（大量节点注册） | 设置 `maxNodes` 上限；liveness 过期自动清理 |

## Open Questions

1. `open` 模式下节点 capabilities 完全自报，是否需要 Hub 做能力验证（probe）？
2. 动态注册的节点信息是否需要持久化（重启后保留）？还是仅内存态？
3. `approval` 模式的审批 UX 是什么？CLI 命令？API 端点？

## Needs Checklist

- [ ] 是否清晰区分了三种注册模式的行为差异？
- [ ] AC 是否都可通过日志/命令复现实证？
- [ ] 向后兼容性是否有明确验收条目？
- [ ] 安全约束是否足够防御 open 模式的风险？
