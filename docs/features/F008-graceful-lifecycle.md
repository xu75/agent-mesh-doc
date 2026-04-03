---
feature_ids: [F008]
related_features: []
topics: [hub, lifecycle, config, shutdown]
doc_kind: spec
created: 2026-04-01
---

# F008: Graceful Lifecycle

> Status: complete | Owner: 宪宪

## Why

Hub currently has no config validation on startup (invalid config silently fails) and no graceful shutdown (in-flight requests dropped on SIGTERM). For any real deployment, these are basic operational requirements.

## What

1. **Config validation**: Validate HubConfig on startup; reject with clear error messages for missing/invalid fields
2. **Graceful shutdown**: On SIGTERM/SIGINT, stop accepting new connections (503), drain in-flight requests (configurable timeout), then exit
3. **Startup health gate**: `/health` returns `{status: "starting"}` until fully initialized, then `{status: "ok"}`

## Acceptance Criteria

- [x] AC-1: Hub rejects startup with invalid config (missing hubId, missing adminToken, etc.) with actionable error message
- [x] AC-2: On SIGTERM, Hub stops accepting new requests (503) and waits for in-flight to complete
- [x] AC-3: Drain timeout is configurable (default 5s)
- [x] AC-4: `/health` reflects lifecycle state (starting/ok/draining)

## Dependencies

None (independent)

## Risk

Low — standard operational patterns

## Open Questions

- Should node disconnection be signaled during drain?
