---
name: task-dag-builder
description: Takes every engineering spec produced for an epic (or a whole initiative) and produces a task DAG — build order, parallelizable groups, cross-story dependencies, and a risk tier per task that determines which human approval gates apply during automated build. Use this whenever the user wants stories "sequenced," "ready for a sprint/build order," asks "what can we parallelize," "what depends on what before we start building," or "what's the risk tier on these." This is stage 13 of the Build Factory, sitting between engineering-spec-writer and build-readiness-checker.
---

# Task DAG Builder

## Pipeline contract

Stage 13 of the Build Factory (see `engineering-spec-writer`'s pipeline contract table for the full chain). Reads every `{slug}-story-<STORY-ID>-engspec.md` for the epic or initiative in scope, plus `{slug}-architecture.md` for component boundaries and, if it exists, `{slug}-relationship-graph.md` (from `relationship-graph-builder`, if that utility has been run) for explicit `depends-on`/`blocks` edges between stories or components. Writes `{slug}-task-dag.md` (`doc_type: task_dag`).

**Input contract:** at least two engspec files (a DAG of one node is just a task list — still valid, but say so plainly rather than presenting a trivial case as if real sequencing work happened). If `{slug}-relationship-graph.md` doesn't exist, derive dependencies yourself from what the engspecs state (shared `COMP-ID` ownership, a Constraint in one story referencing another story's capability, an ADR shared across stories that implies ordering) and say you did so without that utility's more systematic pass.

**Output contract:** always write to `{slug}-task-dag.md`. This is the file that gets read alongside individual engspecs when handing work to BMAD — its risk tier per task tells the orchestrator (human or scripted) which of the five gates in the matrix below need a human before that task's `bmad-build`/`bmad-build-auto` run can proceed to the next stage. Tell the user the build order, what can run in parallel, and flag any task assigned Tier 3 before they start, since those need a human at every gate.

## The risk-tier gate matrix

This is a fixed, five-gate matrix — don't redesign it per initiative, only assign which tier each task sits at:

| Gate | What it checks | Tier 1 (low) | Tier 2 (medium) | Tier 3 (critical) |
|---|---|---|---|---|
| Spec gate | Is the engspec's intent correct? | auto-approved | **human** | **human** |
| Plan gate | Is bmad-build's implementation plan sound? | auto-approved | auto-approved | **human** |
| Diff gate | Is the produced code right? | auto-approved | auto-approved | **human** |
| Results gate | Did verification (tests/CI) actually pass? | auto-approved | auto-approved | **human** |
| Release gate | Safe to ship? | auto-approved | **human** | **human** |

**Assigning a tier per task** — use the highest tier any one of these triggers:
- Touches authentication, payments, PII, or any component an ADR marks as security-sensitive → **Tier 3**
- The engspec's `risk_indicator` is `elevated`, or its Success signal is still `(proposed — confirm)` → at least **Tier 2**
- No existing automated test coverage exists for the touched component (say so if you can't tell — don't assume coverage exists) → at least **Tier 2**
- Default, nothing above triggers → **Tier 1**

New task types should start at Tier 3 regardless of the above, until there's a track record — note this explicitly for any component/task type the initiative hasn't shipped through the Build Factory before.

## Process

1. Read every engspec in scope. Build a node per `STORY-ID`.
2. Derive edges: two stories that touch the same `COMP-ID` where one's Capability reads/depends on state the other writes are sequential (A before B); stories touching disjoint components with no shared Constraint can run in parallel. State your reasoning for any non-obvious edge — don't just assert an ordering.
3. Topologically sort into build waves (groups that can run in parallel within a wave, waves run in sequence).
4. Assign a risk tier per task using the rubric above, citing which trigger applied.
5. Flag any cycle you find (story A depends on B which depends on A) as a blocking issue — this usually means a story boundary is wrong upstream, not something to silently break by picking an arbitrary order.
6. Note any task whose engspec has unresolved Open Questions — these should not enter Tier 1 or 2 auto-approval regardless of what the rubric otherwise says, since an unresolved question means the spec gate can't be trusted to auto-pass.

## Output format

```markdown
---
doc_type: task_dag
schema_version: 1
initiative: <slug>
scope: <epic id or "full initiative">
generated_by: task-dag-builder
source_files: [<engspec files>, <architecture.md>, <relationship-graph.md if used>]
date: <today, ISO format>
---

# Task DAG — <initiative/epic>

## Build order

**Wave 1 (parallel):** STORY-ID (Tier N), STORY-ID (Tier N)
**Wave 2 (parallel):** STORY-ID (Tier N)
...

## Dependency edges

| From | To | Reason |
|---|---|---|
| STORY-ID | STORY-ID | [shared COMP-ID / shared constraint / etc.] |

## Risk tiers

| Story | Tier | Trigger(s) |
|---|---|---|
| STORY-ID | 2 | risk_indicator: elevated; no existing test coverage on COMP-ID |

## Flags

- [cycles, unresolved-open-question tasks forced to a higher effective tier, new-task-type Tier-3 overrides]
```
