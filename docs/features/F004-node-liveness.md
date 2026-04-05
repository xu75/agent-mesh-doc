---
feature_ids: [F004]
related_features: [F003]
topics: [hub, liveness, heartbeat]
doc_kind: spec
created: 2026-04-01
layer: governance
owner_module: mesh-hub
status: complete
phase: 1
depends_on:
  - { id: F003, type: blocking }
evidence: [node-liveness.test.ts]
---

# F004: Node Liveness

> Status: complete | Owner: 宪宪

## Why

CAPS currently reports all allowlisted nodes as "online" regardless of actual state (hub.ts L137). This is false data. When a node goes down, Hub doesn't know — INVOKE blindly fires and waits for timeout (default 10s). Users need accurate node status for reliable capability discovery.

## What

1. **Heartbeat endpoint**: `POST /v1/heartbeat` — node sends periodic heartbeat with `{nodeId, token}`
2. **Status tracking**: Hub maintains per-node last-seen timestamp; derives status:
   - `online`: heartbeat within 1x interval
   - `stale`: heartbeat within 2x interval
   - `offline`: no heartbeat beyond 2x interval
3. **CAPS integration**: Node status included in CAPS response; can filter by `{status: "online"}`
4. **Default config**: heartbeat interval 15s, stale threshold 30s (configurable)

## Acceptance Criteria

- [x] AC-1: Node sends heartbeat, Hub records last-seen timestamp
- [x] AC-2: CAPS returns accurate status (online/stale/offline)
- [x] AC-3: Node that stops heartbeating transitions to stale then offline
- [x] AC-4: INVOKE to offline node returns NODE_OFFLINE (not 10s timeout)
- [x] AC-5: Heartbeat interval and thresholds are configurable
- [x] AC-6: Heartbeat requires valid L1 certificate (no unauthenticated keep-alive)

## Timeline

| Date | Event |
|------|-------|
| 2026-04-01 | F004 merged to main (`e02f236`) — heartbeat + status derivation + offline invoke guard |

## Evidence

- Code path: `/v1/heartbeat`, `deriveNodeStatus()`, `/v1/invoke` offline guard in `packages/mesh-hub/src/hub.ts`
- Test suite: `packages/mesh-hub/src/node-liveness.test.ts` (10 tests)
- Verified on 2026-04-01: `pnpm --filter @agent-mesh/hub exec node --test dist/node-liveness.test.js` → pass

## Deferred

1. Capability update through heartbeat payload（目前仍通过 adapter register 管理）
