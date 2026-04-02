---
feature_ids: [F009]
topics: [hub, identity, token, security, reflection]
doc_kind: reflection
created: 2026-04-02
---

# F009 Reflection Capsule

## What Worked

- **TDD discipline held**: Red tests first confirmed P0 injection bug was real before fixing; green tests confirmed the fix
- **Cross-cat review caught P0 early**: @codex found the `targetCatId` body injection vulnerability before merge — the security boundary was a non-obvious edge case that static analysis alone would have missed
- **AC-4 e2e test structure**: Using `MeshServer.start()` before Hub creation (to get the port dynamically) enabled a clean end-to-end integration test without circular dependencies
- **Backward compat by default**: Making `targetCatId` optional throughout meant zero regression on existing 103 tests

## What Failed

- **P0 security gap in original implementation**: The `if (typeof claims.targetCatId === "string")` guard was correct for the positive case but left the body field unguarded in the negative case. The correct pattern is always `body.field = claims.field ?? undefined` (unconditional override from verified source).
- **AC-2 test was logically incomplete**: Accepted 502/503 as proof of "routing to correct cat" — it only proved "Hub didn't reject at the auth layer". A real callback was needed to prove end-to-end routing correctness.

## Trigger Missed

- When assigning from a JWT claim to a user-controlled struct, always clear the field unconditionally, not conditionally. This pattern should be a standard in any JWT→struct mapping: `struct.field = claims.field ?? undefined`.

## Doc Links

- Spec: `docs/features/F009-l2-token-extension.md`
- Test: `packages/mesh-hub/src/l2-token-extension.test.ts`
- Fix commit: `fbebfa5` (P0 body injection fix)
- Merge commit: `9de1eb9` (PR #14)

## Rule Update Target

Consider adding to `docs/SOP.md` or shared rules:
> JWT→struct mapping: always unconditionally assign `body.field = claims.field ?? undefined`, never use `if (claims.field) { body.field = claims.field }`. The conditional form leaves prior values intact when the claim is absent.
