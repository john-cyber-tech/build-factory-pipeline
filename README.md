# The Build Factory — Reference Documentation

**Version:** Phase 1
**Scope:** 4 skills — 3 numbered pipeline stages (12–14) plus 1 utility — that pick up where the BA-to-Architecture pipeline (stages 1–11) leaves off, hand off to BMAD-method for real code execution, and close the loop with production outcome auditing.
**Audience:** Written to be sufficient, on its own, for an LLM agent or a human operator to run any stage of this pack correctly without first reading the individual `SKILL.md` files. The `SKILL.md` files remain authoritative for exact wording and edge cases; this README is the map, the contract, and the operating manual — same role its companion README plays for the BA-to-Architecture pipeline.

---

## 1. What this pack is, and why it exists

The BA-to-Architecture pipeline (13 skills, stages 1–11 plus two chain-spanning utilities) takes an initiative from raw discovery material through to a **validated, traceable solution architecture**. That is where it stops. Nothing in it writes code, runs a test, opens a pull request, or knows whether anything shipped actually worked in production.

The Build Factory is the second half. It exists to answer one question the first pipeline was never built to answer: **given a validated architecture, how does an organization reliably get to running, verified software — and then find out whether it was the right software?**

It does this in three moves:

1. **Translate** (stages 12–13): turn each story from the BA pipeline into something an execution system can actually build from, and sequence those specs into a build order with risk-appropriate human gates.
2. **Gate** (stage 14): check that translation before any code gets written, the same author/checker discipline the BA pipeline uses at every stage.
3. **Hand off and audit**: real code execution happens in BMAD-method, not in a document-authoring skill — that is a deliberate scope boundary, explained in §7. Once something ships, `outcome-auditor` (a utility, like `traceability-matrix-generator` in the BA pipeline) checks reality against every prediction the whole chain made, and drafts new input for the BA pipeline's own stage 1 or 2 when reality didn't match the prediction. This is the loop the BA pipeline's own gap analysis against outside reference material flagged as entirely absent: *outcome auditing — closing the loop from production metrics back to the original spec.*

## 2. Operating assumptions & dependencies

These carry over from the BA-to-Architecture pipeline unchanged, plus two new ones specific to this pack:

- **File I/O only, no inter-skill memory.** Every skill in this pack reads and writes plain Markdown files with YAML frontmatter. A skill has no memory of a previous run; everything it needs must be in the files it's given. This is identical to the BA pipeline's own operating model — the two packs interoperate because they share this convention, not because either calls the other.
- **Slug and ID permanence.** `{slug}` and every upstream ID (`REQ-*`, `EPIC-*`, `STORY-*`, `ADR-*`, `COMP-*`, `REQ-NFR-*`) are carried forward unchanged from the BA pipeline's output. This pack never renumbers or reissues an upstream ID; it only adds new ID types of its own (see §3).
- **No-fabrication discipline.** A constraint, a dependency edge, a risk-tier trigger, or a production verdict must cite the file and ID it came from. If a skill in this pack can't cite a source for something, it says so as an Open Question or an "unaudited" finding — it does not fill the gap with a plausible-sounding invention. This is the single rule every skill below repeats in its own words, because it's the rule most likely to quietly slip under time pressure.
- **No external tools required to author the documents.** Writing an engineering spec, a task DAG, or a readiness review needs nothing beyond the input files and the model itself — same as the BA pipeline.
- **New — real execution is explicitly out of scope for stages 12–14.** Unlike the BA pipeline, where every stage's output is the final artefact for that concern, stages 12–14 of the Build Factory produce artefacts that are consumed by a *different system* (BMAD-method) to do the actual work of writing and verifying code. This pack does not reimplement an agentic coding loop, a CI runner, or a git/PR workflow — see §7 for exactly where the handoff happens and what BMAD is responsible for from that point on.
- **New — `outcome-auditor` requires real production signal supplied by the user.** It cannot query a live system, a monitoring dashboard, or an incident tracker on its own. Every verdict it produces is only as good as the evidence it was actually handed, and it must say plainly, per story/NFR/ADR, when no evidence was provided rather than assuming a silent pass.

## 3. File / frontmatter / ID conventions

### 3.1 `doc_type` registry (this pack's additions)

