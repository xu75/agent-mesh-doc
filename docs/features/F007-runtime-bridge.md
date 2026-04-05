---
feature_ids: [F007]
related_features: [F002, F003]
topics: [adapter, runtime, clowder, openclaw]
doc_kind: spec
created: 2026-04-01
layer: protocol
owner_module: mesh-bridge
status: won't-do
phase: 1
depends_on:
  - { id: F002, type: blocking }
  - { id: F003, type: blocking }
evidence: []
---

# F007: Runtime Bridge

> Status: spec | Owner: TBD

## Why

F002 built protocol-level adapters (Clowder/OpenClaw mapping), but there's no bridge connecting Hub INVOKE to actual local runtime execution. When Hub relays an invocation to a node, the node needs to dispatch it to the correct local agent (cat) via the runtime's native API. Without this, the mesh is a protocol demo, not a working system.

## What

1. **Runtime adapter interface**: `MeshRuntime` abstract with `listCapabilities()` and `invokeLocal(catId, task, context, scope)` methods
2. **`createClowderRuntime()`**: Bridges INVOKE payload to Clowder's native agent invocation
3. **`createOpenClawRuntime()`**: Bridges INVOKE payload to OpenClaw's instruction-based API
4. **MeshServer integration**: Server uses runtime adapter to dispatch invocations instead of raw handler

## Acceptance Criteria

- [ ] AC-1: `createClowderRuntime()` maps INVOKE payload to Clowder agent call and returns result
- [ ] AC-2: `createOpenClawRuntime()` maps INVOKE payload to OpenClaw instruction call
- [ ] AC-3: MeshServer can be configured with a runtime adapter instead of raw handler
- [ ] AC-4: Capabilities from runtime adapter auto-register with Hub on connect

## Dependencies

F003 (multi-capability model for registration)

## Risk

- Clowder/OpenClaw APIs may not be stable
- Tight coupling to specific runtime versions

## Open Questions

- Should runtime bridge be in mesh-adapter or mesh-node package?
- gpt52's 3-layer SDK design: when to adopt?
