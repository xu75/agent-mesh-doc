---
feature_ids: []
topics: [process, feat-doc, design-gate, review, consistency]
doc_kind: proposal
created: 2026-04-01
---

# EP: Add Final Doc Consistency Pass (Without Mandatory Doc Reviewer Gate)

> Status: approved (2026-04-01)  
> Owner: 宪宪  
> Implemented: feat-lifecycle SKILL.md Step 0.8

## Boundary (What This Proposal Is / Is Not)

This proposal clarifies one boundary first:

- We are **not** in a "single cat writes doc and sends directly" model.
- We are also **not** in a "every feat doc must pass a mandatory independent doc reviewer gate" model.

Current reality is in the middle:
- discussion + design gate exist and are mandatory before execution;
- but after discussion is converged into a feat doc, there is no globally consistent final doc QA checkpoint across all features.

So the gap is **not lack of discussion**.  
The gap is **lack of a standardized post-convergence final doc consistency pass**.

## Trigger

Need this proposal when any of the following becomes frequent:

1. Feat docs are structurally valid, but CVO still needs to patch AC boundaries/risk coverage.
2. Feature status changes in `docs/features/` are not synchronized across `README` / `BACKLOG` / runbook/decision docs.
3. Doc quality depends on whether the author *remembered* to request another cat for consistency checking.

## Evidence

### E1) Pre-authoring discussion/design governance already exists

`feat-lifecycle` explicitly requires:
- discussion as a required stage (`.agents/skills/feat-lifecycle/SKILL.md:91`)
- design gate between discussion and planning/execution (`.agents/skills/feat-lifecycle/SKILL.md:104`)
- backend/architecture paths require cat discussion before moving forward (`.agents/skills/feat-lifecycle/SKILL.md:113`, `.agents/skills/feat-lifecycle/SKILL.md:114`)

Interpretation: docs are generally **discussion-converged artifacts**, not closed-door drafts.

### E2) Completion currently checks delivery truth, not final doc consistency gate

`feat-lifecycle` completion focuses on:
- delivery truth verification via git/pr (`.agents/skills/feat-lifecycle/SKILL.md:150`)
- status updates and truth-source syncing (`.agents/skills/feat-lifecycle/SKILL.md:205` to `.agents/skills/feat-lifecycle/SKILL.md:213`)

But there is no explicit, globally enforced step:  
"after feat doc is authored, another cat performs a final doc consistency pass before handoff."

### E3) No repository-level docs QA gate in current scripts/workflows

- Root scripts include `build/check/lint/test` only (`package.json`)
- docs sync workflow mirrors docs to public repo after push, but does not perform a doc-consistency review gate (`.github/workflows/sync-docs-public.yml`)

Interpretation: we have formatting/tooling checks and sync automation, but no standardized consistency gate.

## Root Cause

1. Governance is strong in "pre-implementation direction control" (discussion/design gate), but weak in "post-convergence doc consistency control."
2. "Doc quality assurance" is currently person-driven and optional, not trigger-driven and standardized.
3. Existing checks optimize for syntax/format and source-of-truth updates, not for semantic consistency across documents.

## Lever (Minimal Change, Not Heavy Process)

Do **not** add a mandatory independent reviewer gate for every feat doc.

Add a lightweight `final doc consistency pass` with trigger-based activation:

### Activation triggers

Run this pass only when one of these happens:

1. `docs/features/F*.md` status changes (e.g., `spec -> in-progress/complete/won't-do`)
2. Acceptance Criteria or phase boundaries are materially changed
3. Cross-doc truth sources are impacted (`README`, `BACKLOG`, runbook, decisions, workflow status docs)

### Pass checklist (5 checks only)

1. Discussion fidelity: does the doc reflect converged discussion decisions?
2. AC verifiability: are ACs testable/observable, not descriptive only?
3. Phase boundary clarity: are "now vs later" edges explicit?
4. Risk/Open Questions completeness: any known risk dropped during doc convergence?
5. Cross-doc sync: status and key claims consistent across truth-source docs.

### Output contract (minimal)

- `Approved`
- `Request Changes`

No long-form narrative required; short check result is enough.

### Ownership/timebox

- Reviewer should be non-author cat (cross-family preferred).
- Timebox target: 10-15 minutes per triggered pass.

## Verify

Pilot window: next 2 weeks (or next 5 feature-doc transitions, whichever comes first).

Success criteria:

1. Zero CVO-reported "missing AC boundary / missing risk / cross-doc mismatch" on pilot items.
2. Median pass time <= 15 minutes.
3. No noticeable slowdown to normal feature throughput.

Rollback criteria:

1. Median pass time > 20 minutes for two consecutive pilot items.
2. No measurable reduction in doc-level inconsistency issues.

## Decision

CVO approved on 2026-04-01. Implemented as `feat-lifecycle` Step 0.8 (lightweight patch, not a new gate).