The BA pipeline's own registry (`stakeholder_needs`, `requirements`, `requirements_review`, `nfrs`, `epics`, `epics_review`, `stories`, `stories_review`, `adr`, `architecture`, `architecture_review`, plus utilities `traceability_matrix` and `glossary`) is extended by four new values:

| `doc_type` | Produced by | Filename pattern |
|---|---|---|
| `engineering_spec` | engineering-spec-writer | `{slug}-story-<STORY-ID>-engspec.md` |
| `task_dag` | task-dag-builder | `{slug}-task-dag.md` |
| `build_readiness_review` | build-readiness-checker | `{slug}-build-readiness-review.md` |
| `outcome_report` | outcome-auditor | `{slug}-outcome-report.md` |

### 3.2 ID-prefix registry (this pack's additions)

The BA pipeline's ID prefixes (`REQ-BR-`, `REQ-FR-`, `REQ-NFR-`, `EPIC-`, `STORY-`, `ADR-`, `COMP-`) are never reissued by this pack. This pack introduces exactly one new document-level key, not a new per-item ID scheme:

| Key | Meaning |
|---|---|
| `ENGSPEC-<STORY-ID>` | An engineering spec's own document identity — always derived from the `STORY-ID` it translates, never independently numbered. There is exactly one engspec per story. |

Task DAGs and readiness reviews reference existing `STORY-ID`s directly; they don't need a new ID scheme of their own.

### 3.3 Frontmatter fields specific to this pack

Beyond the shared fields every pipeline document carries (`doc_type`, `schema_version`, `initiative`, `generated_by`, `source_files`, `date`), this pack's documents add:

- `engineering_spec`: `story_id`, `epic_id`, `component_id`, `adr_refs` (list), `nfr_refs` (list), `risk_indicator` (`low` | `elevated`)
- `task_dag`: `scope` (an `EPIC-ID` or the literal string `"full initiative"`)
- `build_readiness_review`: `scope`, `review_verdict` (`build_ready` | `gaps_found` | `not_ready`)
- `outcome_report`: no extra fields beyond the shared set — its verdicts live in the body, per story/NFR/ADR, because a single top-level verdict would hide exactly the kind of mixed result (story shipped clean, NFR quietly missed) this skill exists to surface.

## 4. How this pack feeds from the BA-to-Architecture pipeline

This is the interoperability contract — exactly which BA-pipeline output files this pack reads, and at which stage:

| BA pipeline output file | `doc_type` | Consumed by (this pack) | Used for |
|---|---|---|---|
| `{slug}-epic-<EPIC-ID>-stories.md` | `stories` | engineering-spec-writer | The one story being translated — its Given/When/Then ACs become Capabilities. |
| `{slug}-epics.md` | `epics` | engineering-spec-writer | The story's owning `EPIC-ID`, for the spec's "Why." |
| `{slug}-architecture.md` | `architecture` | engineering-spec-writer, task-dag-builder | Which `COMP-ID` owns the story; component interfaces/data ownership become Constraints; component boundaries inform dependency edges. |
| `{slug}-architecture-review.md` (`review_verdict: architecture_sound`) | `architecture_review` | engineering-spec-writer | **Gate check** — engineering-spec-writer refuses to proceed if this file exists and its verdict isn't clean, since writing build-ready specs against an unvalidated design just moves rework downstream. |
| `{slug}-adr-<ADR-ID>-*.md` (`status: accepted`) | `adr` | engineering-spec-writer, task-dag-builder, outcome-auditor | Binding decisions become cited Constraints; security-sensitive ADRs feed the risk-tier rubric; an ADR's Consequences section becomes a falsifiable prediction `outcome-auditor` checks later. |
| `{slug}-nfrs.md` | `nfrs` | engineering-spec-writer, outcome-auditor | NFR budgets relevant to a specific story become cited Constraints; every NFR with a concrete target becomes something `outcome-auditor` later checks against real measured values. |
| `{slug}-relationship-graph.md` (if present) | *(BA pipeline utility output)* | task-dag-builder | Explicit `depends-on`/`blocks` edges, when available, replace task-dag-builder's own inference of dependencies from shared component ownership. |
| `{slug}-traceability-matrix.md` (if present) | `traceability_matrix` | outcome-auditor | Not edited directly — `outcome-auditor` recommends it be regenerated with production status folded in as a new terminal column once its own report exists. |

