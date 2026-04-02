---
feature_ids: []
topics: [repo-layout, packages, bridges, experiments, governance]
doc_kind: decision
created: 2026-04-02
---

# Repo Layout Governance (2026-04-02)

## Decision

Adopt a three-tier directory boundary for bridge-related evolution:

1. **Publishable artifacts** go to `packages/`
2. **Deployment/runtime wrappers** go to `bridges/` (or `services/bridges/`)
3. **Exploratory prototypes** stay in `experiments/`

Current MVP baseline remains stable:
- no package rename
- no immediate directory migration
- no workspace topology break in this step

## Why

1. `@agent-mesh/bridge` is already promoted as a formal package and consumed by experiments/docs.
2. Forcing a directory move now creates avoidable churn in lockfile, docs path references, and onboarding scripts.
3. The project needs a governance rule first, then incremental migration by feature maturity.

## Classification Rules

### `packages/`

Use when all are true:
- reusable by multiple integrations
- versioned/semver-managed
- expected to be part of released surface

Examples:
- `@agent-mesh/protocol`
- `@agent-mesh/node`
- `@agent-mesh/hub`
- `@agent-mesh/adapter`
- `@agent-mesh/bridge`

### `bridges/` (future target)

Use for runtime-specific wrappers that are not inherently npm package surfaces:
- Python/Swift service wrappers
- process supervisors
- deployment manifests and host-specific glue

`bridges/` may depend on `packages/*`, but should not define core protocol contracts.

### `experiments/`

Use for:
- throwaway/validation work
- pre-promotion prototypes

Promotion path:
`experiments/*` -> mature -> `packages/*` or `bridges/*`

## Migration Strategy

1. **Now**: enforce directory policy and local-ignore hygiene only.
2. **Later**: promote new bridge capabilities directly into the right tier (avoid back-and-forth moves).
3. **Optional cleanup phase**: migrate legacy experiment paths with compatibility window and doc backfill.

## Guardrails

1. No "big-bang" path migration on active feature branches.
2. Any package rename requires compatibility plan and docs update in the same PR.
3. `experiments/` entries are not release-gate truth sources unless explicitly promoted.

## Effective Scope

This decision applies to all upcoming bridge-like components (TTS, ASR, camera, GPU, etc.) and future repo layout discussions.

