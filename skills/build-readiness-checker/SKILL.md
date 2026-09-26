---
name: build-readiness-checker
description: Audits a set of engineering specs and their task DAG before handoff to BMAD-method execution — checking every spec has a testable success signal, every Constraint is actually cited from an upstream source, every task's risk tier is justified, and the DAG has no unresolved cycles or orphaned Open Questions sitting in auto-approvable tiers. Use whenever the user wants engineering specs or a task DAG "validated," "checked before we start building," "ready for bmad-build," or asks "are we actually ready to build this." This is stage 14 — the last gate in the Build Factory's bridge pipeline before real code gets written.
---

# Build Readiness Checker

## Pipeline contract

Stage 14 of the Build Factory — the last gate before handoff to real execution. Reads every `{slug}-story-<STORY-ID>-engspec.md` in scope and `{slug}-task-dag.md`. Writes `{slug}-build-readiness-review.md` (`doc_type: build_readiness_review`). This mirrors the BA pipeline's own pattern (`requirements-checker`, `story-checker`, `architecture-vs-requirements-checker`) of pairing a dedicated critic with every author stage — you are that critic for `engineering-spec-writer` and `task-dag-builder` combined, since the two outputs need to be checked against each other, not just individually.

**Input contract:** requires the full engspec set for whatever scope `{slug}-task-dag.md` claims to cover, plus the DAG itself. If any engspec the DAG references is missing from what you were given, say so explicitly — don't review a partial set as if it were complete.

**Output contract:** always write to `{slug}-build-readiness-review.md`, with a machine-readable `review_verdict` field. Route findings back to `engineering-spec-writer` (spec-level gaps) or `task-dag-builder` (sequencing/tier gaps) by name, since they're different authors and a vague "fix this" doesn't tell the user which stage to re-run. If `review_verdict` is `build_ready`, tell the user explicitly that the set is cleared for BMAD handoff — note which tasks, if any, are Tier 3 and therefore need a human at every gate from the start.

## What to check

**Success-signal testability.** Every engspec's Success signal must be falsifiable — a metric plus a threshold, not a vague outcome. Flag any spec where the signal reads like a restated capability ("the feature works") rather than something measurable post-release. A signal still marked `(proposed — confirm)` is a gap, not a pass — it means a human hasn't actually committed to the number yet.

**Constraint provenance.** Every line in a spec's Constraints section should cite a `COMP-ID`, `ADR-ID`, or `REQ-NFR-ID`. An uncited constraint is either an invented one (bad) or a real one missing its citation (fixable, but still a gap until fixed) — treat both the same way here: flag it, don't guess which it is.

**Capability-to-AC coverage.** Every capability in a spec should trace back to an acceptance criterion in the source story — spot-check this; a capability with no AC behind it is scope that crept in during translation.

**DAG consistency with the engspec set.** Every `STORY-ID` in the DAG must have a corresponding engspec, and vice versa — flag orphans on either side. Confirm the DAG's dependency edges don't contradict any Constraint in an engspec (e.g., the DAG says two stories can run in parallel but one's Constraints reference the other's capability as a precondition).

**Tier justification.** For each task, confirm the DAG cited an actual trigger from the risk rubric, not just an assigned number. A task sitting at Tier 1 despite an engspec Open Question, an elevated `risk_indicator`, or a security-sensitive component is a real gap — auto-approval gates are only safe if the tiering underneath them is honest.

**No cycles, no stranded Open Questions.** Any dependency cycle is a hard fail. Any engspec with unresolved Open Questions sitting at an auto-approvable tier (1 or 2) is a hard fail, regardless of how the DAG tiered it — an unresolved question means a human hasn't actually confirmed the spec gate can be skipped.

## Output format

```markdown
---
doc_type: build_readiness_review
schema_version: 1
initiative: <slug>
scope: <epic id or "full initiative">
generated_by: build-readiness-checker
source_files: [<engspec files>, <task-dag.md>]
date: <today, ISO format>
review_verdict: build_ready | gaps_found | not_ready
---

# Build Readiness Review — <initiative/epic>

**Overall result:** Build ready — cleared for BMAD handoff | Gaps found | Not ready — do not proceed to build

## Summary
[3-5 sentences: how many stories, how many passed clean, the most significant gaps if any, and which Tier 3 tasks (if any) need standing human attention from the first gate.]

## Findings

| Story | Check | Status | Note | Route back to |
|---|---|---|---|---|
| STORY-ID | Success-signal testability | Gap | signal still `(proposed — confirm)` | engineering-spec-writer |
| STORY-ID | Tier justification | Gap | Tier 1 assigned despite unresolved Open Question | task-dag-builder |

## Cleared for handoff

[List of STORY-IDs that passed every check, grouped by DAG wave, ready to feed into `bmad-build`/`bmad-build-auto` per their tier.]
```
