---
feature_ids: [F005]
related_features: [F003, F004]
topics: [sdk, examples, docs, dx]
doc_kind: spec
created: 2026-04-01
---

# F005: Developer Experience Pack

> Status: spec | Owner: TBD

## Why

MVP "frontend" = "developer clones repo, runs examples, sees cross-node invocation in < 5 minutes". Currently there are no runnable examples, no quick start guide, and the SDK requires understanding low-level HELLO/TOKEN/INVOKE flow. This blocks adoption and demo-ability.

## What

1. **`examples/hello-world/`**: Minimal two-file example (echo-node.ts + caller.ts) + README
2. **`examples/two-node-chat/`**: A and B invoke each other, showing bidirectional flow
3. **Hub startup banner**: Clear console output showing hub status, connected nodes, events
4. **SDK convenience wrapper** (optional): Thin `MeshNode` class wrapping MeshClient + MeshServer lifecycle for simpler onboarding. Existing low-level API remains available.
5. **Quick start README section**: 3-step instructions in project root README

## Acceptance Criteria

- [ ] AC-1: `examples/hello-world/` runs with `pnpm build && node examples/hello-world/caller.js` and produces successful invocation output
- [ ] AC-2: `examples/two-node-chat/` demonstrates bidirectional invocation
- [ ] AC-3: Hub startup prints clear banner with config summary
- [ ] AC-4: Hub logs each HELLO/INVOKE event in structured format
- [ ] AC-5: README quick start covers: install, start hub, run example
- [ ] AC-6: Total setup time from clone to first successful invocation < 5 minutes

## Dependencies

F003 (multi-capability model), F004 (liveness — examples should show real status)

## Risk

- Examples rot if not tested in CI
- Port conflicts in examples (use port 0 or clearly documented ports)

## Open Questions

- Docker-compose for examples or pure Node.js scripts?
