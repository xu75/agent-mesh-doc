# Agent Mesh

Agent Mesh is an open interoperability platform for AI agents.

It is runtime-agnostic: Clowder-AI is one integrator, not a hard dependency.  
The mesh exposes one shared security and protocol baseline for heterogeneous agent runtimes.  
Any plugin-capable agent runtime can integrate with low adaptation cost.

## Vision

Build a general interoperability foundation for agents:

- Runtime-neutral by design: Clowder/OpenClaw are reference integrations, not product boundaries.
- Standardize cross-agent collaboration: replace pairwise custom integrations with one shared protocol and identity baseline.
- Governability first: every invocation path is enforceable and auditable (authn, authz, revoke, replay guard, traceability).
- Practical first, then scale: MVP prioritizes secure and reliable collaboration over peak throughput optimization.
- Capability network long-term: local models, internal APIs, and automation workflows can be shared safely across team agents.

## Positioning

- Runtime-agnostic interoperability
- Security-first execution path (L0/L1/L2 identity + revocation + replay guard + mTLS)
- Evolvable topology (Hub-and-Spoke start, federation target)

## MVP Success Criteria

- Two heterogeneous plugin-capable runtimes can integrate with low adaptation cost and complete cross-runtime invoke flows.
- Invocation paths are policy-enforced and evidence-backed (scope, revoke, replay guard, audit trail).
- Team-shared capabilities (local model, internal API, workflow pipeline) can be registered and consumed under scope control.

## Non-Goals (MVP)

- No commitment to decentralized settlement or on-chain reputation.
- No commitment to replacing Hub governance with pure P2P data paths.
- No commitment to throughput-first optimization as MVP acceptance criteria.

## Current MVP Status (2026-04-02)

MVP foundation is implemented and runnable.

| Feature | Status | Notes |
|---------|--------|-------|
| F001 Security & Identity | complete | L0/L1/L2 + mTLS + revoke + replay guard |
| F002 Plugin Adapter | complete | Clowder/OpenClaw adapter + Hub registration endpoint |
| F003 Capability Model | complete | multi-capability + CAPS filtering + adapter alignment |
| F004 Node Liveness | complete | heartbeat + online/stale/offline + offline invoke guard |
| F005 Developer Experience | complete | hello-world + two-node-chat + startup banner + quick start |
| F006 Observability | complete | structured logs + metrics + trace correlation |
| F007 Runtime Bridge | won't-do | superseded by existing `mesh-node` + `mesh-adapter` + `mesh-bridge` composition |
| F008 Graceful Lifecycle | complete | config validation + drain + SIGTERM/SIGINT shutdown |
| F009 L2 Token Extension | complete | optional `targetCatId` claim + cat-level token validation |
| F010 Mesh-Cat Cafe Integration | in-progress | MeshBridge + MCP tools (mesh_discover/mesh_invoke) + sidecar shim |
| F011 Local TTS Bridge Example | spec | first real-capability bridge (Qwen3-TTS) |
| F012 Public Discovery Layer | complete | landing page + /v1/discovery + OpenAPI spec |

## MVP Practical Run

Use the runbook:
- [MVP Practical Runbook](docs/discussions/2026-04-01-mvp-practical-runbook.md)

Fast path:

```bash
pnpm install
pnpm build
node examples/hello-world/dist/run.js
node examples/two-node-chat/dist/run.js
```

## Architecture Decision

- [Mesh Hub MVP Communication Architecture](docs/decisions/2026-04-01-mesh-hub-mvp-communication-architecture.md)
  - MVP: pure Hub Relay
  - Future: Hub governance + optional direct data path
  - Not selected for MVP: pure P2P topology

## Integration Model

Any runtime can join via either path:

| Path | Description | Current Stage |
|------|-------------|---------------|
| Plugin Adapter | Map runtime semantics to Mesh protocol and identity flow | complete for Clowder/OpenClaw (`@agent-mesh/adapter`) |
| Bridge Convenience Layer | Compose adapter + node SDK into one Clowder-first wiring layer | available as `@agent-mesh/bridge` |
| Native Protocol | Implement protocol directly (HELLO/CAPS/INVOKE/RESULT/ERROR + identity model) | available |

## Application Scenarios (MVP)

- Team capability sharing: expose local models, internal APIs, and automation workflows (for example coding-plan pipelines) through Hub registration so other agents in the team can reuse them with scope control.
- Cross-runtime collaboration: let heterogeneous runtimes invoke each other through one protocol and one identity baseline without per-pair custom integration.
- Secure tool relay: centralize authn/authz, replay guard, revocation, and audit trail in Hub so invocation evidence is queryable and governance-ready.
- Incremental rollout: start from Hub Relay in controlled environments, then scale scenarios without replacing protocol or identity foundations.

MVP execution path remains Hub Relay (`Agent A -> Hub -> Agent B`) to preserve synchronous enforcement and full-path auditability.

## Phase 2 Explorations (Not MVP Commitments)

- OpenRouter-like multi-node capability exchange with richer routing and quota policies.
- Optional direct data path (P2P-style) under Hub governance, while keeping Hub as control plane for identity, policy, and revocation.
- Optional decentralized settlement/reputation mechanisms (including blockchain-based designs) after MVP value and threat models are validated in production-like runs.

These are roadmap explorations, not current delivery commitments.

## Repository Layout

```text
packages/
  mesh-adapter/   Runtime plugin adapter (Clowder/OpenClaw -> Mesh semantics)
  mesh-bridge/    Clowder-first wiring layer (adapter + MeshClient + MeshServer)
  mesh-hub/       Hub service (identity, routing, liveness, metrics)
  mesh-node/      Node SDK (MeshClient + MeshServer)
  mesh-protocol/  Shared protocol and identity definitions
examples/
  hello-world/    Minimal echo invocation
  two-node-chat/  Bidirectional invocation demo
docs/
  features/       Feature specs and acceptance criteria
  decisions/      Architecture decisions
  discussions/    MVP runbook and discussion artifacts
```

## Local Development

Prerequisites:
- Node.js >= 20
- pnpm 10

Commands:

```bash
pnpm install
pnpm build
pnpm --filter @agent-mesh/hub start
```

Default Hub listen address:
- `0.0.0.0:3004`

Health check:

```bash
curl http://127.0.0.1:3004/health
```

## Source of Truth

- [README](README.md)
- [BACKLOG](BACKLOG.md)
- [MVP runbook](docs/discussions/2026-04-01-mvp-practical-runbook.md)
- [MVP communication architecture decision](docs/decisions/2026-04-01-mesh-hub-mvp-communication-architecture.md)
- [20-round debate notes](docs/debate-20r.md)
- [Feature specs directory (F001-F012)](docs/features/)

## Related

- [Clowder-AI](https://github.com/xu75/Clowder-AI)
- OpenClaw (integrated via Plugin Adapter path)
