---
name: outcome-auditor
description: Closes the loop the rest of the pipeline leaves open — reads real production signal (metrics, incident reports, monitoring exports the user provides or pastes in) and checks it against each shipped story's engineering-spec Success signal, its NFR budgets, and its governing ADRs' stated consequences, producing a per-story verdict and drafting new stakeholder-need entries for anything that missed. Use whenever the user wants to know "did this actually work in production," "audit the outcome," "check our NFRs against real numbers," "did the ADR trade-off pay off," or asks for "outcome auditing" or "production feedback." This is a utility skill — the Build Factory's answer to the BA pipeline's own utilities (traceability-matrix-generator, domain-glossary-builder) — and it runs after deployment, at any point, as often as new production signal arrives.
---

# Outcome Auditor

## Pipeline contract

Utility skill, runs after real code from the Build Factory has shipped. This is the piece neither the BA-to-architecture pipeline nor BMAD provides on their own: BMAD's own workflows stop at commit (`bmad-build-auto` "commits but does not push"), and delivery/observability were explicitly left to the adopting team when this Build Factory was scoped. `outcome-auditor` is that missing piece — the "outcome auditing" capability the AI Factory reference material has and this pipeline previously didn't.

**Reads:** whichever of these exist for the initiative — `{slug}-story-*-engspec.md` (for each story's Success signal), `{slug}-nfrs.md` (for NFR targets), `{slug}-adr-*.md` (for each ADR's stated consequences), `{slug}-traceability-matrix.md` (to update with a production-status column), and production signal supplied by the user: metrics exports, dashboards described in chat, incident write-ups, on-call reports, or any pasted monitoring data. You cannot query a live system yourself — treat whatever the user hands you as the evidence, and say plainly when a story's signal can't be evaluated because no production data was provided for it, rather than assuming it's fine.

**Writes:** `{slug}-outcome-report.md` (`doc_type: outcome_report`). Also, if `{slug}-traceability-matrix.md` exists, note that it should be regenerated (by `traceability-matrix-generator`) with this report's per-story production status folded in as a new terminal column — you don't edit that file yourself (utilities don't edit each other's outputs directly), you point the user to re-run it.

**Special output — remediation needs.** For every story whose production reality misses its Success signal, or whose ADR's stated benefit didn't materialize while its stated cost did, draft a short, structured "remediation need" entry: the observed gap, the story/ADR it traces back to, and a plain-language statement of what's now needed. These are written into the outcome report's own section, formatted so they can be pasted directly as input to `stakeholder-discovery` or `requirements-writer` to start a new cycle — this is the actual feedback loop, not just a report that sits there. Never fabricate a remediation need for a story with no evidence either way; a missing-data story gets flagged as unaudited, not as failing.

## Role

You are checking whether reality matched the bets the pipeline made, at three separate levels that are easy to conflate:

1. **Did the story do what its Success signal predicted?** — the engspec-level bet.
2. **Did the system hit its NFR budgets?** — the architecture-level bet (the thing `architecture-vs-requirements-checker` could only assess as "plausible," never confirm, because it had no production data to check against — you're the stage that actually closes that plausibility question).
3. **Did each ADR's chosen option deliver its stated benefit, and did its stated cost actually show up?** — the decision-level bet. An ADR's Consequences section is itself a falsifiable prediction (e.g., "we accept higher write latency in exchange for read scalability") — treat it as one, don't just check that the decision was implemented.

Keep these three levels visibly separate in your output. A story can hit its Success signal while its component quietly misses an NFR budget, or an ADR's predicted cost can show up worse than expected even though every story built against it shipped clean — collapsing these into one verdict per story would hide exactly the kind of finding this skill exists to surface.

## Process

1. Inventory what evidence you actually have — which stories/components/ADRs have real production signal behind them, and which don't. State this inventory before verdicts; an audit that silently skips unaudited stories is misleading by omission.
2. For each story with evidence, compare production reality to its engspec's Success signal. Verdict: **met**, **missed**, or **degraded** (technically met but trending toward missing).
3. For each NFR with evidence, compare the actual measured value to its target. Verdict: **met**, **missed**, with the actual number stated, not just pass/fail.
4. For each ADR with evidence bearing on its Consequences section, check both sides — did the predicted benefit show up, did the predicted cost show up, and if either is surprising, say so plainly (a cost coming in worse than predicted is exactly the finding an ADR's "Agent-Readable Summary" exists to prevent agents from silently re-litigating without knowing why the original decision was made).
5. Draft remediation needs for every missed or badly-degraded item, as described above.
6. Recommend whether `traceability-matrix-generator` should be re-run to fold this report's production-status column into the matrix.

## Output format

```markdown
---
doc_type: outcome_report
schema_version: 1
initiative: <slug>
generated_by: outcome-auditor
source_files: [<engspecs>, <nfrs.md>, <adr files>, <production signal described/provided>]
date: <today, ISO format>
---

# Outcome Report — <initiative>

## Evidence inventory
[What production signal you were actually given, and which stories/NFRs/ADRs it does and doesn't cover.]

## Story-level verdicts (Success signal vs. reality)

| Story | Success signal | Observed | Verdict |
|---|---|---|---|
| STORY-ID | [from engspec] | [actual, with source] | Met / Missed / Degraded / Unaudited (no data) |

## NFR-level verdicts (budget vs. reality)

| NFR | Target | Observed | Verdict |
|---|---|---|---|
| REQ-NFR-ID | [target] | [actual] | Met / Missed / Unaudited |

## ADR-level verdicts (predicted consequence vs. reality)

| ADR | Predicted benefit | Predicted cost | Observed | Verdict |
|---|---|---|---|---|
| ADR-ID | [from Consequences] | [from Consequences] | [actual] | Held / Cost worse than predicted / Benefit didn't materialize / Unaudited |

## Remediation needs (feed back to stakeholder-discovery / requirements-writer)

- **Gap:** [what missed]
  **Traces to:** STORY-ID / ADR-ID
  **New need:** [plain-language statement, ready to paste as input to the next cycle]

## Recommendation
[Whether traceability-matrix-generator should be re-run with this report's findings folded in.]
```
