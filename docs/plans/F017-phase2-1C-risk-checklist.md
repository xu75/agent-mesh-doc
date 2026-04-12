---
feature_ids: ["F017"]
topics: ["phase-1c", "fingerprint-normalization", "e2e", "risk-checklist"]
doc_kind: "plan"
created: "2026-04-12"
---

# F017 Phase 2 — Phase 1C Risk Checklist (Round 1)

## Goal

Scope: AC-P2-12 ~ AC-P2-15 in `docs/features/F017-phase2-oauth-proxy.md` §6.3.

- AC-P2-12: outbound `User-Agent` / `x-stainless-*` matches owner environment
- AC-P2-13: outbound body `<env>` block reflects owner environment (not caller)
- AC-P2-14: non-Claude-Code caller requests remain unmodified
- AC-P2-15: Claude Code E2E (Read + Write + Bash) succeeds through pool

## Current Baseline

- Phase 1B is merged and green on main.
- `plan-bridge-oauth/src/proxy.ts` already sets `user-agent`, but no `x-stainless-*` normalization yet.
- Feature doc references `plan-bridge-oauth/src/env-normalizer.ts`, but file does not exist yet.
- Existing tests cover proxy pass-through, billing header injection, routing hard-fail, and session semantics.

## Risk Matrix

| ID | Risk | Impact | Likelihood | AC | First Mitigation |
|---|---|---|---|---|---|
| R1 | Owner fingerprint source is undefined/unstable, causing drift from true owner CLI headers | High | Medium | P2-12 | Define single source-of-truth for owner fingerprint snapshot and fallback behavior before implementation |
| R2 | Over-normalization mutates non-Claude callers, violating transparency | High | Medium | P2-14 | Add explicit caller classification guard and negative tests first |
| R3 | `<env>` rewrite parser is brittle (multi-turn/system-array/tool blocks), corrupting prompt body | High | Medium | P2-13 | Implement parser/rewriter with strict pattern boundary + snapshot tests |
| R4 | Header/body normalization diverges between non-streaming and streaming code paths | Medium | Medium | P2-12/13 | Centralize normalization function and invoke from both proxy handlers |
| R5 | E2E validates only text path, misses tool chain and filesystem side-effects | High | Medium | P2-15 | Build scripted E2E with deterministic read/write/bash assertions |
| R6 | Token/billing injection ordering conflicts with env normalization | Medium | Low | P2-12/13 | Freeze transformation pipeline order and test each stage output |

## AC-by-AC Verification Plan

### AC-P2-12 (Header fingerprint)

Required checks:

1. Outbound `user-agent` equals owner snapshot value.
2. Outbound `x-stainless-*` set is complete and owner-matching.
3. Caller-supplied conflicting fingerprint headers are ignored/replaced.
4. Non-streaming and streaming paths emit identical fingerprint behavior.

Suggested red tests first:

- `proxy.test.ts`: owner fingerprint override case.
- `streaming-proxy.test.ts`: same assertions for stream path.

### AC-P2-13 (`<env>` normalization)

Required checks:

1. Claude Code request with `<env>` gets rewritten to owner environment.
2. Rewriter is idempotent (double pass does not stack or damage content).
3. System string and system block array both handled.
4. Tool payload and non-env blocks remain byte-stable.

Suggested red tests first:

- New `env-normalizer.test.ts` with fixture snapshots.
- Integration tests through proxy handler to verify final outbound body.

### AC-P2-14 (No modification for non-Claude caller)

Required checks:

1. Requests without Claude `<env>` marker remain unchanged.
2. OpenAI-format or generic Anthropic client payloads are pass-through.
3. No accidental header rewrite for unknown clients.

Suggested red tests first:

- Negative fixtures for plain Anthropic/OpenAI callers.
- Assertion on exact outbound body equality.

### AC-P2-15 (E2E workflow)

Required checks:

1. Read: model can read repo file through pool route.
2. Write: model can create/modify file and persist expected content.
3. Bash tool: command execution works end-to-end.
4. Multi-turn session remains sticky to the same bridge.
5. Usage/ledger records are settled and non-error.

Suggested acceptance harness:

- Scripted E2E runner under `packages/plan-pool/src/integration.test.ts` (or dedicated `e2e` test).
- Use deterministic workspace fixture + cleanup.

## Implementation Guardrails

1. Normalize in one place, call from both non-stream and stream handlers.
2. Keep transformation pipeline explicit: classify caller -> normalize headers -> normalize body -> inject billing -> dispatch.
3. Add an opt-out/feature flag for quick rollback if fingerprint logic regresses production traffic.
4. Log normalization decisions at debug level (without leaking token or sensitive env values).

## Decision Freeze (2026-04-12)

