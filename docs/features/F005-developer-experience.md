---
feature_ids: [F005]
related_features: [F003, F004]
topics: [sdk, examples, docs, dx]
doc_kind: spec
created: 2026-04-01
layer: protocol
owner_module: mesh-node
status: complete
phase: 1
depends_on:
  - { id: F003, type: blocking }
  - { id: F004, type: blocking }
evidence: [examples/hello-world, examples/two-node-chat]
---

# F005: Developer Experience Pack

> Status: complete | Owner: 宪宪

## Why

MVP "frontend" = "developer clones repo, runs examples, sees cross-node invocation in < 5 minutes". Currently there are no runnable examples, no quick start guide, and the SDK requires understanding low-level HELLO/TOKEN/INVOKE flow. This blocks adoption and demo-ability.

## What

1. **`examples/hello-world/`**: Minimal two-file example (echo-node.ts + caller.ts) + README
2. **`examples/two-node-chat/`**: A and B invoke each other, showing bidirectional flow
3. **Hub startup banner**: Clear console output showing hub status, connected nodes, events
4. **SDK convenience wrapper** (optional): Thin `MeshNode` class wrapping MeshClient + MeshServer lifecycle for simpler onboarding. Existing low-level API remains available.
5. **Quick start README section**: 3-step instructions in project root README

## Acceptance Criteria

- [x] AC-1: `examples/hello-world/` runs with `pnpm build && node examples/hello-world/dist/run.js` and produces successful invocation output
- [x] AC-2: `examples/two-node-chat/` demonstrates bidirectional invocation
- [x] AC-3: Hub startup prints clear banner with config summary
- [x] AC-4: Hub logs each HELLO/INVOKE event in structured format
- [x] AC-5: README quick start covers: install, start hub, run example
- [x] AC-6: Total setup time from clone to first successful invocation < 5 minutes（本地演示路径）

## Timeline

| Date | Event |
|------|-------|
| 2026-04-01 | F005 merged to main (`21665d5`) — examples + startup banner + quick start |

## Evidence

- Example entrypoints:
  - `examples/hello-world/src/run.ts`
  - `examples/two-node-chat/src/run.ts`
- Hub startup banner: `packages/mesh-hub/src/index.ts`
- Verified on 2026-04-01:
  - `node examples/hello-world/dist/run.js` → pass
  - `node examples/two-node-chat/dist/run.js` → pass

## Deferred

1. Example CI matrix（防止示例老化）
2. Dockerized demo profile（便于外部演示环境复制）
