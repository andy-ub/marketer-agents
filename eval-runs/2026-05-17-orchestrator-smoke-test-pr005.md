---
title: Orchestrator smoke test — PR #5 pre-fix
run-date: 2026-05-17
orchestrator-version: v0.1 (Phase 2 implementation)
input: src/Umbraco.Engage.AI/Tools/GetGoalsTool.cs @ d483e05^ (pre-fix code)
input-form: local (no PR-ref) — origin-stamping skipped per design (gh CLI absent + no PR metadata)
runs: 2 (consistency check per Phase 3 plan)
total-dispatches: 8 Agent calls (4 personas × 2 runs)
---

# Orchestrator smoke test — PR #5 pre-fix

First end-to-end run of the orchestrator runbook ([`orchestrator/orchestrator-prompt.md`](../orchestrator/orchestrator-prompt.md)) against real input. Goal: validate baseline functionality + capture observations on the smoke-test attention points Andy flagged in Phase 2 approval.

---

# Run 1 — Panel review (canonical output)

```markdown
# Panel Review — local-diff (PR #5 pre-fix GetGoalsTool.cs) — 2026-05-17

## Summary

- Total findings: 7 raw across 3 of 4 personas (Brand-Voice 2, Funnel-Stage 2, Data-Trust 3; Hypothesis-Driven sentinel).
- Panel-level severity counts: Block × 4, Concern × 3, Nit × 0.
- AI-reviewer-suggested findings: 0 stamped (origin-stamping skipped — gh CLI unavailable on this machine; all findings tagged `origin: unknown — gh CLI not installed`).
- No-findings sentinel personas: [hypothesis-driven]
- Personas that failed to respond: [] (all 4 succeeded; durations 8–44 seconds)
- Semantic merge groups: 5 (after manual correction; strict token-overlap heuristic over-merged — see Run health + Bugs)

## Findings

### Finding 1: MonetaryValue rename promotes configurable decimal to currency
*Semantic merge group MV — 1 Block + 1 Block + 1 Concern across {brand-voice, data-trust}*

- **Panel severity:** Block
- **Persona severity:** Block (from brand-voice)
- **Compound flags:** Block + Concern compounded from {brand-voice, data-trust}; brand-voice Finding 2 (Block, description prose) folded into Recommendation
- **Multi-persona agreement:** 2 personas flagged related Evidence ({brand-voice, data-trust})
- **Origin:** unknown — gh CLI not installed
- **Evidence:** `src/Umbraco.Engage.AI/Tools/GetGoalsTool.cs` — `decimal MonetaryValue` (field on `GoalResult`, populated from `goal.Value`)
- **Framework citation:** Q3 generic-vs-imposed semantic, Q1(c) Stone-vs-Opinion (Opinion treated as Stone)
- **Issue:** This rename teaches the LLM to call configurable amounts "money." Marketers configure `Goal.Value` as a mix of currency, counts, scores, weights, and zero-for-tracking — when the LLM sums the goals and reports "your goals represent $4,820 in tracked pipeline," it is fabricating dollar arithmetic against numbers that aren't dollars. A marketing lead will plan spend, forecast revenue, or pitch the CFO against a total that the data layer never promised was currency.
- **Recommendation:** Rename the field back to `Value` on the wire format so the LLM has no lexical handhold for a currency claim. Add to the tool `Description`: *"Value is a marketer-configured decimal whose meaning depends on each goal's setup — it may represent a monetary amount, a count, a score, a weight, or zero for tracking-only goals. Do not assume currency, and do not sum Values across goals as if they share units."* Keep the underlying `decimal` type as-is; the data layer is honest, only the wire-format name was lying.
  *Related concerns from brand-voice (Finding 2, Block):* Replace "monetary value" in the description with neutral language that names the field and surfaces its configurability — see proposed wording in the related-finding original.
  *Related concerns from data-trust (Finding 3, Concern, Q1 data-source fidelity):* If the product genuinely intends monetary semantics, surface a sibling `ValueKind` enum (`Monetary` / `Score` / `Weight` / `Count` / `Unspecified`) so the LLM can refuse to sum across kinds.

### Finding 2: GoalResult drops IsInverted — LLM will celebrate failure modes as top performers

- **Panel severity:** Block
- **Persona severity:** Block (from funnel-stage)
- **Compound flags:** single-persona finding
- **Multi-persona agreement:** 1 persona flagged this Evidence ({funnel-stage})
- **Origin:** unknown — gh CLI not installed
- **Evidence:** `src/Umbraco.Engage.AI/Tools/GetGoalsTool.cs:44-50` — `GoalResult(Guid Key, string Name, string GoalType, decimal MonetaryValue, bool IsMain, bool IsActive)` (no `IsInverted`); source-of-truth: `IGoal.IsInverted` in `Umbraco.Engage.Infrastructure.Analytics.Goals`.
- **Framework citation:** Q2 polarity semantic
- **Issue:** A marketer asking "which goals are performing best this quarter?" will be told their Cart Abandoned goal is a top performer with however many hundreds of completions it logged — the LLM has no flag on the wire to tell it that goal counts a failure mode, so it reads the count as an upward arrow. Every inverted goal in the configuration (abandonment, bounce, form-started-not-submitted) becomes a celebration candidate the moment its count outpaces a normal goal. The direction of the arrow is inverted at the entity level in Engage Core, and that direction is being silently dropped on the way to the LLM.
- **Recommendation:** Add `IsInverted` (bool) to `GoalResult` and populate from `goal.IsInverted`. Then add a sentence to the tool description that teaches the LLM how to read it: *"IsInverted=true means the goal counts a negative outcome (e.g., Cart Abandoned, Bounced, Form Started but Not Submitted). For inverted goals, a high completion count is BAD, not good — interpret count direction accordingly and never rank them alongside normal goals as 'top performers'."* Without both the field and the description sentence, the LLM stays polarity-blind.

### Finding 3: GoalResult strips IsInvalid even though IGoalService does not auto-filter

- **Panel severity:** Block
- **Persona severity:** Block (from data-trust)
- **Compound flags:** single-persona finding
- **Multi-persona agreement:** 1 persona flagged this Evidence ({data-trust})
- **Origin:** unknown — gh CLI not installed
- **Evidence:** `src/Umbraco.Engage.AI/Tools/GetGoalsTool.cs:42-48` — `public record GoalResult(...)` (omits `IsInvalid`); source: `IGoal.IsInvalid` in `Umbraco.Engage.Infrastructure.Analytics.Goals`; contrast: `CampaignGroupResult` correctly omits because `ICampaignGroupService.GetAll()` auto-filters.
- **Framework citation:** Q1 repository filter policy
- **Issue:** A marketer asking "why is Newsletter Signup sitting at zero completions?" will be told the goal is configured, active, and just under-performing — when in reality the Form node it was bound to was deleted weeks ago and the goal has been physically unable to fire ever since. The wire format drops `IsInvalid` before it reaches the LLM, so the only signal that the configuration is structurally broken never makes it into the narration; `IsActive=true` on a goal that cannot fire reads as healthy. The implementer copy-pasted the `CampaignGroupResult` shape, but `ICampaignGroupService.GetAll()` auto-filters invalid groups at query level and `IGoalService.GetAll()` does not — the field that was correctly redundant for Campaigns is load-bearing for Goals. The marketer spends two weeks tweaking copy and CTAs against a goal that is incapable of recording an event.
- **Recommendation:** Add `bool IsInvalid` to `GoalResult` and project it from `goal.IsInvalid` in the `Select`. Add a sentence to the tool `Description` along the lines of: *"IsInvalid=true means the goal's tracked configuration is broken (e.g., it references a deleted Form node or removed page); the goal cannot fire even when IsActive=true, so a zero-completion reading on an IsInvalid goal is a configuration problem, not a performance problem."* Per-entity verification of repository filter behavior should be the rule going forward — do not carry the Campaigns omission across to Segments or any other entity without re-checking the upstream `GetAll()` semantics.

### Finding 4: Tool description never declares scope or hands off adjacent stages

- **Panel severity:** Concern
- **Persona severity:** Concern (from funnel-stage)
- **Compound flags:** single-persona finding
- **Multi-persona agreement:** 1 persona flagged this Evidence ({funnel-stage})
- **Origin:** unknown — gh CLI not installed
- **Evidence:** `src/Umbraco.Engage.AI/Tools/GetGoalsTool.cs:24-30` — *"Does NOT include per-goal completion counts — a separate tool covers goal performance metrics. Does NOT explain the trigger criteria for a goal — a separate tool covers goal rule explanations."*
- **Framework citation:** Q6 funnel-stage attribution, Q6 cross-tool handoff
- **Issue:** A marketer asking "why are our signups down this week?" or "which campaigns are driving the most conversions?" will get the LLM rummaging in this tool's output and narrating campaign-stage or performance-stage answers from a configuration-stage surface — because the description never says this tool measures *goal configuration*, not goal performance or campaign attribution. The two handoff hints that are present ("a separate tool covers …") are dangling pointers — they name no tool, so the LLM cannot actually route the marketer. The result is a tool that fences off what it doesn't do but never tells the LLM where to send the marketer instead.
- **Recommendation:** Pin the scope explicitly in sentence 1 of the description; then name the adjacent tools by ID in the handoff sentences (e.g. `engage_get_goal_performance`, `engage_explain_goal_rule`, `engage_get_campaigns`). Named handoffs turn a fence into a router.

### Finding 5: GoalType resolution conflates tracking-health gap with "no type"

- **Panel severity:** Concern
- **Persona severity:** Concern (from data-trust)
- **Compound flags:** single-persona finding
- **Multi-persona agreement:** 1 persona flagged this Evidence ({data-trust})
- **Origin:** unknown — gh CLI not installed
- **Evidence:** `src/Umbraco.Engage.AI/Tools/GetGoalsTool.cs:23-28` — *"If GoalType is empty, the registered goal type could not be resolved (the addon that registered the type may no longer be loaded)."* paired with `GoalType: _goalTypeFactory.GetGoalType(goal.GoalTypeId)?.Name ?? string.Empty` at line 36.
- **Framework citation:** Q1 data-source fidelity
- **Issue:** A marketer asking "what KPIs are we tracking?" will be handed a list where some rows say `GoalType: ""` and the LLM will narrate that as "this goal has no type set" or quietly skip it — when the actual state is that an entire addon has been unloaded and a whole class of goals is silently dark across the install. The description does flag the resolution failure in one line, but it conflates a tracking-health failure ("addon gone") with a benign empty value via the same `string.Empty` sentinel, and gives the LLM no structural signal to escalate the difference.
- **Recommendation:** Split the resolution signal from the value. Either add `bool IsGoalTypeResolved` (or `string? GoalType` nullable plus a sibling `bool GoalTypeAddonMissing`) so the LLM has a structural flag to narrate against, or emit a distinct sentinel like `"<unresolved>"` that the description binds to a specific tracking-health meaning.

## Context-gap routing

| Case | Source persona | Target | Resolution |
|---|---|---|---|
| C out-of-lane | funnel-stage | Q3 generic-vs-imposed on MonetaryValue | merged — brand-voice Finding 1 already owns this Evidence |
| C out-of-lane | funnel-stage | Q1 filter/IsInvalid on GoalResult | merged — data-trust Finding 1 already owns this Evidence |
| C out-of-lane | data-trust | Q3 on MonetaryValue | merged — brand-voice Finding 1 already owns this Evidence |

## Run health

- Persona dispatch:
  - brand-voice: ok (24s)
  - funnel-stage: ok (33s)
  - hypothesis-driven: ok (8s, sentinel)
  - data-trust: ok (44s)
- Origin-stamping: skipped (no PR ref and gh CLI unavailable)
- Coverage holes logged this run: 0
- **Manual override applied:** semantic merge groups corrected from strict token-overlap heuristic output (see Bugs §1 below).
```

