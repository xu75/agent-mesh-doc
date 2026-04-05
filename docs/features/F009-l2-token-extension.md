---
feature_ids: [F009]
related_features: [F003]
topics: [hub, identity, token, authorization]
doc_kind: spec
created: 2026-04-01
layer: governance
owner_module: mesh-hub
status: complete
phase: 1
depends_on:
  - { id: F003, type: blocking }
evidence: [l2-token-extension.test.ts]
---

# F009: L2 Token Extension

> Status: done | Owner: 宪宪 | Completed: 2026-04-02

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

## Timeline

| Date | Event |
|------|-------|
| 2026-04-01 | F009 spec written |
| 2026-04-02 | Implementation complete (112/112 tests) |
| 2026-04-02 | @codex review: P0 security fix (body injection), P2 AC-2 e2e test |
| 2026-04-02 | PR #14 merged (squash) → main `9de1eb9` |

## Evolution

→ **Evolved toward**: Per-cat scope policies (Phase 2+). Currently scope is node-level; a future feature could add cat-level scope validation once targeting is stable.

**Vision Guard** (2026-04-02, @gpt52): 正式放行 ✅

| 铲屎官原话 | 当前实际状态 | 匹配？ |
|-----------|------------|--------|
| "authorization at node level, but routing needs cat/skill level" | 调用侧 `targetCatId` 透传：`client.ts:108`、`bridge.ts:146`。Hub TOKEN/INVOKE 双处校验：`hub.ts:334`、`identity-service.ts:68`。Node B 只信 JWT claim：`server.ts:123`。端到端证明调用落到指定 cat：`l2-token-extension.test.ts:247`、`bridge.test.ts:150` | ✅ |

验证证据：hub 全套 127/127、node client 10/10、bridge 6/6 通过。

@gpt52 审查中发现 calling-side gap（MeshClient/MeshBridge 未传 targetCatId），已在 `4d57cf9` 修复闭环。

## Reflection

`docs/reflections/F009-reflection.md`

## Open Questions (deferred)

- Should targetCatId be required in Phase 2+ or always optional?
- Scope policy: node-level or cat-level?
