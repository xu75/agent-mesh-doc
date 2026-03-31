---
feature_ids: [F004]
related_features: [F003]
topics: [hub, liveness, heartbeat]
doc_kind: spec
created: 2026-04-01
---

# F004: Node Liveness

> Status: spec | Owner: TBD

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

- [ ] AC-1: Node sends heartbeat, Hub records last-seen timestamp
- [ ] AC-2: CAPS returns accurate status (online/stale/offline)
- [ ] AC-3: Node that stops heartbeating transitions to stale then offline
- [ ] AC-4: INVOKE to offline node returns NODE_OFFLINE (not 10s timeout)
- [ ] AC-5: Heartbeat interval and thresholds are configurable
- [ ] AC-6: Heartbeat requires valid L1 certificate (no unauthenticated keep-alive)

## Dependencies

None (can parallel with F003)

## Risk

- Heartbeat interval too aggressive = unnecessary traffic; too lax = stale detection lag
- Timer cleanup on Hub shutdown

## Open Questions

- Should heartbeat carry capability updates (dynamic re-registration)?
