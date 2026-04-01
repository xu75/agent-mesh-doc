---
feature_ids: [F006]
related_features: []
topics: [hub, logging, metrics, observability]
doc_kind: spec
created: 2026-04-01
---

# F006: Observability MVP

> Status: complete | Owner: 宪宪

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

- [x] AC-1: All Hub log output is structured JSON with timestamp, level, traceId
- [x] AC-2: LOG_LEVEL env var controls verbosity
- [x] AC-3: `GET /v1/metrics` returns invocation counts, error counts, latency percentiles
- [x] AC-4: Cross-request correlation: same traceId appears in TOKEN, INVOKE, and AUDIT logs
- [x] AC-5: pino integrated with Fastify (not a separate logger)

## Timeline

| Date | Event |
|------|-------|
| 2026-04-01 | F006 merged to main (`91888c5`) — structured logs + metrics + trace correlation |

## Evidence

- Code path: `MetricsCollector`, `/v1/metrics`, TOKEN/INVOKE trace logging in `packages/mesh-hub/src/hub.ts`
- Test suite: `packages/mesh-hub/src/observability.test.ts` (6 tests)
- Verified on 2026-04-01: `pnpm --filter @agent-mesh/hub exec node --test dist/observability.test.js` → pass

## Deferred

1. Metrics auth策略（是否对 `/v1/metrics` 增加保护）
2. Prometheus/OpenTelemetry 导出格式（当前仅 JSON）
