---
feature_ids: [F002]
related_features: [F001]
topics: [plugin-adapter, interoperability, clowder, openclaw]
doc_kind: spec
created: 2026-03-31
---

# F002: Plugin Adapter Dual-Stack (Clowder/OpenClaw)

> Status: complete | Owner: 砚砚

## Why

Phase 2 的异构 Runtime 接入需要先有一个稳定的语义桥接层。我们必须把本地 Runtime 的能力声明、调用参数、结果结构和标识体系，映射到 Agent Mesh 的 `NodeCapability` / `InvokePayload` / `RESULT` 语义，避免各 Runtime 直连时重复造轮子。

## What

交付 `@agent-mesh/adapter` 包，提供:

1. `ClowderPluginAdapter`
2. `OpenClawPluginAdapter`
3. `createPluginAdapter("clowder" | "openclaw")` 统一工厂

统一接口能力:

- 能力映射：runtime capability -> `NodeCapability[]`
- 调用映射：runtime invoke request <-> `InvokePayload`
- 标识映射：runtime target id <-> mesh `catId`
- 结果映射：mesh invoke result -> runtime result

## Current Contract (v0)

### Clowder

- `catId` 直接映射到 mesh `catId`
- `skills` 直接映射到 mesh `skills`
- invoke 字段保持原语义：`task/context/scope`

### OpenClaw

- `agentId` 映射到 `catId = openclaw:{agentId}`（防冲突）
- `abilities` 映射到 mesh `skills`
- `availability` 映射:
  - `ready -> online`
  - `busy -> degraded`
  - `offline -> offline`
- invoke 字段映射:
  - `instruction -> task`
  - `memory -> context`
  - `permissions -> scope`

## Acceptance Criteria

- [x] AC-1: `ClowderPluginAdapter` 能正确完成能力/调用/结果/标识映射
- [x] AC-2: `OpenClawPluginAdapter` 能正确完成能力/调用/结果/标识映射
- [x] AC-3: OpenClaw target `catId` 非 `openclaw:` 前缀时明确拒绝
- [x] AC-4: 适配器行为有自动化测试覆盖（`plugin-adapter.test.ts`）
- [x] AC-5: Hub-side adapter registration endpoint (`POST /v1/adapters/register`)

## Timeline

| Date | Event |
|------|-------|
| 2026-03-31 | F002 kickoff — Plugin Adapter dual-stack |
| 2026-03-31 | Adapter package merged (PR #10, `cfdc94a`) — AC 1-4 |
| 2026-03-31 | Hub registration endpoint merged (PR #11, `5e32148`) — AC-5 |

## Evidence

- 92 tests (64 hub + 26 node + 2 adapter), all passing
- Cross-family review: @opus approved both PRs

## Deferred

1. 多租户前缀策略（当前固定 `openclaw:`）
2. Runtime 版本协商与向后兼容矩阵
3. STREAM 模式语义映射
4. OpenClaw e2e — adapter → Hub registration → real cross-node invoke