**Nothing in this pack ever edits a BA-pipeline output file.** The relationship is strictly read-then-produce-new-file, exactly like every stage boundary inside the BA pipeline itself. The only file this pack ever asks to be regenerated by *the other pack* is `{slug}-traceability-matrix.md`, and even then it's a recommendation to re-run that pipeline's own utility, not an edit performed here.

**What flows back upstream, and where.** `outcome-auditor`'s remediation-need entries are the one place this pack writes something explicitly meant to re-enter the BA pipeline — formatted so they can be pasted directly as input to `stakeholder-discovery` (if the gap suggests a need nobody had identified) or `requirements-writer` (if the need is already well-understood and just needs a formal requirement). This is what actually closes the loop end to end: BA pipeline → Build Factory → real production signal → new BA pipeline input.

## 5. Full pipeline diagram

```
                    ── BA-to-Architecture pipeline (stages 1–11) ──
 1  stakeholder-discovery
 2  requirements-writer            ──▶ 3  requirements-checker ──(loop back)──┐
 4  nfr-elicitor                                                              │
 5  requirements-to-epics          ──▶ 6  epics-vs-requirements-checker ──────┤
 7  epic-to-story-decomposer       ──▶ 8  story-checker ──────────────────────┤
 9  adr-writer                                                                │
10  solution-architecture-writer   ──▶ 11 architecture-vs-requirements-checker┘
    (loops back to 10 if review_verdict != architecture_sound)

    utilities, any point: traceability-matrix-generator · domain-glossary-builder ·
                            relationship-graph-builder
════════════════════════════ BA pipeline output boundary ══════════════════════════

                         ── Build Factory (stages 12–14) ──
12  engineering-spec-writer   (once per story)
        │  gate: refuses if architecture-review verdict isn't architecture_sound
        ▼
13  task-dag-builder          (once per epic / initiative)
        ▼
14  build-readiness-checker  ──(loop back to 12 or 13, by finding)──┐
        │  review_verdict: build_ready                             │
        ▼                                                          │
════════════════════ Build Factory → BMAD handoff boundary ════════│═══════════════
        │                                                          │
    BMAD-method: bmad-build / bmad-build-auto, gated per           │
    task-dag risk tier (spec / plan / diff / results / release)    │
        │                                                          │
    your CI/CD: canary, rollback  (outside both packs' scope)      │
        │                                                          │
        ▼                                                          │
    outcome-auditor  (utility, runs post-deployment, any time      │
                       new production signal arrives)              │
        │                                                          │
        └─▶ remediation needs ──▶ back to BA pipeline stage 1 or 2 ─┘
             (new cycle)                                      (or straight to 12/13
                                                                 if only re-work is
                                                                 needed, no new need)
```

## 6. The four skills in detail

### 6.1 `engineering-spec-writer` — Stage 12

**Purpose.** Translates one story plus its governing architecture component, accepted ADRs, and relevant NFR budgets into a single build-ready engineering spec, shaped so it can be fed directly into BMAD's `bmad-build`/`bmad-build-auto` as intent.

**Trigger phrases.** "make this story buildable," "turn this into a SPEC.md," "engineering spec," "dev-ready spec," "hand this to engineering," "get this ready for bmad-build."

**Inputs.** One story from `{slug}-epic-<EPIC-ID>-stories.md`; `{slug}-architecture.md`; any `{slug}-adr-*.md` with `status: accepted` bound to the owning component; `{slug}-nfrs.md` for story-relevant budgets; `{slug}-architecture-review.md` as a gate check.

**Outputs.** `{slug}-story-<STORY-ID>-engspec.md` — Why / Capabilities / Constraints / Non-goals / Success signal, plus a test plan and a first-pass risk indicator.

**Key rule.** Every Constraint must cite a `COMP-ID`, `ADR-ID`, or `REQ-NFR-ID`. The Success signal must be falsifiable — a metric and a threshold, not a restated capability.

**Common failure mode.** Restating Given/When/Then as prose instead of translating each into a genuinely new Capability statement with constraint context attached.

