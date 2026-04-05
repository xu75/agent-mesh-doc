---
feature_ids: [F003]
related_features: [F001, F002]
topics: [hub, capability, discovery]
doc_kind: spec
created: 2026-04-01
layer: governance
owner_module: mesh-hub
status: complete
phase: 1
depends_on:
  - { id: F001, type: blocking }
  - { id: F002, type: blocking }
evidence: [capability-model.test.ts]
---

# F003: Capability Model Refactor

> Status: complete | Owner: 宪宪

## Why

Current Hub assumes one-node-one-catId (config.ts AllowedNode has single `catId` + `skills`), but product needs multi-capability per node. The adapter interface already uses `NodeCapability[]` (plugin-adapter.ts), creating a data model mismatch. This is the #1 architectural risk identified by all cats in Round 1 discussion — if not fixed, future routing, discovery, and authorization will be forced into patch-style growth.

## What

1. **Hub registration model**: AllowedNode supports `capabilities: NodeCapability[]` (array of `{catId, skills}`) instead of single `catId + skills`
2. **CAPS endpoint**: Returns per-capability entries; supports filtering by `catId` and `skill` query params
3. **Adapter alignment**: `POST /v1/adapters/register` uses the same multi-capability model; `adapterByNode` data participates in CAPS responses (fix write-only bug)
4. **Protocol alignment**: Ensure `@agent-mesh/protocol` NodeCapability type is the single source of truth
5. **Backward compat**: Single `catId + skills` in config still works (sugar for single-element capabilities array)

## Acceptance Criteria

- [x] AC-1: A node can register with multiple catId/skills via config
- [x] AC-2: A node can register with multiple catId/skills via adapter endpoint
- [x] AC-3: `GET /v1/caps` returns all capabilities across all nodes
- [x] AC-4: `POST /v1/caps` with `{catId: "codex"}` returns only nodes hosting codex
- [x] AC-5: `POST /v1/caps` with `{skill: "review"}` returns only nodes with review skill
- [x] AC-6: adapterByNode data is reflected in CAPS responses
- [x] AC-7: Backward compat: single catId+skills config still works
- [x] AC-8: All existing tests pass (no regression)

## Timeline

| Date | Event |
|------|-------|
| 2026-04-01 | F003 merged to main (`240864d`) — multi-capability model + CAPS filtering + adapter alignment |

## Evidence

- Code path: `resolveCapabilities()` + per-capability CAPS flattening in `packages/mesh-hub/src/hub.ts`
- Test suite: `packages/mesh-hub/src/capability-model.test.ts` (14 tests)
- Verified on 2026-04-01: `pnpm --filter @agent-mesh/hub exec node --test dist/capability-model.test.js` → pass

## Deferred

1. Runtime capability mutation policy（是否允许热更新）
2. Scope policy粒度（per-node vs per-capability）
