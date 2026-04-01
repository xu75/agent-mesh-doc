# Agent Mesh

Agent Mesh is an open interoperability platform for AI agents.

It is runtime-agnostic: Clowder-AI is one integrator, not a hard dependency.  
The mesh exposes one shared security and protocol baseline for heterogeneous agent runtimes.

## Positioning

- Runtime-agnostic interoperability
- Security-first execution path (L0/L1/L2 identity + revocation + replay guard + mTLS)
- Evolvable topology (Hub-and-Spoke start, federation target)

## Current MVP Status (2026-04-01)

MVP foundation is implemented and runnable.

| Feature | Status | Notes |
|---------|--------|-------|
| F001 Security & Identity | complete | L0/L1/L2 + mTLS + revoke + replay guard |
| F002 Plugin Adapter | complete | Clowder/OpenClaw adapter + Hub registration endpoint |
| F003 Capability Model | complete | multi-capability + CAPS filtering + adapter alignment |
| F004 Node Liveness | complete | heartbeat + online/stale/offline + offline invoke guard |
| F005 Developer Experience | complete | hello-world + two-node-chat + startup banner + quick start |
| F006 Observability | complete | structured logs + metrics + trace correlation |
| F007 Runtime Bridge | spec | next-phase runtime dispatch bridge |
| F008 Graceful Lifecycle | spec | next-phase reliability lifecycle |
| F009 L2 Token Extension | spec | next-phase token model extension |

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
| Native Protocol | Implement protocol directly (HELLO/CAPS/INVOKE/RESULT/ERROR + identity model) | available |

## Repository Layout

```text
packages/
  mesh-adapter/   Runtime plugin adapter (Clowder/OpenClaw -> Mesh semantics)
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
- [F001 spec](docs/features/F001-security-identity-foundation.md)
- [F002 spec](docs/features/F002-plugin-adapter-dual-stack.md)

## Related

- [Clowder-AI](https://github.com/xu75/Clowder-AI)
