---
title: PR #6 re-validation with Reasoning + Considered-but-not-flagged transparency
run-date: 2026-05-17
persona-versions: 01 v0.5, 02 v0.3, 03 v0.3, 04 v0.3 (all bumped this session)
input: src/Umbraco.Engage.AI/Tools/GetTopPagesTool.cs @ 97a1ec9 (same as 2026-05-17-orchestrator-pr006-pre-fix-dry-run.md)
runs: 2 (consistency check)
predecessor: eval-runs/2026-05-17-orchestrator-pr006-pre-fix-dry-run.md (original, before transparency update)
purpose: Validate that adding Reasoning + Considered-but-not-flagged surfaces transparency WITHOUT regressing catches OR producing post-hoc rationalization.
---

# PR #6 re-validation — transparency surface

## Setup

Same input as the original PR #6 dry-run. Same context-supplement block. Personas all bumped one minor version (01 v0.4→v0.5; 02 v0.2→v0.3; 03 v0.2→v0.3; 04 v0.2→v0.3) — Andy's plan said "v0.4 / v0.3 / v0.3 / v0.2" but persona #1 was already at v0.4 (Bug §3 fix), so the next available version is v0.5. Honored intent of single-bump-per-persona for the transparency update.

## Run 1 summary

| Persona | Findings | Severities | Reasoning quality | Considered-but-not-flagged | Notable |
|---|---|---|---|---|---|
| Brand-Voice v0.5 | 3 | Block × 3 | Genuine × 3 (cites signal, alternative considered, severity boundary) | 4 items (Pageviews, AvgTimeOnPage, Period arg, PageUrl) | F1 explicitly considers Stone-class alternative; F2 cites description-only-prose-leak subcase from rubric; F3 distinguishes from F2 (not double-counting) |
| Funnel-Stage v0.3 | 2 | Block, Concern | Genuine × 2 (clean source-of-truth check; severity-boundary explicit) | 4 items (AvgTimeOnPage polarity ambiguity, Pageviews uniform, Period/Limit, TopPagesMetric enum folded into F1) | **Explicitly rejected AvgTimeOnPage Q2 polarity** in Considered-but-not-flagged — same dimension persona had flagged as Concern in original dry-run Run 1. v0.3 surfaces this self-correction visibly |
| Hypothesis-Driven v0.3 | 0 (sentinel) | — | N/A | 4 items (4 trigger phrases evaluated and rejected with specific reasons) | Sentinel + considered-but-not-flagged shape — useful for audit despite spec saying "nothing else" after sentinel |
| Data-Trust v0.3 | 2 | Block × 2 | Genuine × 2 (alternative rejection cites lane boundary; severity boundary cites wrong-diagnosis test) | 5 items (AvgTimeOnPage forward-looking Q5, Limit out of lane, Pageviews not at risk, Metric arg out of lane, timezone correctness routing) | F2 timezone = **Block** in Run 1 (was Concern in original dry-run Run 1, Block in original dry-run Run 2 — now stable Block in transparency runs) |

## Run 2 summary

