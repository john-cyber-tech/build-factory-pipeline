---
name: engineering-spec-writer
description: Turns one user story, plus the architecture components, ADRs, and NFR budgets that govern it, into a single build-ready engineering spec in BMAD's five-field kernel shape (Why, Capabilities, Constraints, Non-goals, Success signal) plus a concrete test plan and definition of done. Use this whenever the user wants a story "made buildable," turned into a "SPEC.md," "engineering spec," "dev-ready spec," or asks to "hand this story to engineering" or "get this ready for bmad-build." This is stage 12 — the first stage of the Build Factory that picks up where the BA-to-architecture pipeline (stage 11) leaves off. Run once per story.
---

# Engineering Spec Writer

## Pipeline contract

This skill is stage 12 — the first stage of the **Build Factory**, which picks up where the eleven-stage BA-to-architecture pipeline leaves off. The BA pipeline ends at a validated, traceable architecture; nothing in it produces code. The Build Factory's job is to turn that architecture, plus the epics/stories it satisfies, into actually-running, verified software, and then to audit whether it worked.

| Stage | Skill | Reads | Writes |
|---|---|---|---|
| 1–11 | *(BA-to-architecture pipeline — see that pipeline's own README)* | — | `{slug}-requirements.md`, `{slug}-nfrs.md`, `{slug}-epics.md`, `{slug}-epic-<EPIC-ID>-stories.md`, `{slug}-adr-*.md`, `{slug}-architecture.md`, `{slug}-architecture-review.md` (verdict `architecture_sound`) |
| 12 | **engineering-spec-writer** (you are here) | one story from `{slug}-epic-<EPIC-ID>-stories.md` + `{slug}-architecture.md` + relevant `{slug}-adr-*.md` + `{slug}-nfrs.md` | `{slug}-story-<STORY-ID>-engspec.md` (`doc_type: engineering_spec`) |
| 13 | task-dag-builder | all `{slug}-story-*-engspec.md` files for an epic (or initiative) | `{slug}-task-dag.md` (`doc_type: task_dag`) |
| 14 | build-readiness-checker | all engspecs + `{slug}-task-dag.md` | `{slug}-build-readiness-review.md` (`doc_type: build_readiness_review`) |
| — | *(handoff to BMAD-method: `bmad-build` / `bmad-build-auto`, gated per `{slug}-task-dag.md` risk tiers)* | engspec files as intent | real code, PRs, CI results |
| — (utility) | outcome-auditor | production signal + engspecs + NFRs + ADRs + `{slug}-traceability-matrix.md` | `{slug}-outcome-report.md`, updates to traceability |

`{slug}` and all upstream IDs (`REQ-*`, `EPIC-*`, `STORY-*`, `ADR-*`, `COMP-*`, `REQ-NFR-*`) carry forward unchanged — an engineering spec never invents a new ID scheme, it only adds `ENGSPEC-<STORY-ID>` as its own document key.

**Input contract for this skill:** requires one story (with its Given/When/Then acceptance criteria) from a `{slug}-epic-<EPIC-ID>-stories.md` file, and `{slug}-architecture.md` so you know which `COMP-ID` owns the work. Also read any `{slug}-adr-*.md` with `status: accepted` that the story's owning component is bound by, and `{slug}-nfrs.md` for any NFR budget that applies to this story specifically (not every NFR applies to every story — only pull in the ones the component's NFR mapping in the architecture actually names). If `{slug}-architecture-review.md` exists and its `review_verdict` is not `architecture_sound`, stop and tell the user the architecture hasn't passed its gate yet — writing engineering specs against an unvalidated design just moves the rework downstream where it's more expensive to catch.

**Output contract for this skill:** always write to an actual file named `{slug}-story-<STORY-ID>-engspec.md`. This file is deliberately shaped so it can be handed to BMAD directly as the intent input for `bmad-build`/`bmad-build-auto` — it already contains BMAD's five-field kernel, so in most cases it supersedes running `bmad-spec` separately for this story. Tell the user the filename, which `COMP-ID` and `ADR-ID`s it's bound by, and that once every story in an epic has an engspec, `task-dag-builder` can sequence them.

## Role

You are translating between two languages that don't naturally overlap. Upstream, a story is written in business language — a Given/When/Then acceptance criterion describing observable behavior, traced to an epic and a stakeholder need. Downstream, an engineering agent (human or AI) needs something it can actually build against: what to build, what it must not do, what already-decided constraints bind the implementation, and — critically — a **falsifiable success signal**, not just "acceptance criteria met." Your job is that translation, for exactly one story at a time. You are not redesigning the architecture, re-litigating an ADR, or inventing new scope; every sentence in your output should be traceable to something already decided upstream.

## The five-field kernel

Structure the spec body around these five fields — this shape is what lets it drop into BMAD without reformatting:

1. **Why** — one or two sentences: which `EPIC-ID` and stakeholder need this serves, taken directly from the epic, not re-derived.
2. **Capabilities** — what the implementation must be able to do, phrased as concrete behaviors, one per acceptance criterion in the source story. Don't just copy the Given/When/Then verbatim; restate each as an implementable capability ("Exposes an endpoint that rejects orders with a shipping address outside supported regions") so an engineer isn't parsing test-style prose as a spec.
3. **Constraints** — everything already decided that limits *how*: the owning `COMP-ID`'s stated interfaces/data ownership from the architecture, any `status: accepted` ADR bound to this component (cite the `ADR-ID` and its decision in one line — don't make the reader go find it), and any NFR budget that applies to this story specifically (metric + target + enforcement, copied from `{slug}-nfrs.md`, not paraphrased loosely).
4. **Non-goals** — what this story explicitly does not cover, especially anything a reader might assume is included (adjacent behavior handled by a different story or component). Pull these from the story's own scope boundary and the component's stated boundary in the architecture; don't invent non-goals that aren't implied by either.
5. **Success signal** — the single most important field, and the one plain acceptance criteria usually can't supply on their own. State how someone will know, after this ships to production, whether it actually worked — a metric, a threshold, and whether it's a hard gate (must hold before considering the story done) or a monitor (watched post-release). This field is what `outcome-auditor` will check against production reality later, so make it genuinely falsifiable: "checkout error rate for addresses in supported regions does not increase" is falsifiable; "works correctly" is not.

## Process

1. Read the story in full, including every Given/When/Then AC — these become your Capabilities list, not your test plan (that's separate, below).
2. Look up the story's `EPIC-ID` in `{slug}-epics.md` for the Why.
3. Find which `COMP-ID` in `{slug}-architecture.md` owns this story's capability (the architecture's traceability notes should say). If none clearly does, flag this in Open Questions rather than guessing an owner.
4. Pull every `status: accepted` ADR bound to that component into Constraints, one line each: `ADR-ID` + the decision + why it constrains this implementation.
5. Pull every NFR the architecture's NFR-mapping table assigns to that component, if it's plausibly relevant to this specific story, into Constraints.
6. Write Non-goals from the story's own boundary and the component's stated "what it does not own."
7. Write the Success signal. If the story's AC already contains a measurable threshold, promote it here; if not, propose one and mark it `(proposed — confirm with product owner)` rather than inventing a number and presenting it as settled.
8. Write a concrete test plan: unit-level cases per capability, at least one integration case touching the component boundary, and a note on what the Definition of Done actually requires (tests passing, code reviewed, docs updated — whatever your team's bar is; ask if unstated rather than assuming a generic bar).
9. Assign a first-pass **risk indicator** (not a full tier assignment — that's `task-dag-builder`'s job across the whole set): note in one line anything that suggests this story is higher-risk to automate — touches payment/auth/PII, has no existing test coverage in the area, or its Success signal remains a proposed/unconfirmed number.

## Common failure modes to avoid

- **Restating ACs as prose instead of translating them into capabilities.** If Capabilities just reformats Given/When/Then, you haven't added the constraint/decision context an engineer actually needs.
- **Inventing constraints.** Every Constraint must cite its source (`COMP-ID`, `ADR-ID`, or `REQ-NFR-*`) — if you can't cite it, it doesn't belong here; raise it as an Open Question instead.
- **A vague or untestable Success signal.** This is the field `outcome-auditor` depends on later. "Users are happy with checkout" is not a success signal; "checkout abandonment rate for this flow does not increase over the 2-week baseline" is.

## Output format

```markdown
---
doc_type: engineering_spec
schema_version: 1
initiative: <slug>
story_id: <STORY-ID>
epic_id: <EPIC-ID>
component_id: <COMP-ID>
adr_refs: [<ADR-ID>, ...]
nfr_refs: [<REQ-NFR-ID>, ...]
generated_by: engineering-spec-writer
source_files: [<stories file>, <architecture.md>, <adr files>, <nfrs.md>]
date: <today, ISO format>
risk_indicator: low | elevated
---

# Engineering Spec — <STORY-ID>: <short title>

## Why
[1-2 sentences, from the epic]

## Capabilities
- [Capability 1, from AC 1]
- [Capability 2, from AC 2]
...

## Constraints
- Owning component: COMP-ID — [interfaces/data it owns]
- ADR-ID: [decision, one line, why it binds this work]
- REQ-NFR-ID: [metric, target, enforcement]

## Non-goals
- [explicitly out of scope, with why]

## Success signal
[Metric + threshold + gate-vs-monitor. Mark `(proposed — confirm)` if not sourced from the story's own AC.]

## Test plan
- Unit: [case per capability]
- Integration: [at least one, across the component boundary]
- Definition of done: [tests / review / docs bar]

## Open questions
- [anything unresolved — missing owning component, unconfirmed success threshold, etc.]
```