---

# Run 2 — Consistency check

Same input, same dispatch template, same 4 personas. Dispatched independently 5 minutes after Run 1.

```markdown
# Panel Review — local-diff (PR #5 pre-fix GetGoalsTool.cs) — 2026-05-17 (Run 2)

## Summary

- Total findings: 7 raw across 3 of 4 personas (Brand-Voice 2, Funnel-Stage 2, Data-Trust 3; Hypothesis-Driven sentinel).
- Panel-level severity counts: Block × 5, Concern × 2, Nit × 0. **(Run 1 was Block × 4 / Concern × 3 — drift on 1 finding.)**
- AI-reviewer-suggested findings: 0 stamped (origin-stamping skipped — same as Run 1).
- No-findings sentinel personas: [hypothesis-driven]
- Personas that failed to respond: []
- Semantic merge groups: 5

## Findings (Run 2 deltas vs Run 1 marked **DRIFT**)

1. **Finding 1 (MonetaryValue rename):** unchanged from Run 1 — Block, same Evidence, similar voice.
2. **Finding 2 (Description "monetary value"):** **DRIFT — Run 2 brand-voice marked this Concern**, Run 1 marked it Block. Same Evidence, same persona, same dispatch input. Standalone severity reading produced different calls on identical input. See Consistency analysis below.
3. **Finding 3 (IsInverted missing):** unchanged — Block.
4. **Finding 4 (Handoff missing):** unchanged — Concern.
5. **Finding 5 (IsInvalid missing):** unchanged — Block.
6. **Finding 6 (Data-Trust on IsInverted via Q1 data-source fidelity):** **DRIFT — Run 2 Data-Trust produced an out-of-lane Q2 finding** wrapped in "Q1 data-source fidelity — validity-adjacent" framing. Run 1 Data-Trust did NOT produce this finding (its Finding 2 was about GoalType resolution, a different topic entirely). See Consistency analysis below for lane-drift discussion.
7. **Finding 7 (MonetaryValue overclaims, Data-Trust):** unchanged — Concern, Q1 data-source fidelity. Same finding as Run 1 Finding 3 (`Q1 data-source fidelity` on MonetaryValue), independently produced.

## Context-gap routing (Run 2)

| Case | Source persona | Target | Resolution |
|---|---|---|---|
| C out-of-lane | brand-voice | Q2 IsInverted + Q1 IsInvalid | **NEW in Run 2** — Brand-Voice escalated where it didn't in Run 1; merged into Funnel-Stage F1 + Data-Trust F1 |
| C out-of-lane | funnel-stage | Q1 IsInvalid on GoalResult | merged — data-trust F1 |
| A overlap | funnel-stage | Q3 generic-vs-imposed on MonetaryValue | merged — brand-voice F1 (case A vs C drift — Run 1 marked this Case C; Run 2 marked it Case A) |
| C out-of-lane | data-trust | Q3 on MonetaryValue | merged — brand-voice F1 |

## Run health (Run 2)

- brand-voice: ok (24s); funnel-stage: ok (30s); hypothesis-driven: ok (8s, sentinel); data-trust: ok (42s)
- Origin-stamping: skipped (same as Run 1)
```