| Persona | Findings | Severities | Reasoning quality | Considered-but-not-flagged | Notable |
|---|---|---|---|---|---|
| Brand-Voice v0.5 | 3 | Block × 3 | Genuine × 2, **Borderline post-hoc × 1** (F1) | 4 items (Pageviews Stone, PageUrl, Period arg, Limit) | F1 reframed around AvgTimeOnPage scope claim with "row guilt by association" reasoning — Reasoning acknowledges alternative ("could AvgTimeOnPage alone be Stone? yes, it's page-scoped"), then rejects on weak grounds ("the field is published as one column of a record whose page-identity is broken for sibling columns"). Hybrid signal: reasoning surface acknowledges the tension but the conclusion is reachier than Run 1's framing |
| Funnel-Stage v0.3 | 2 | Block, Concern | Genuine × 2 | 4 items (handoff well-formed, scope-narrowing correct, Pageviews uniform, pageUrl dimension correct) | **F2 surfaces AvgTimeOnPage via Q6 angle** ("engagement" framing + polarity ambiguity combined). Reasoning explicitly considers + rejects Q2-only framing per v0.3 exclusion, then keeps Q6-only finding. Run 1 rejected this dimension entirely; Run 2 keeps it under Q6 wrapper. **Persona-internal disagreement across runs on whether AvgTimeOnPage is in-lane** |
| Hypothesis-Driven v0.3 | 0 (sentinel) | — | N/A | 4 items (same shape as Run 1 — trigger phrases evaluated and rejected) | Sentinel exact (`No Q7 findings.`) + considered list. Stable |
| Data-Trust v0.3 | 2 | Block × 2 | Genuine × 2 | 4 items (AvgTimeOnPage Q5 forward-looking, Period/Limit/Metric args out of lane, Pageviews not at risk, description prose routing to Brand-Voice) | Same shape as Run 1. F2 timezone Block stable across both runs |

## Reasoning quality scoring

Per the rubric (Genuine / Post-hoc rationalization / Hybrid):

| Run | Total findings | Genuine | Post-hoc | Hybrid |
|---|---|---|---|---|
| Run 1 | 7 (BV 3 + FS 2 + DT 2; HD sentinel) | 7 | 0 | 0 |
| Run 2 | 7 (BV 3 + FS 2 + DT 2; HD sentinel) | 5 | 1 (BV F1 row-guilt-by-association) | 1 (FS F2 Q6-via-engagement-framing surfacing AvgTimeOnPage after Run 1 rejected it) |

