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
| AC-P2-25 | Bridge 首次 profile 验证成功后自动从 warn 升级到 strict | plan-bridge-oauth |

## Task Breakdown

### Task 1: Fingerprint Profile Bootstrap CLI (AC-P2-23)

**What**: 新增 `bootstrap.ts` 入口，运行时采集当前 Node.js 环境 + Claude Code CLI 版本信息，生成 `fingerprint-profile.json`。

**实现要点**：
- 采集 `process.platform`, `process.arch`, `process.version` → 映射到 x-stainless-* 字段
- 探测 `claude --version` 获取 CLI 版本 → 构造 user-agent
- 采集 `os.version()`, shell 环境变量 → 构造 promptEnv
- 输出经 `validateFingerprintProfile` 校验的 JSON 文件
- 同时生成 `.last-known-good` 副本

**红测先行**：
- bootstrap 输出通过 preflight 校验
- 缺少 claude CLI 时 graceful fallback（user-agent 使用模板）
- 输出文件可被 `loadProfileWithFallback` 正确加载

### Task 2: Real Claude Code E2E Validation (AC-P2-24)

**What**: 在本地启动 pool + bridge 全栈，用真实 Claude Code CLI 发送请求，验证 Read/Write/Bash tool use 工作流。

**实现要点**：
- 编写 `e2e-real.sh` 脚本：启动 pool → 启动 bridge（bootstrap 模式）→ 配置 Claude Code 环境变量 → 执行测试请求 → 验证响应 → 清理
- 验证项：tool use 响应正确、fingerprint headers 一致、session sticky、usage 记账
- 此测试不纳入 CI（需要真实 OAuth token），作为手动验收脚本

**前置条件**：
- 有效的 Claude Max OAuth token
- Task 1 的 bootstrap 工具已完成

### Task 3: Auto Policy Transition (AC-P2-25)

**What**: Bridge 启动时如果 policy=bootstrap-warn，在首次成功处理 Claude caller 请求后自动升级到 strict。

**实现要点**：
- 新增 `FingerprintPolicy` 值 `"bootstrap-warn"`
- bootstrap-warn 行为等同 warn，但在首次 `preflight.ok && isClaudeCaller` 后：
  - 将运行时 policy 切换为 strict
  - 日志记录策略升级事件
  - 触发 pool re-registration（更新健康状态）
- 如果 bootstrap 阶段持续失败（超过配置时间窗口），保持 warn 并告警

**红测先行**：
- bootstrap-warn 首次成功后 policy 变为 strict
- bootstrap-warn 下 preflight 失败不升级
- 升级后的 strict 行为与直接配置 strict 一致

## Risk Matrix

| ID | Risk | Impact | Mitigation |
|---|---|---|---|
| R1 | Bootstrap 采集的环境数据不完整或不匹配真实 Claude Code 指纹 | High | 采集后立即 preflight 校验；与已知 Claude Code 请求 header 对比验证 |
| R2 | 真实 E2E 因 API 限流或 token 过期失败 | Medium | 使用小请求（短 prompt）；确保 token 刷新机制工作；脚本支持重试 |
| R3 | bootstrap-warn → strict 转换时机不当，阻断正常流量 | Medium | 转换仅在首次成功后触发；LKG fallback 提供安全网；支持手动回退到 warn |
| R4 | Claude CLI 版本更新导致 user-agent 格式变化 | Low | user-agent 采用模板 + 版本号拼接，不硬编码完整字符串 |

## Quality Gates

- [ ] Task 1: bootstrap 输出通过 preflight + loadProfileWithFallback 往返测试
- [ ] Task 2: 真实 Claude Code session 完成至少一次 Read + Write + Bash 工作流
- [ ] Task 3: bootstrap-warn → strict 自动转换有红绿测试覆盖
- [ ] 全量门禁: plan-bridge-oauth 全测试通过 + lint clean

## Execution Order

1. Task 1 (bootstrap CLI) — 无依赖，可立即开始
2. Task 3 (auto policy transition) — 依赖 Task 1 的 profile 格式
3. Task 2 (real E2E) — 依赖 Task 1 + Task 3，作为最终验收