---

# Consistency analysis (smoke-test attention point #1)

Run 1 vs Run 2 on identical input produced **3 substantive drifts**:

| Drift | Run 1 | Run 2 | Severity of drift |
|---|---|---|---|
| Brand-Voice Finding 2 severity | Block | Concern | **Material** — Block/Concern boundary is the marketer's "should-I-stop-the-ship" threshold |
| Data-Trust Finding 2 topic | GoalType resolution (Q1 data-source fidelity) | IsInverted polarity (Q1 data-source fidelity "validity-adjacent") | **Material + lane drift** — Run 2 stretched Q1 to cover Q2's lane |
| Brand-Voice Context-gap escalation | none | Case C escalations for Q2 + Q1 | **Minor** — escalation behavior |

**Non-drifts (consistency held):**

- All 3 core PR #5 findings (MonetaryValue, IsInverted, IsInvalid) caught in both runs with same severity (Block) on the primary finding.
- Hypothesis-Driven returned identical sentinel both runs (8s exact).
- Funnel-Stage findings identical across runs (Block IsInverted + Concern handoff).
- Voice quality consistent — both runs lead with consequence-first marketer narration.

**Verdict on consistency:** the core panel signal (catch the 3 PR #5 failure modes with Block severity) is stable. Drift exists on secondary findings (BV Finding 2 severity, DT Finding 2 topic). This confirms Andy's Phase 2 concern #1 ("in-session execution architecture risks consistency drift") — drift is real but contained to lower-confidence secondary findings.

**Mitigation candidates for Phase 3 backlog:**
- Tighten Brand-Voice Concern definition in severity rubric so the description-prose case unambiguously calls one of (Concern | Block) given a standalone reading.
- Tighten Data-Trust Q1 data-source-fidelity lane — explicit "validity-adjacent" exclusion to prevent the Q2 stretch in Run 2.
- Run-to-run variance is intrinsic to LLM sampling and cannot be fully eliminated by prompt tightening; tracking drift rate over time (across 10+ orchestrator runs) is the eventual statistical answer.

---

# Bugs encountered

## Bug §1 — Token-overlap heuristic over-merges (smoke-test attention #2 — CONFIRMED)

The Phase 2 token-overlap heuristic (`evidence_file equal AND ≥1 shared identifier token`) over-merges in this run.

Concrete misbehavior: findings on `MonetaryValue`, `IsInverted`, and `IsInvalid` all cite `GoalResult` in their Evidence strings (each cites the record declaration line range as context for an omission-class finding). They share the token `goalresult`. Strict heuristic merges all three into one group. **Semantically, they are three independent findings on three different fields.**

Frequency: 3 over-merge pairs in Run 1, 3-4 in Run 2 (depending on Brand-Voice's context-gap behavior). **Per-run over-merge rate: ~3 false-positive merges per real Engage tool diff.**

**Mitigation deployed for this run:** manual semantic correction of merge groups. The "Findings" sections above use the corrected merge groups, not strict heuristic output.

**Recommended fix (Phase 2.5 work, not v0.1 blocker):**

- **Option A — require ≥2 shared tokens AFTER additional stopword filtering.** Add structural common-words to stopword list: `goalresult`, `campaignresult`, `record`, `class`, `public`, etc. — words that are common to record declarations but don't identify specific fields. With ≥2 shared tokens and tighter stopwords, MonetaryValue/IsInverted/IsInvalid no longer merge (each shares only `goalresult` in common, which gets stopword-filtered out).
- **Option B — weight tokens by specificity.** A token appearing in only 1 finding's Evidence is "specific" (weight 1.0); a token appearing in N findings has weight 1/N. Merge if shared-weight-sum > threshold (e.g. 1.5). This naturally penalizes the `goalresult` token because it appears in all 3 omission findings.
- **Option C — split Evidence into "anchor symbol" vs "context location":** require the anchor symbol (the specific field/phrase being flagged) to match, not the whole Evidence string. Persona prompts would need a small format tweak — currently Evidence mixes both.

**Recommend A as cheapest fix** for Phase 2.5. Test with this same smoke-test input after applying.

## Bug §2 — Data-Trust lane drift in Run 2 (smoke-test attention point — NEW)

Run 2 Data-Trust Finding 2 framed IsInverted (Q2 polarity, Funnel-Stage's lane) as a Q1 data-source-fidelity finding ("validity-adjacent semantic the LLM cannot trust the count without"). This is genuine lane stretching: the persona used the Q1(v1.1)-extension "data-source fidelity" sub-clause to claim coverage of a polarity question.

**Why it happened:** the persona prompt §"Q1 also covers (per audit rule v1.1 extensions)" allows broad Q1 reading; the persona stretched it. Run 1 did not stretch (Run 1 produced a clean GoalType-resolution finding for its Finding 2 slot). This is a sampling-variance lane drift, not a prompt bug per se.

**Why it matters:** Funnel-Stage F1 already owned this issue with the correct Q2 framework citation. Data-Trust's Q1-framed take is a duplicate finding on the same Evidence with a wrong framework citation. In the merged panel output it would correctly fold (Block + Block on same Evidence) but the framework-citation provenance becomes ambiguous.

**Recommended fix (Phase 2.5):** tighten Data-Trust persona's "you do NOT review for" list with an explicit "Q2 polarity (even when framed as 'validity-adjacent data-source fidelity')" exclusion. One-line edit.

## Bug §3 — Strict heuristic vs semantic merge gap requires manual override

The Phase 2 design said the orchestrator's merge step is mechanical (token-overlap). In practice, Run 1's correct merge required semantic understanding the heuristic does not capture. The smoke-test output uses manual override; this is acceptable for v0.1 but means the orchestrator is not fully automatic until Bug §1 is fixed.

**Phase 2.5 priority:** Bug §1 fix is the gating item before the orchestrator can run unattended.

---

# Token-overlap accuracy analysis (smoke-test attention #2)

Per Phase 2 concern #2 verbatim: *"how often does the heuristic group findings that a human reviewer would treat as independent?"*

Measurements from Run 1 (7 findings, 21 pairs):

| Pair | Strict heuristic | Semantic ground truth | Match |
|---|---|---|---|
| BV-F1 ↔ BV-F2 (MonetaryValue field + description) | related | related | ✓ |
| BV-F1 ↔ FS-F1 (MonetaryValue + IsInverted) | related (`goalresult`, `monetary`, `value`) | **independent** | ✗ over-merge |
| BV-F1 ↔ FS-F2 (MonetaryValue + handoff) | related (`description`, `goal`) | **independent** | ✗ over-merge |
| BV-F1 ↔ DT-F1 (MonetaryValue + IsInvalid) | related (`goalresult`, `monetaryvalue`) | **independent** | ✗ over-merge |
| BV-F1 ↔ DT-F2 (MonetaryValue + GoalType) | related (`goal`) | **independent** | ✗ over-merge |
| BV-F1 ↔ DT-F3 (MonetaryValue + MonetaryValue overclaims) | related | related | ✓ |
| BV-F2 ↔ FS-F1 | related (`goal`) | **independent** | ✗ over-merge |
| BV-F2 ↔ FS-F2 | related (`goal`) | **independent** | ✗ over-merge |
| BV-F2 ↔ DT-F1 (description + IsInvalid) | related (`goal`) | **independent** | ✗ over-merge |
| BV-F2 ↔ DT-F2 (description + GoalType) | related (`goaltype`, `goal`) | **independent** | ✗ over-merge |
| BV-F2 ↔ DT-F3 | related (`monetary`, `value`) | related | ✓ |
| FS-F1 ↔ FS-F2 | related (`goal`) | independent (same persona, different lanes) | ✗ over-merge |
| FS-F1 ↔ DT-F1 | related (`goalresult`, `igoal`) | **independent** | ✗ over-merge |
| FS-F1 ↔ DT-F2 | related (`goal`) | **independent** | ✗ over-merge |
| FS-F1 ↔ DT-F3 | related (`goalresult`, `monetaryvalue`) | **independent** | ✗ over-merge |
| FS-F2 ↔ DT-F1 | related (`goalresult`) | **independent** | ✗ over-merge |
| FS-F2 ↔ DT-F2 | related (`goaltype`) | **independent** | ✗ over-merge |
| FS-F2 ↔ DT-F3 | related (`goalresult`) | **independent** | ✗ over-merge |
| DT-F1 ↔ DT-F2 | related (`goal`) | independent (same persona) | ✗ over-merge |
| DT-F1 ↔ DT-F3 | related (`goalresult`, `monetaryvalue`) | **independent** | ✗ over-merge |
| DT-F2 ↔ DT-F3 | related (`goal`) | independent (same persona) | ✗ over-merge |

**Accuracy: 3 correct / 21 pairs ≈ 14% precision; 100% recall** (no semantic merges missed).

This is well above Andy's tolerance threshold (`>1 case per smoke-test PR → tighten`). Bug §1 fix is required before the orchestrator can run unattended on real PRs.

The good news: precision can be fixed cheaply (Option A above) without retraining anything — it's a stopword + threshold tweak.

---

# Smoke-test attention point measurements

| Attention point | Measurement | Action |
|---|---|---|
| #1 Semantic-overlap-without-token-overlap (false NEGATIVES) | 0 cases in 1 run | Hold for more runs |
| #2 Token-overlap over-merge (false POSITIVES) | 18 / 21 pairs over-merge | **Fix in Phase 2.5 (Bug §1)** |
| #3 Origin-stamping precision | not measured (gh CLI absent) | Run after gh CLI install |
| Case C noise rate | Run 1: 3 Case C / 1 run. Run 2: 3 Case C + 1 Case A / 1 run. | Within ≤2/run threshold per persona prompt guidance; acceptable for v0.1 |
| Run-to-run drift rate | 3 drifts on 7 findings (≈43% per finding has some drift); 0/3 drift on core PR #5 catches | Acceptable for v0.1 — secondary findings drift, core catches stable |

---

# Phase 3 verdict

**Smoke test outcome: orchestrator baseline functionality works.** Core PR #5 catches landed in both runs at Block severity. Voice quality preserved. Sentinel personas (Hypothesis-Driven) handled correctly. Context-gap escalations parsed correctly.

**Blocking issue: Bug §1 (token-overlap over-merge).** The orchestrator cannot run unattended until this is fixed — without manual semantic correction, the merged panel output would conflate unrelated findings on different fields just because they cite the same record. Fix is cheap (Option A stopword + threshold tweak).

**Phase 2.5 backlog (priority order):**

1. **[BLOCKER]** Bug §1 fix — extend stopword list with structural common-words; require ≥2 shared tokens. Re-run this smoke test to verify accuracy improves.
2. **[medium]** Bug §2 — Data-Trust persona "you do NOT review for" list add Q2-polarity exclusion to prevent lane stretch via Q1 data-source-fidelity sub-clause.
3. **[medium]** Brand-Voice Finding 2 (description prose) severity calibration — sharpen rubric so Block/Concern doesn't drift between runs on identical input.
4. **[low]** Install `gh` CLI when origin-stamping value becomes load-bearing. Until then all findings tag `origin: unknown`.

**Phase 4 backlog (further out):**

5. Run-to-run variance instrumentation — track drift across 10+ orchestrator runs to set a statistical baseline.
6. CI integration wrapper — if orchestrator needs to run outside Claude Code, build a CLI wrapper.
7. Q5 eval lift if/when AttributionType lands in Engage tools (trigger criterion in `notes/orchestrator-design-notes.md` §5).
