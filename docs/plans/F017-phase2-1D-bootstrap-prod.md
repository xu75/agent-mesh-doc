---
feature_ids: ["F017"]
topics: ["phase-1d", "bootstrap", "production-validation", "fingerprint-capture"]
doc_kind: "plan"
created: "2026-04-12"
---

# F017 Phase 2 — Phase 1D: Bootstrap & Production Validation

## Goal

Bridge Phase 1A-1C 代码已完成，但从"测试通过"到"生产可用"仍有三个缺口：

1. 指纹 profile 需要手动创建 JSON 文件 — 没有自动化采集工具
2. 所有 E2E 测试使用 mock 响应 — 没有真实 Claude Code → pool → bridge → api.anthropic.com 验证
3. bootstrap-warn → strict 策略转换未自动化 — 需要手动改环境变量

Phase 1D 目标：填补这三个缺口，使 oauth-proxy bridge 可以一键部署并自动进入生产就绪状态。

## Scope

| AC | 描述 | 包 |
|---|---|---|
| AC-P2-23 | `plan-bridge-oauth bootstrap` 采集当前环境生成 fingerprint profile | plan-bridge-oauth |
| AC-P2-24 | 真实 Claude Code session 通过 pool + bridge 完成 Read/Write/Bash | plan-bridge-oauth + plan-pool |
| AC-P2-25 | Bridge 首次 profile 验证成功后自动从 warn 升级到 strict（含持久化） | plan-bridge-oauth |

## Task Breakdown

### Task 1: Fingerprint Profile Bootstrap CLI (AC-P2-23)

**What**: 新增 `bootstrap.ts` 入口，采集当前 Node.js 运行时环境生成 partial fingerprint profile；SDK 级字段（user-agent、x-stainless-package-version）通过受信校准请求回填。

**设计原则**：
- **禁止用 `claude --version` 合成 user-agent** — CLI 版本 ≠ SDK 出站 UA，合成会产生伪指纹
- Bootstrap 只采集可靠的环境源：`process.platform`, `process.arch`, `process.version`, `os.version()`, shell 环境变量
- SDK 级字段（`userAgent`, `xStainlessLang`, `xStainlessPackageVersion`）标记为 pending，由受信校准请求回填
- **LKG 只在回填完成且 preflight.ok=true 后才写入** — partial profile 禁止晋升为 LKG

**实现要点**：
- 采集 `process.platform` → `mapPlatformToStainlessOs()` → `xStainlessOs`
- 采集 `process.arch` → `xStainlessArch`
- 采集 `process.version` → `xStainlessRuntimeVersion`, `xStainlessRuntime = "node"`
- 采集 `os.version()`, `$SHELL` → `promptEnv`（workingDirectory 使用配置值或 cwd）
- `userAgent` 和 `xStainlessPackageVersion` 初始为空字符串，preflight 会报 missing
- 输出 partial profile JSON（preflight.ok = false 是预期的）
- **不生成 `.last-known-good` 副本** — partial profile 不是 "known good"
- 生成一次性校准 nonce，写入 `_meta.bootstrapNonce`

**受信校准回填机制**（替代无鉴权的 first-request backfill）：

信任边界三重约束：
1. **Nonce 验证**：回填仅在请求携带 `x-bootstrap-nonce` header 且值匹配 `_meta.bootstrapNonce` 时触发
2. **一次性**：回填成功后立即清除 nonce，后续请求无法再触发回填
3. **时间窗口**：bootstrap-warn 模式下回填窗口默认 5 分钟（`BOOTSTRAP_WINDOW_MS`），超时后即使 nonce 正确也拒绝回填

回填流程：
- bootstrap-warn 模式下，如果请求携带有效 nonce 且在时间窗口内：
  - 从入站请求 headers 提取 `user-agent` 和 `x-stainless-package-version`
  - 回填到 profile，重新校验 preflight
  - 如果 preflight.ok=true：持久化 profile + 写入 LKG + 清除 nonce + 触发策略升级
  - 如果 preflight.ok=false：回填失败，保留 nonce 允许重试（在窗口内）

校准自动化：
- `bootstrap.ts --calibrate` 模式：生成 profile 后，自动发送一个携带 nonce 的最小请求通过 bridge
- 操作员也可手动：`curl -H "x-bootstrap-nonce: <nonce>" ...` 发送校准请求

**红测先行**：
- bootstrap 输出包含环境字段（os/arch/runtime/promptEnv），SDK 字段为空，无 LKG 文件
- 携带正确 nonce 的请求触发回填，回填后 profile 通过 preflight 校验
- 携带错误 nonce 的请求不触发回填
- 超过时间窗口的请求即使 nonce 正确也不触发回填
- 回填成功后 nonce 被清除，第二次请求不再触发回填
- 回填成功后 LKG 文件被写入
- 非 Claude caller 的请求不触发回填（即使携带 nonce）

### Task 2: Real Claude Code E2E Validation (AC-P2-24)

**What**: 在本地启动 pool + bridge 全栈，用真实 Claude Code CLI 发送请求，验证 Read/Write/Bash tool use 工作流。

