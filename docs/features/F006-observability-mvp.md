---
feature_ids: [F006]
related_features: []
topics: [hub, logging, metrics, observability]
doc_kind: spec
created: 2026-04-01
---

# F006: Observability MVP

> Status: spec | Owner: TBD

## Why

Hub currently uses plain console.log with no structure, no log levels, no request correlation. When debugging cross-node issues, there's no way to trace a request through the system. Operators need structured logs and basic metrics to monitor a running Hub.

## What

1. **Structured logging**: Replace console.log with pino (Fastify ecosystem standard); JSON format with traceId correlation across request lifecycle
2. **Log levels**: configurable via `LOG_LEVEL` env var (default: info)
3. **`GET /v1/metrics`**: Returns JSON with basic counters:
   - Total invocations (success/error/timeout)
   - Latency percentiles (p50, p95, p99)
   - Connected nodes count
   - Uptime
4. **Request logging**: Every endpoint logs request/response with traceId, duration, status

## Acceptance Criteria

- [ ] AC-1: All Hub log output is structured JSON with timestamp, level, traceId
- [ ] AC-2: LOG_LEVEL env var controls verbosity
- [ ] AC-3: `GET /v1/metrics` returns invocation counts, error counts, latency percentiles
- [ ] AC-4: Cross-request correlation: same traceId appears in TOKEN, INVOKE, and AUDIT logs
- [ ] AC-5: pino integrated with Fastify (not a separate logger)

## Dependencies

None (independent)

## Risk

- pino adds a dependency (but it's Fastify's default logger, minimal risk)
- Metrics endpoint without auth could leak operational info (add adminToken guard?)

## Open Questions

- Prometheus format vs JSON for /v1/metrics?
- Should metrics be opt-in (separate config flag)?
