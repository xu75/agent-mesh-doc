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
| Plugin Adapter | Implement a plugin/adapter that maps local runtime semantics to Agent Mesh protocol and identity flow. | Planned |
| Native Protocol | Implement the open protocol directly (HELLO/CAPS/INVOKE/STREAM/RESULT/ERROR + identity token model). | In progress |

## Status (2026-03-30)

**Phase: W1 in progress** (`F001: Security & Identity Foundation`)

Completed in repository:
- Monorepo scaffold and TypeScript workspace layout.
- `@agent-mesh/protocol` type definitions for protocol messages, error codes, and identity claims.
- `@agent-mesh/hub` skeleton with `/health`, `/v1/hello`, `/v1/caps`, `/v1/token` endpoints (stubbed for W1/W2 tasks).

In progress:
- L0 key generation + persistence.
- L1 Mesh Certificate issuance/verification.
- L2 Invocation Token issuance/verification.
- jti replay detection and mTLS test coverage (E1/E2).

## Repository Layout

```text
packages/
  mesh-hub/       Hub service (registration, issuance, revocation)
  mesh-node/      Node SDK (placeholder in current phase)
  mesh-protocol/  Shared protocol and identity definitions
docs/
  debate-20r.md   20-round architecture debate record
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
- [BACKLOG](BACKLOG.md) for active feature tracking.

## Related

- [Clowder-AI](https://github.com/xu75/Clowder-AI): one potential runtime integration target, not a hard dependency of Agent Mesh.
