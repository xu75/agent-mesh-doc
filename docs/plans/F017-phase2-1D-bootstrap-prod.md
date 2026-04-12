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

**What**: 新增 `bootstrap.ts` 入口，采集当前 Node.js 运行时环境生成 partial fingerprint profile；SDK 级字段（user-agent、x-stainless-package-version）由首个真实 Claude Code 请求回填。

**设计原则**：
- **禁止用 `claude --version` 合成 user-agent** — CLI 版本 ≠ SDK 出站 UA，合成会产生伪指纹
- Bootstrap 只采集可靠的环境源：`process.platform`, `process.arch`, `process.version`, `os.version()`, shell 环境变量
- SDK 级字段（`userAgent`, `xStainlessLang`, `xStainlessPackageVersion`）标记为 `pending`，由首个真实请求回填（first-request backfill）
- 回填完成后触发 profile 持久化 + LKG 更新

**实现要点**：
- 采集 `process.platform` → `mapPlatformToStainlessOs()` → `xStainlessOs`
- 采集 `process.arch` → `xStainlessArch`
- 采集 `process.version` → `xStainlessRuntimeVersion`, `xStainlessRuntime = "node"`
- 采集 `os.version()`, `$SHELL` → `promptEnv`（workingDirectory 使用配置值或 cwd）
- `userAgent` 和 `xStainlessPackageVersion` 初始为空字符串，preflight 会报 missing
- 输出经 `validateFingerprintProfile` 校验的 JSON 文件（partial profile，preflight.ok = false 是预期的）
- 同时生成 `.last-known-good` 副本

**首请求回填机制**（在 proxy handler 中）：
- warn/bootstrap-warn 模式下，如果 `isClaudeCaller && !preflight.ok`，从入站请求 headers 中提取 `user-agent` 和 `x-stainless-package-version`
- 回填到 profile，重新校验 preflight
- 如果 preflight.ok 变为 true，持久化 profile + 触发策略升级

**红测先行**：
- bootstrap 输出包含环境字段（os/arch/runtime/promptEnv），SDK 字段为空
- 回填后 profile 通过 preflight 校验
- 输出文件可被 `loadProfileWithFallback` 正确加载
- 回填不修改非 Claude caller 的请求

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
| R1 | Bootstrap 环境采集不完整，SDK 字段需要回填 | Medium | 设计为 two-phase：环境采集 + 首请求回填；warn 模式允许 partial profile 通过 |
| R2 | 首请求回填提取的 header 不是 Claude Code SDK 的真实值 | High | 仅从 `isClaudeCaller` 请求（有 `<env>` marker）提取；非 Claude 请求不触发回填 |
| R3 | 真实 E2E 因 API 限流或 token 过期失败 | Medium | 止损门禁：max 3 consecutive failures → hard exit；5s cooldown；300s total timeout |
| R4 | bootstrap-warn → strict 转换后重启回退到 warn | High | 持久化 `_meta.policyState` 到 profile JSON；重启时读取并恢复 strict |
| R5 | Profile metadata `_meta` 字段与 `FingerprintProfile` 类型冲突 | Low | `_meta` 使用下划线前缀约定；`validateFingerprintProfile` 忽略未知字段 |

## Quality Gates

- [ ] Task 1: bootstrap 输出通过 preflight 环境字段校验 + loadProfileWithFallback 往返测试
- [ ] Task 1: 首请求回填后 profile 通过完整 preflight 校验
- [ ] Task 2: 真实 Claude Code session 完成至少一次 Read + Write + Bash 工作流
- [ ] Task 3: bootstrap-warn → strict 自动转换有红绿测试覆盖（含持久化 + 重启恢复）
- [ ] 全量门禁: plan-bridge-oauth + plan-pool 双包 test/lint 通过

## Execution Order

1. Task 1 (bootstrap CLI + first-request backfill) — 无依赖，可立即开始
2. Task 3 (auto policy transition + 持久化) — 依赖 Task 1 的 profile 格式和回填机制
3. Task 2 (real E2E) — 依赖 Task 1 + Task 3，作为最终验收