**Downstream consumers.** `task-dag-builder` (reads every engspec in scope), `build-readiness-checker` (audits it), BMAD (consumes it directly as build intent), `outcome-auditor` (checks its Success signal against production reality, later).

### 6.2 `task-dag-builder` — Stage 13

**Purpose.** Sequences every engineering spec in an epic or initiative into a build order — parallelizable waves plus dependency edges — and assigns each task a risk tier against the fixed 5-gate matrix.

**Trigger phrases.** "sequence these stories," "build order," "what can we parallelize," "what depends on what," "what's the risk tier on these."

**Inputs.** Every `{slug}-story-<STORY-ID>-engspec.md` in scope; `{slug}-architecture.md`; `{slug}-relationship-graph.md` if it exists.

**Outputs.** `{slug}-task-dag.md` — build waves, dependency edges with stated reasons, and a risk tier per task with its triggering rule cited.

**Key rule.** A task's tier is the *highest* tier any single trigger fires (security-sensitive component → Tier 3; elevated risk indicator or unconfirmed success signal → at least Tier 2; no existing test coverage → at least Tier 2; new task type with no track record → Tier 3 regardless). Cycles are a hard fail, not something to silently resolve by picking an arbitrary order.

**Common failure mode.** Assigning Tier 1 to a task whose engspec still has an unresolved Open Question — an unresolved question should never sit at an auto-approvable tier.

**Downstream consumers.** `build-readiness-checker` (audits DAG consistency against the engspec set), BMAD orchestration (reads the DAG to decide which gates apply to which `bmad-build` run).

### 6.3 `build-readiness-checker` — Stage 14

**Purpose.** The critic gate over stages 12 and 13 combined — the same author/checker discipline the BA pipeline applies at every authoring stage, applied here to two authors whose outputs must also be checked *against each other*, not just individually.

**Trigger phrases.** "validate these engineering specs," "are we ready to build this," "check the task DAG," "ready for bmad-build."

**Inputs.** Every engspec in scope; `{slug}-task-dag.md`.

**Outputs.** `{slug}-build-readiness-review.md` — findings routed explicitly to `engineering-spec-writer` (spec-level gaps) or `task-dag-builder` (sequencing/tier gaps) by name.

**Key rule.** A dependency cycle, or any auto-approvable-tier task with an unresolved Open Question, is a hard fail regardless of what the DAG's own tiering claims.

**Common failure mode.** Treating an uncited Constraint as a pass because it merely sounds plausible — provenance is the check, not plausibility.

**Downstream consumers.** The human or orchestrator deciding whether to actually invoke BMAD for this batch — a `build_ready` verdict is the literal green light.

### 6.4 `outcome-auditor` — Utility, runs post-deployment

**Purpose.** Checks real production signal against three separate levels of prediction the whole chain made — a story's Success signal, an NFR's target, and an ADR's stated Consequences — and drafts remediation needs formatted for re-entry into the BA pipeline.

**Trigger phrases.** "did this work in production," "audit the outcome," "check our NFRs against real numbers," "did the ADR trade-off pay off," "outcome auditing," "production feedback."

**Inputs.** Whichever engspecs, `{slug}-nfrs.md`, `{slug}-adr-*.md`, and `{slug}-traceability-matrix.md` exist, plus production signal supplied by the user — metrics, incident reports, dashboards described in chat, monitoring exports. It cannot query a live system itself.

**Outputs.** `{slug}-outcome-report.md` — three separate verdict tables (story/Success-signal, NFR/budget, ADR/Consequences), each row marked Met / Missed / Degraded / Unaudited, plus a Remediation Needs section pre-formatted for `stakeholder-discovery` or `requirements-writer`.

**Key rule.** Keep the three levels visibly separate — a story can hit its Success signal while its component quietly misses an NFR budget, and collapsing that into one verdict would hide the finding. Never fabricate a remediation need for a story with no evidence either way; mark it Unaudited instead.

**Downstream consumers.** `traceability-matrix-generator` (recommended re-run, to fold production status in as a terminal column), `stakeholder-discovery` / `requirements-writer` (remediation needs become new cycle input).

## 7. Where BMAD-method fits, precisely

BMAD is not part of this pack — it's the execution system this pack hands off to, and it's a separate, independently maintained tool (github.com/bmad-code-org/bmad-method) you install into the target repository, not a skill you run inside a chat.

