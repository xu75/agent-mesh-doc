---
feature_ids: [F003]
related_features: [F001, F002]
topics: [hub, capability, discovery]
doc_kind: spec
created: 2026-04-01
---

# F003: Capability Model Refactor

> Status: spec | Owner: TBD

## Why

Current Hub assumes one-node-one-catId (config.ts AllowedNode has single `catId` + `skills`), but product needs multi-capability per node. The adapter interface already uses `NodeCapability[]` (plugin-adapter.ts), creating a data model mismatch. This is the #1 architectural risk identified by all cats in Round 1 discussion — if not fixed, future routing, discovery, and authorization will be forced into patch-style growth.

## What

1. **Hub registration model**: AllowedNode supports `capabilities: NodeCapability[]` (array of `{catId, skills}`) instead of single `catId + skills`
2. **CAPS endpoint**: Returns per-capability entries; supports filtering by `catId` and `skill` query params
3. **Adapter alignment**: `POST /v1/adapters/register` uses the same multi-capability model; `adapterByNode` data participates in CAPS responses (fix write-only bug)
4. **Protocol alignment**: Ensure `@agent-mesh/protocol` NodeCapability type is the single source of truth
5. **Backward compat**: Single `catId + skills` in config still works (sugar for single-element capabilities array)

## Acceptance Criteria

- [ ] AC-1: A node can register with multiple catId/skills via config
- [ ] AC-2: A node can register with multiple catId/skills via adapter endpoint
- [ ] AC-3: `GET /v1/caps` returns all capabilities across all nodes
- [ ] AC-4: `POST /v1/caps` with `{catId: "codex"}` returns only nodes hosting codex
- [ ] AC-5: `POST /v1/caps` with `{skill: "review"}` returns only nodes with review skill
- [ ] AC-6: adapterByNode data is reflected in CAPS responses
- [ ] AC-7: Backward compat: single catId+skills config still works
- [ ] AC-8: All existing tests pass (no regression)

## Dependencies

None (foundational change)

## Risk

- Config schema change may affect deployed Hub instances (mitigated by backward compat)
- CAPS response shape change may break existing clients (version or additive change)

## Open Questions

- Should capabilities be mutable at runtime (re-register to update)?
- Scope policy: per-node or per-capability?

## Discussion Source

Round 1: codex (adapterByNode write-only), gpt52 (data model mismatch), opus (CAPS filtering)
