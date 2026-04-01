# Agent Mesh

Agent Mesh is an open interoperability platform for AI agents.

It is not bound to any single runtime (including Clowder-AI). Clowder-AI can be one integrator, but the mesh is designed as a public platform that any agent system can join.

## Positioning

- Runtime-agnostic: connect different agent stacks through one security and protocol baseline.
- Security-first: three-layer identity (L0/L1/L2), mTLS, replay protection.
- Evolvable topology: start Hub-and-Spoke, then move toward federation.

## Integration Model

Any agent runtime can join via either path:

| Path | Description | Current Stage |
|------|-------------|---------------|
| Plugin Adapter | Implement a plugin/adapter that maps local runtime semantics to Agent Mesh protocol and identity flow. | In progress (`@agent-mesh/adapter`: Clowder/OpenClaw) |
| Native Protocol | Implement the open protocol directly (HELLO/CAPS/INVOKE/STREAM/RESULT/ERROR + identity token model). | In progress |

## Quick Start

Get from clone to your first cross-node invocation in 3 steps:

```bash
# 1. Install dependencies
pnpm install

# 2. Build all packages
pnpm build

# 3. Run the hello-world example
node examples/hello-world/dist/run.js
```

You'll see two nodes register with Hub, discover each other's capabilities, and perform a remote invocation — all with full identity verification (L0/L1/L2).

For a bidirectional chat example:

```bash
node examples/two-node-chat/dist/run.js
```

## Status (2026-04-01)

**Phase: Wave 1 complete, Wave 2 in progress**

| Wave | Features | Status |
|------|----------|--------|
| Wave 1 | F001 Security, F002 Adapter, F003 Capabilities, F004 Liveness, F006 Observability | done |
| Wave 2 | F005 Developer Experience | in-progress |
| Wave 3 | F007 Runtime Bridge, F008 Lifecycle, F009 L2 Extension | spec |

## Repository Layout

```text
packages/
  mesh-adapter/   Runtime plugin adapter (Clowder/OpenClaw -> Mesh semantics)
  mesh-hub/       Hub service (identity, routing, liveness, metrics)
  mesh-node/      Node SDK (MeshClient + MeshServer)
  mesh-protocol/  Shared protocol and identity definitions
examples/
  hello-world/    Minimal echo invocation (5-minute onboarding)
  two-node-chat/  Bidirectional invocation between two nodes
docs/
  features/       Feature specs and acceptance criteria
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

## Scope (Current MVP Stage)

In scope now:
- W1-W2 security and identity foundation (E1/E2 gate).

Out of scope now:
- Trading/token economics.
- Large-scale open admission.

## 12-Week Blueprint

| Period | Goal | Exit Criteria |
|--------|------|---------------|
| W1-W2 | Security & Identity foundation | E1/E2 pass; handshake >= 99.5%; replay block 100% |
| W3-W4 | Protocol & first cross-node call | First e2e success; 500 schema drift = 0 |
| W5-W6 | P0 guardrails & audit replay | Token revoke <= 5s; node revoke <= 60s; trace 100% |
| W7-W8 | Routing & governance integration | E5/E6/E7 pass; remote @mention >= 95% |
| W9-W10 | Small-scale external pilot | 14 days zero P0; cross-node >= 95% |
| W11-W12 | Beta convergence & Go/No-Go | 4 acceptance scenarios pass |

Hard stop at W6: if any metric fails, fall back to single-node enhancement.

## Source of Truth

- [README](README.md) for project entry.
- [20-round debate](docs/debate-20r.md) for architecture reasoning.
- [F001 spec](docs/features/F001-security-identity-foundation.md) for current implementation contract and AC.
- [F002 spec](docs/features/F002-plugin-adapter-dual-stack.md) for dual-stack adapter contract.
- [BACKLOG](BACKLOG.md) for active feature tracking.

## Related

- [Clowder-AI](https://github.com/xu75/Clowder-AI): one potential runtime integration target, not a hard dependency of Agent Mesh.