**What crosses the boundary.** `{slug}-task-dag.md` plus the cleared engspecs from `build-readiness-checker`'s output. In most cases an engspec's five-field kernel (Why / Capabilities / Constraints / Non-goals / Success signal) already matches what `bmad-spec` would otherwise produce, so you can typically skip running `bmad-spec` separately and feed the engspec straight in as the intent for `bmad-build` / `bmad-build-auto`.

**What the risk tier controls.** BMAD's own spec `status` field (`draft → ready-for-dev → in-progress → in-review → done → blocked`) is the gate *mechanism*; the task DAG's risk tier decides which of the five transitions in that state machine require a human before continuing, using the fixed matrix:

| Gate | What it checks | Tier 1 | Tier 2 | Tier 3 |
|---|---|---|---|---|
| Spec gate | Is the engspec's intent correct? | auto | **human** | **human** |
| Plan gate | Is `bmad-build`'s implementation plan sound? | auto | auto | **human** |
| Diff gate | Is the produced code right? | auto | auto | **human** |
| Results gate | Did verification (tests/CI) actually pass? | auto | auto | **human** |
| Release gate | Safe to ship? | auto | **human** | **human** |

**What stays outside both packs.** BMAD is explicit that `bmad-build-auto` commits but does not push — real CI/CD (build pipeline, canary rollout, automated rollback) and real production telemetry are infrastructure the adopting team supplies. `outcome-auditor` is only as useful as the telemetry you feed it; without real monitoring in place, the outcome-auditing loop is a manual spot-check rather than a running feedback system.

## 8. Decision procedure — which skill to invoke for an arbitrary request

1. Does the request name a specific story that needs to become buildable, or ask to "spec out" / "make buildable" a story? → `engineering-spec-writer`.
2. Does it ask about build order, parallelization, sequencing, or risk tiers across multiple stories? → `task-dag-builder` (requires engspecs to already exist for the stories in scope).
3. Does it ask whether a set of specs/DAG is "ready," "validated," or "safe to build from"? → `build-readiness-checker`.
4. Does it ask about production behavior, whether something "worked," NFR/ADR reality-checking, or "closing the loop"? → `outcome-auditor`.
5. Does it ask about the BA-to-Architecture side (stakeholder needs through architecture)? → that's the other pipeline; see its own README.
6. Does it ask to actually invoke BMAD, write code, or run `bmad-build`? → that's outside this pack's skills entirely; point to the handoff described in §7.

## 9. Change log

**Phase 1 (this release).** Initial four skills: `engineering-spec-writer`, `task-dag-builder`, `build-readiness-checker`, `outcome-auditor`. Fixed 5-gate risk-tier matrix adopted unchanged from the standalone dark-factory execution architecture scoped previously. BMAD-method adopted as the execution/verification layer rather than a custom orchestrator.

**Open item.** This pack does not yet include a "Decision Memory" artefact spanning the whole SDLC (a broader version of what ADRs cover only at the architecture stage) — noted as a candidate for a future phase, not built here.

## 10. Verification checklist

- [ ] Run `engineering-spec-writer` against a story whose architecture-review verdict is *not* `architecture_sound`; confirm it refuses and names the gate rather than proceeding.
- [ ] Run it against a normal, clean story; confirm every Constraint line cites a real `COMP-ID`/`ADR-ID`/`REQ-NFR-ID` from the actual input files, not an invented one.
- [ ] Run `task-dag-builder` against a set of engspecs with one unresolved Open Question; confirm that task is not silently placed at Tier 1.
- [ ] Introduce a deliberate cycle (story A's constraints reference story B, and vice versa); confirm `task-dag-builder` flags it as a hard fail rather than picking an arbitrary order.
- [ ] Run `build-readiness-checker` against a DAG that references a `STORY-ID` with no corresponding engspec; confirm it flags the orphan rather than reviewing a partial set silently.
- [ ] Run `outcome-auditor` with production signal covering only some stories in an initiative; confirm the unaudited stories are marked Unaudited, not assumed to have passed.
- [ ] Confirm `outcome-auditor`'s remediation-need entries are worded so they can be pasted directly into `stakeholder-discovery` or `requirements-writer` without further editing.
