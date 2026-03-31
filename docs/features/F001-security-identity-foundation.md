---
feature_ids: [F001]
related_features: []
topics: [security, identity, hub, w1-w2]
doc_kind: spec
created: 2026-03-30
---

# F001: Security & Identity Foundation (W1-W2)

> Status: in-progress | Owner: 宪宪

## Why

Agent Mesh 的一切跨节点交互都建立在身份与安全之上。没有可信身份就没有可信调用。W1-W2 先把地基打好，后续协议层、路由层、治理层才有根基。

## What

建立 Hub 骨架和三层身份体系（L0/L1/L2），实现节点注册、握手、重放防护和 mTLS，使两个节点能安全握手并通过 E1/E2 验收门禁。

## MVP 场景定义（已对齐 2026-03-30）

**Phase 1（W1-W6）— Clowder-AI × Clowder-AI**

两个 Cat Cafe 实例通过 Agent Mesh 互联，覆盖"任务分发"和"算力协同"两个使用场景（共用 INVOKE/RESULT 协议，差异仅在路由策略）。

验收场景：
1. 基础调用：Node A `@remote/nodeB/codex` → Node B 执行 → 结果返回
2. 权限拒绝：越权请求 → SCOPE_DENIED → 审计记录
3. 紧急撤销：Hub 撤销 Node B 证书 → 后续调用被拒 → <= 60s 传播
4. 审计回放：`mesh audit trace <traceId>` 输出完整时间线

**Phase 2（W7+）— 异构 Runtime 接入**

通过 Plugin Adapter 或 Native Protocol 支持非 Clowder-AI 的 Agent 系统接入。需前置产出 Adapter 接入契约文档。

## Acceptance Criteria

### E1 — 身份链完整性

- [x] AC-1: L0 密钥对生成 — 节点可生成 Ed25519 密钥对并持久化存储
- [x] AC-2: L1 证书签发 — Hub 接收节点 L0 公钥 + 注册请求 → 签发 JWT Mesh Certificate（EdDSA 签名，含 nodeId/pubkey/expiry）
- [x] AC-3: L1 证书验证 — 任意节点可用 Hub 公钥验证 L1 证书，篡改任一字段即拒绝
- [x] AC-4: L2 令牌签发 — Hub 根据 L1 证书签发 Invocation Token（JWT，含 scope/ttl/jti）
- [x] AC-5: L2 令牌验证 — 接收方可验证 L2 令牌的签名、scope、ttl，过期或越权即拒绝
- [x] AC-6: 全链完整性 — 篡改 L0/L1/L2 中任一层，调用链中的验证步骤立即拒绝（测试覆盖 >= 5 种篡改场景）

### E2 — 安全基线

- [ ] AC-7: mTLS 握手 — 双向 TLS 认证成功率 >= 99.5%（1000 次握手测试）
- [ ] AC-8: mTLS 拒绝 — 无证书或无效证书的连接 100% 被拒绝
- [x] AC-9: jti 重放防护 — 相同 jti 的 L2 令牌 100% 被拒绝（含并发重放场景）
- [x] AC-10: 过期令牌拒绝 — ttl 过期的令牌 100% 被拒绝
- [x] AC-11: 撤销证书拒绝 — Hub 撤销 L1 证书后，后续请求被拒绝
- [x] AC-12: HELLO/CAPS 握手 — 两个节点可完成 HELLO → CAPS 握手流程，交换能力声明

### 量化指标

| 指标 | 目标 | 测量方式 |
|------|------|----------|
| 握手成功率 | >= 99.5% | 1000 次自动化握手测试 |
| 重放拦截率 | 100% | 含并发场景的 jti 测试 |
| E1 篡改拒绝率 | 100% | 5+ 种篡改场景测试 |

## Task Breakdown

### W1 — 身份底座 + Hub 骨架

| Task | 描述 | 输入 | 产出 |
|------|------|------|------|
| T1 | 项目脚手架 | 决策文档 | monorepo 结构、pnpm workspace、tsconfig、biome |
| T2 | mesh-protocol 消息定义 | R9 协议最小集 | TypeScript 类型：6 种消息 + 11 错误码 |
| T3 | L0 — Ed25519 密钥生成 | — | `mesh-node` 密钥生成 + 文件持久化 |
| T4 | L1 — Mesh Certificate 签发 | T3 | Hub 签发 JWT，EdDSA 签名 |
| T5 | L2 — Invocation Token | T4 | Hub 签发带 scope/ttl/jti 的 JWT |
| T6 | Hub 骨架 | T1 | Fastify 服务、配置加载、健康检查、注册端点 |
| T7 | jti 重放检测 | T5 | 内存 Set + TTL 淘汰 |

