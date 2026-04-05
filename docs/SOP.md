---
topics: [sop, workflow]
doc_kind: note
created: 2026-03-30
---

# Standard Operating Procedure

## Pre-Work: Architecture Check

Before starting any feature work, read [`docs/ARCHITECTURE.md`](ARCHITECTURE.md):
- Identify which layer(s) your work touches
- Verify your dependencies against the whitelist matrix
- Check if your feature is Phase 1 or Phase 2 scope

## Workflow (6 steps)

| Step | What | Skill |
|------|------|-------|
| 1 | Create worktree | `worktree` |
| 2 | Self-check (spec compliance) | `quality-gate` |
| 3 | Peer review | `request-review` / `receive-review` |
| 4 | Merge gate | `merge-gate` |
| 5 | PR + cloud review | (merge-gate handles) |
| 6 | Merge + cleanup | (SOP steps) |

## Change Admission (PR Checklist)

Before submitting a PR, verify:

1. [ ] Which layers does this change touch? (check dependency whitelist)
2. [ ] Does it add a new cross-layer dependency? → requires ADR
3. [ ] Does it change communication topology / registration / routing / identity? → requires ADR
4. [ ] Does a feature spec modify Phase boundaries? → forbidden, must amend ADR
5. [ ] Does it affect an architecture invariant? → explicit justification needed
6. [ ] Feature frontmatter status updated? (frontmatter is the single source of truth)

## Code Quality

- Biome: `pnpm check` / `pnpm check:fix`
- Types: `pnpm lint`
- File limits: 200 lines warn / 350 hard cap