**Aggregate: 12 of 14 findings (~86%) produced genuine reasoning across 2 runs.** 1 borderline post-hoc, 1 hybrid. Threshold for ship (per Andy's plan): "mostly genuine across 2 runs" — met. No findings are pure post-hoc rationalization where reasoning paraphrases issue without adding insight.

The post-hoc + hybrid cases both concentrate on the **AvgTimeOnPage borderline catch**:
- Run 1 BV rejected it cleanly (Considered-but-not-flagged)
- Run 1 FS rejected it cleanly (Considered-but-not-flagged, citing v0.3 explicit rule)
- Run 2 BV surfaced it via "row scope" reasoning (post-hoc)
- Run 2 FS surfaced it via "engagement framing" Q6 angle (hybrid — acknowledges Q2 exclusion then routes around it)

This is the kind of borderline drift the Bug §3 framework already documents as "intrinsic LLM variance on severity calls." Reasoning surface makes the variance visible instead of hiding it — that's the goal of transparency, even if the variance itself doesn't disappear.

## Considered-but-not-flagged quality

All 4 personas emitted this section in both runs (8/8). Items consistently:

- **Cite specific elements** (named field, named arg, named description phrase)
- **Give one-line reasons** rooted in lane definition or framework rules
- **Avoid vague rejections** like "didn't seem important" or "out of scope" without elaboration

Notable example from Funnel-Stage Run 1:
> *"`AvgTimeOnPage` — borderline polarity ambiguity (high time = deep engagement OR high friction / confusion), but no per-instance polarity flag exists on a page entity in Engage Core. This is a content-design ambiguity, not a Q2 wire-format omission; the source data has no `IsInverted`-equivalent to surface."*

This is the kind of audit signal Andy wanted — it documents WHY a finding was rejected with enough context for Andy to spot if the rejection logic is wrong.

Hypothesis-Driven Considered-but-not-flagged after a sentinel is the most surprising win — sentinel persona normally returns "No Q7 findings." and stops, but the considered section now shows the trigger phrases the persona evaluated (`"ranking content by performance"`, `"which pages drive engagement"`) with reasons for rejection. Useful audit trail even when no findings fire.

## Output bloat measurement

Word counts per finding (Issue + Recommendation + Reasoning + structured metadata):

| Source | Avg words / finding | Bloat vs original |
|---|---|---|
| Original PR #6 dry-run findings | ~240 words | baseline |
| Transparency findings (Reasoning added) | ~330 words | **+38%** |

Plus Considered-but-not-flagged section: ~100-180 words per persona output (5-8 items × ~20 words each).

**Total output growth: ~40-50% per persona response.** Trade-off: pay 40% bloat for genuine reasoning surface + audit trail. Acceptable for v0.5+; revisit if real-PR usage shows the bloat slows down panel-review reading.

## Catch regression check (vs original PR #6 dry-run)

| Ground truth | Original Run 1 | Original Run 2 | Reasoning Run 1 | Reasoning Run 2 |
|---|---|---|---|---|
| GT-1 metric routing (sessions/users on per-page) | ✅ multi-persona Block | ✅ multi-persona Block | ✅ multi-persona Block | ✅ multi-persona Block |
| GT-2b timezone-aware boundaries | ✅ DT Concern | ✅ DT Block | ✅ DT Block | ✅ DT Block |

**Zero catch regression.** All in-scope ground truth still caught with Block severity. Timezone catch now stable at Block across both transparency runs (no Concern/Block drift like the original cycles).

## Two-run drift on Reasoning

| Finding | Reasoning Run 1 vs Run 2 | Drift |
|---|---|---|
| BV Sessions/Users column claim | Different anchor (sessions vs row-as-whole) | **Moderate** — finding shape drifted, reasoning followed |
| BV description prose leak | Same core (description-only-prose-leak subcase from rubric) | Low |
| BV TopPagesMetric enum | Same core (Stone claim via enum name) | Low |
| FS Sessions/Users misrouting | Same Q6 attributional-misrouting reasoning | Low |
| FS handoff dangling pointer | Run 1 = Q6 cross-tool handoff. Run 2 = Q6 "engagement" + Q2 polarity combined | **Moderate** — reasoning anchor shifted |
| DT Sessions/Users data-source fidelity | Same Q1 + Q5 reasoning | Low |
| DT timezone | Same Q1 data-source-fidelity-overclaim reasoning | Low |

**Overall: Low-to-moderate drift.** Reasoning fields stay similar where the finding shape stays similar; reasoning drifts when finding shape drifts (which makes sense — reasoning should track the finding, not be independent of it). No evidence of reasoning being generated independently of findings (which would suggest post-hoc rationalization). 

## Verdict

**Ship persona transparency update (v0.5 / v0.3 / v0.3 / v0.3).**

- **Reasoning quality:** 12/14 genuine across 2 runs (~86%). Borderline + hybrid both on AvgTimeOnPage edge case, consistent with intrinsic LLM variance on rubric boundaries.
- **Catch regression:** Zero — same ground-truth catches as original PR #6 dry-run.
- **Two-run consistency on Reasoning:** Low-to-moderate drift; reasoning tracks finding shape (good signal).
- **Output bloat:** ~40-50% per persona response. Acceptable trade-off for transparency.
- **Notable wins:**
  - Funnel-Stage Run 1 explicitly rejected AvgTimeOnPage polarity in Considered-but-not-flagged — visible self-correction.
  - Hypothesis-Driven sentinel + considered section provides audit trail even when no findings fire.
  - All Considered-but-not-flagged items cite specific elements with specific reasons.

**Phase 2.5 backlog item raised by this run:** AvgTimeOnPage continues to drift across persona / run combinations as a borderline catch. Worth a rubric tightening in v0.6 — explicit decision rule for ambiguous polarity on page-level metrics: "If source entity has no per-instance polarity flag, do not flag Q2 even via cross-dimension framing." That eliminates the FS Run 2 Q6-as-Q2-route path. Track recurrence rate before fixing — single edge case in 4 persona-runs is below mitigation threshold.

Proceed to commit + push.
