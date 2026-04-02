---
feature_ids: [F009]
related_features: [F003]
topics: [hub, identity, token, authorization]
doc_kind: spec
created: 2026-04-01
---

# F009: L2 Token Extension

> Status: complete | Owner: 宪宪

## Why

L2 invocation token currently has `aud: targetNodeId` but no cat/skill-level targeting. All three cats in Round 1 identified this as the core architectural risk: "authorization at node level, but routing needs cat/skill level". Adding `targetCatId` as an optional claim prepares the token for future cat/skill-level authorization without breaking existing flow.

## What

1. **targetCatId optional claim**: L2 token includes `targetCatId` when specified in TOKEN request
2. **Invoke validation**: If L2 has targetCatId, Hub verifies the invocation reaches the correct cat (not just the correct node)
3. **Backward compat**: targetCatId is optional; existing TOKEN requests without it continue to work

## Acceptance Criteria

- [x] AC-1: TOKEN request with targetCatId produces L2 with matching claim
- [x] AC-2: INVOKE with targetCatId-bearing token routes to correct cat on target node
- [x] AC-3: TOKEN request without targetCatId works as before (backward compat)
- [x] AC-4: Node B can verify targetCatId in L2 token locally

## Dependencies

F003 (multi-capability model — needed to know which cats exist on a node)

## Risk

- Adding claims to JWT increases token size (negligible)
- Future: may need per-cat scope policies (deferred)

## Open Questions

- Should targetCatId be required in Phase 2+ or always optional?
- Scope policy: node-level or cat-level?