**实现要点**：
- 编写 `e2e-real.sh` 脚本：启动 pool → 启动 bridge（bootstrap-warn 模式）→ 配置 Claude Code 环境变量 → 执行测试请求 → 验证响应 → 清理
- 验证项：tool use 响应正确、fingerprint headers 一致、session sticky、usage 记账
- 此测试不纳入 CI（需要真实 OAuth token），作为手动验收脚本

**止损门禁（硬约束）**：
- `MAX_CONSECUTIVE_FAILURES = 3`：连续 3 次请求失败 → hard exit，不再重试
- `COOLDOWN_BETWEEN_REQUESTS = 5s`：请求间最小间隔，防止高频打上游
- `TOTAL_TIMEOUT = 300s`：整个 E2E 运行总超时，超时即 abort
- `AUTO_SUSPEND_CHECK`：每次请求前检查 bridge 是否进入 SUSPENDED_FINGERPRINT，如果是则 abort 并报告
- 脚本退出时无论成功失败都执行 cleanup（kill pool/bridge 进程）

**前置条件**：
- 有效的 Claude Max OAuth token
- Task 1 的 bootstrap 工具已完成
- Task 3 的 auto policy transition 已完成

### Task 3: Auto Policy Transition (AC-P2-25)

**What**: Bridge 启动时如果 policy=bootstrap-warn，在首次成功处理 Claude caller 请求后自动升级到 strict，并持久化策略状态。

**实现要点**：
- 新增 `FingerprintPolicy` 值 `"bootstrap-warn"`
- bootstrap-warn 行为等同 warn，但在首次 `preflight.ok && isClaudeCaller` 后：
  - 将运行时 policy 切换为 strict
  - 日志记录策略升级事件
  - 触发 pool re-registration（更新健康状态）
- 如果 bootstrap 阶段持续失败（超过配置时间窗口），保持 warn 并告警

**持久化（重启不回退）**：
- 策略升级时，在 profile JSON 中写入 metadata：
  ```json
  { "_meta": { "policyState": "strict", "verifiedAt": "2026-04-12T22:00:00Z" } }
  ```
- Bridge 启动时检查 profile metadata：
  - 如果 `_meta.policyState === "strict"` 且 profile preflight.ok → 直接以 strict 启动
  - 如果 profile 损坏/缺失 → 回退到 LKG + warn 模式（安全降级）
  - 如果 `_meta` 不存在 → 按环境变量配置的 policy 启动（向后兼容）

**红测先行**：
- bootstrap-warn 首次成功后 policy 变为 strict
- bootstrap-warn 下 preflight 失败不升级
- 升级后的 strict 行为与直接配置 strict 一致
- 持久化后重启读取 `_meta.policyState` 直接进入 strict
- profile 损坏时重启回退到 warn（不 crash）

## Risk Matrix

| ID | Risk | Impact | Mitigation |
|---|---|---|---|
| R1 | Bootstrap 环境采集不完整，SDK 字段需要回填 | Medium | 设计为 two-phase：环境采集 + 受信校准回填；bootstrap-warn 模式允许 partial profile 通过 |
| R2 | 校准 nonce 泄露导致非受信请求触发回填 | Medium | 三重约束：nonce 仅在 bootstrap 输出中显示（不持久化到日志）+ 一次性消耗 + 5 分钟时间窗口；攻击者需同时获得 nonce 值、bridge 网络访问、且在窗口内发送 Claude caller 请求 |
| R3 | 真实 E2E 因 API 限流或 token 过期失败 | Medium | 止损门禁：max 3 consecutive failures → hard exit；5s cooldown；300s total timeout |
| R4 | bootstrap-warn → strict 转换后重启回退到 warn | High | 持久化 `_meta.policyState` 到 profile JSON；重启时读取并恢复 strict |
| R5 | Profile metadata `_meta` 字段与 `FingerprintProfile` 类型冲突 | Low | `_meta` 使用下划线前缀约定；`validateFingerprintProfile` 忽略未知字段 |
| R6 | Partial profile 被误存为 LKG，后续 strict 模式使用残缺指纹 | High | LKG 写入仅在回填完成且 `preflight.ok=true` 后触发；bootstrap 阶段显式禁止生成 `.last-known-good` 副本 |

## Quality Gates

- [ ] Task 1: bootstrap 输出包含环境字段（os/arch/runtime/promptEnv），SDK 字段为空，无 LKG 文件
- [ ] Task 1: 携带正确 nonce 的校准请求触发回填，回填后 profile 通过完整 preflight 校验 + LKG 写入
- [ ] Task 1: 错误 nonce / 超时 nonce / 已消耗 nonce 均不触发回填（红测覆盖）
- [ ] Task 2: 真实 Claude Code session 完成至少一次 Read + Write + Bash 工作流
- [ ] Task 3: bootstrap-warn → strict 自动转换有红绿测试覆盖（含持久化 + 重启恢复）
- [ ] 全量门禁: plan-bridge-oauth + plan-pool 双包 test/lint 通过

## Execution Order

1. Task 1 (bootstrap CLI + 受信校准回填) — 无依赖，可立即开始
2. Task 3 (auto policy transition + 持久化) — 依赖 Task 1 的 profile 格式和回填机制
3. Task 2 (real E2E) — 依赖 Task 1 + Task 3，作为最终验收