1. Owner fingerprint source (CVO confirmed):
   - First install of bridge collects owner machine environment snapshot and writes runtime config.
   - Users can override manually via config.
   - Code keeps only template/defaults; no hardcoded machine-specific fingerprint in source.
2. `<env>` / header target template (team consensus, aligned with cc-gateway + Stainless conventions):
   - Canonical fingerprint fields:
     - `user-agent`
     - `anthropic-version`
     - `x-stainless-lang`
     - `x-stainless-package-version`
     - `x-stainless-os`
     - `x-stainless-arch`
     - `x-stainless-runtime`
     - `x-stainless-runtime-version`
   - Tiering:
     - T1 required: `user-agent`, `anthropic-version`, `x-stainless-os`, `x-stainless-arch`
     - T2 recommended: remaining `x-stainless-*` fields
3. Missing fingerprint policy (team consensus):
   - Enforcement modes: `strict | warn | off` (per bridge configurable).
   - Default progression: `bootstrap-warn` (initial capture window) -> `strict` (steady state).
   - `strict` behavior: T1 missing => fail-closed (`503 bridge_fingerprint_unavailable`), T2 missing => fallback to config defaults + warning.
   - `warn` behavior: T1/T2 missing => fallback + warning (no immediate block), time-boxed for calibration only.
   - No baked-in fingerprint defaults in runtime dispatch path; only captured config or `last-known-good`.
4. E2E environment (CVO confirmed):
   - E2E runs locally in this repo/runtime (not CI-first).
   - Required flow: Read + Write + Bash + multi-turn sticky session evidence.

## Adopt / Avoid Checklist (execution contract)

Adopt:

1. `env` + `prompt_env` dual-profile model.
2. `<env>` whole-block rewrite only.
3. `bootstrap-warn -> strict` progression with strict steady-state.
4. Runtime source restricted to captured profile + `last-known-good`.
5. Failure-budget suspension (`SUSPENDED_FINGERPRINT`) before repeated upstream attempts.

Avoid:

1. Long-running `warn` mode.
2. Hardcoded generic fingerprint defaults in live dispatch path.
3. Unauthenticated fallback forwarding.
4. Letting fallback assets become long-term primary source.

## Closed Questions

- Q1 owner fingerprint source: closed.
- Q2 canonical template: closed.
- Q3 fail-closed vs fallback policy: closed.
- Q4 E2E runtime location: closed.

## New Risk Controls (CVO Follow-up 2026-04-12)

### A. Failure Burst Protection (anti-account-risk)

Goal: avoid repeated suspicious outbound attempts when fingerprint validation keeps failing.

1. Preflight first: validate fingerprint completeness locally before any upstream call.
2. If preflight fails:
   - Return `503 bridge_fingerprint_unavailable` immediately.
   - Do not send traffic to Anthropic for that request.
3. Circuit breaker:
   - Track rolling failure count per bridge.
   - If failures exceed threshold (example: 5 in 10 minutes), mark bridge fingerprint state as `SUSPENDED_FINGERPRINT`.
   - In suspended state, pool should stop routing new external sessions to this bridge and require refresh/recovery.
4. Recovery:
   - Successful refresh + one successful probe request clears suspended state.
   - Keep `last-known-good` snapshot for emergency rollback.

### B. Fingerprint Drift Auto-Refresh

Goal: keep owner fingerprint aligned with real local runtime changes (OS/Node/SDK).

1. Refresh triggers:
   - bridge startup
   - periodic timer (example: every 6h)
   - detected runtime/version drift
2. Refresh flow:
   - recapture local fingerprint
   - schema validate
   - stage as candidate profile
   - run lightweight probe
   - promote to active profile only on probe success
3. Safety:
   - store previous profile as `last-known-good`
   - probe failure reverts to `last-known-good`
   - emit structured events for every promote/revert

### C. Existing State Machine Integration

1. Reuse `authHealthy`/unbind path in `plan-pool`:
   - persistent fingerprint failure should mark bridge unavailable to routing logic.
2. Non-Claude callers remain passthrough and are excluded from fingerprint enforcement.
3. Expose observability fields in heartbeat:
   - `fingerprintConfigPresent`
   - `fingerprintPolicy`
   - `fingerprintState` (`ok` | `degraded` | `suspended`)

## Exit Gate for Phase 1C

All must pass:

1. `pnpm --filter @agent-mesh/plan-bridge-oauth test`
2. `pnpm --filter @agent-mesh/plan-pool test`
3. `pnpm --filter @agent-mesh/plan-bridge-oauth lint`
4. `pnpm --filter @agent-mesh/plan-pool lint`
5. Explicit evidence artifact for AC-P2-15 (logs or snapshot report)