### W2 — 协议消息 + 握手 + 安全

| Task | 描述 | 输入 | 产出 |
|------|------|------|------|
| T8 | HELLO 消息实现 | T2, T6 | 节点 → Hub 握手请求/响应 |
| T9 | CAPS 消息实现 | T8 | 能力查询请求/响应 |
| T10 | mTLS 配置 | T4 | 节点↔Hub、节点↔节点双向 TLS |
| T11 | 白名单节点注册 | T6 | Hub 启动配置允许注册的节点公钥列表 |
| T12 | E1/E2 测试套件 | T3-T11 | 全部 AC 的自动化测试 |

## Dependencies

- Ed25519 / JWT 库选择（`jose` 推荐，纯 JS 实现 EdDSA）
- Node.js >= 20（原生 crypto 支持 Ed25519）

## Risk

| 风险 | 缓解 |
|------|------|
| mTLS 配置复杂度超预期 | 提供一键证书生成脚本；W1 先用自签 CA |
| jti 内存缓存在 Hub 重启后丢失 | TTL 短（默认 5min），重启窗口内的重放风险可接受；后续持久化 |
| Ed25519 在旧 Node.js 版本不支持 | 锁定 Node.js >= 20；CI 验证 |

## Timeline

| Date | Event |
|------|-------|
| 2026-03-30 | W1 identity foundation merged to main (T1-T7) |
| 2026-03-31 | W2 HELLO client + CAPS endpoint merged (PR #1, T8/T9) |
| 2026-03-31 | Plan C approved — skip mTLS, go W3 with guardrails |
| 2026-03-31 | W3 INVOKE relay implemented (`616adc1`) |
| 2026-03-31 | W3 security hardening Red→Green (`f05fa9e`, `9eb8c71`), pending final re-review |

## Phase Progress Summary (2026-03-31)

### Completed

| Week | Deliverable | Evidence |
|------|-------------|----------|
| W1 | L0 Ed25519 keypair, L1 Mesh Cert, L2 Invocation Token, jti replay/expiry/revocation, Hub skeleton | 38 hub + 13 node tests |
| W2 | HELLO client, CAPS endpoint, L1/L2 type confusion P0 fix | PR #1 merged → `7b46c94` |
| W3 | INVOKE relay + security hardening (scope enforcement, Node B local verify, fail-closed) | commits `616adc1` + `f05fa9e` + `9eb8c71`, 61 tests (38 hub + 23 node), pending final @codex re-review |

### AC Status: 10/12

- E1 (AC-1~6): all complete — identity chain integrity
- E2 (AC-9~12): complete — replay/expiry/revocation/handshake
- E2 (AC-7/AC-8): **open** — mTLS bilateral auth, blocked by Plan C guardrails

### Open Items (遗留敞口)

1. **W3 final re-review + merge-gate pending**: `feat/w3-invoke` 已完成两轮 Red→Green 修复，等待 @codex 最终放行后开 PR #2 合入
2. **T10 mTLS**: Plan C guardrail requires completion within ≤5 days after first e2e success
3. **T12 quantitative tests**: 1000-handshake benchmark not yet run
4. **STREAM mode**: not implemented, current INVOKE is request/response only

### Handoff Notes

- **Worktree**: `/Users/xujinsong/VSCode/SynologyDrive/agent-mesh-w3-invoke` on branch `feat/w3-invoke`
- **Build/test**: `pnpm build` clean, 61/61 tests pass, biome clean
- **Next step**: @codex final re-review → receive-review → merge-gate → PR #2
- **After merge**: backfill T10 (mTLS) + T12 (quant tests) per Plan C guardrails

## Execution Decision (2026-03-31)

`@co-creator` approved **Plan C (Go with Guardrails)**:

1. Proceed to W3 (`INVOKE/RESULT`) immediately to validate core interoperability value.
2. Keep hard guardrails until mTLS is completed:
   - No external/untrusted network admission.
   - Do not claim E2 fully complete while AC-7/8 are open.
   - Mark E2 as partially complete in Mission Hub/workflow.
3. After first W3 end-to-end invocation success, return to complete T10 (mTLS) + T12 (quant benchmarks) within one milestone (target <= 5 days).
4. Phase 2 (heterogeneous runtime onboarding) is blocked until AC-7/8 pass.

## Open Questions

1. L1 证书的默认有效期？建议 30 天，支持续期
2. L2 令牌的默认 TTL？建议 5 分钟
3. Hub 自身的密钥对如何初始化和备份？
4. Phase 2 Adapter 接入契约文档（字段、版本协商、错误语义、兼容策略）何时立项？建议 W7 前产出
