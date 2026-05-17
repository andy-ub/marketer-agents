---
title: Orchestrator design (Phase 1)
status: design — Phase 2 implementation pending Andy review
created: 2026-05-16
inputs:
  - notes/orchestrator-design-notes.md (5 deferred decisions from persona drafting)
  - eval-runs/2026-05-16-panel-final-review-prompt.md (final-review architectural risks)
  - personas/01..04 final versions
relates-to: README.md "Panel-level final review" line
---

# Orchestrator design — Phase 1

Resolves the 3 architectural risks the panel final-review flagged as carry-forward to orchestrator phase. Each section: chosen default + reasoning + one explicit edge case the default handles + one explicit case the default does NOT handle (escalation path documented). Refinement deferred to smoke-test data after first 3-5 real PR runs.

---

## Risk 1 — Merge mechanics

### 1a. Related-Evidence definition

**Default: token-overlap heuristic on Evidence anchors.**

Extract identifier tokens from backticks-quoted content in each finding's `Evidence` field (e.g. `Goal.Value`, `MonetaryValue`, `IsInverted`, `IGoalService.GetAll`). Two findings are **related Evidence** if:

1. They cite the same `<file>` AND
2. Their identifier token sets share ≥1 token (`Goal.Value` matches `Goal.Value`; `MonetaryValue` matches `MonetaryValue`; `Description` alone does NOT match unless paired with a specific phrase-token like `monetary-value` parsed from the prose).

For prose-citation findings (whole sentence quoted from `Description`), tokenize the quoted substring on whitespace + camelCase boundaries; same ≥1 shared token rule. Length-normalize to avoid noise — drop stopwords (`is`, `the`, `a`, `for`, `with`, etc.) and tokens shorter than 3 chars before comparison.

**Reasoning.** Exact file:line is too strict (Brand-Voice Finding 1 on `MonetaryValue` cites line 39, Data-Trust Finding 1 on the same line cites lines 46-52 for the record declaration — both are about the same field, but line ranges don't overlap). Symbol-name match alone misses prose findings. Framework-citation-pair is orthogonal to Evidence and conflates different concerns. Token-overlap on Evidence anchors captures the actual semantic the personas were pointing at.

**Edge case it handles.** Brand-Voice Finding 2 (description prose: "monetary value") and Funnel-Stage Finding 2 (description prose: "a separate tool covers…") cite the same `Description` property but different phrases within it. Token-overlap correctly identifies them as **NOT related** because the phrase-tokens differ. They are two distinct concerns in the same property.

**Edge case it does NOT handle.** Semantic overlap without token overlap — e.g., Brand-Voice flags `MonetaryValue` rename (token: `MonetaryValue`); Funnel-Stage flags the same field's missing polarity context (token: `IsInverted`); both reference the same `GoalResult` record but different fields, different tokens. These are not "the same finding" — they're independent. Default correctly leaves them as separate findings. If they ever need to be merged, that's a panel-summary clustering job, not a related-Evidence merge job.

---

### 1b. Concern → Block elevation threshold

**Default: N=2 Concerns from DISTINCT personas on related Evidence → panel-level Block flag.**

Concrete rules:

| Configuration | Panel-level severity |
|---|---|
| 1 Block on Evidence | Block (no change) |
| Block + Concern(s) on related Evidence | Block; Concern's Recommendation folded into Block's remediation plan; emit `compound-severity: Block + Concern from {persona-X, persona-Y}` flag |
| 2+ Concerns from distinct personas on related Evidence | **Panel-level Block** (elevation); preserve original per-finding Concern severity; emit `compound-severity: 2× Concern from {persona-X, persona-Y} elevated to Block` flag |
| 2+ Concerns from the SAME persona on related Evidence | No elevation (persona had full visibility of its own findings; if it called both Concerns, that is the per-finding standalone severity holding) |
| 1 Concern alone on Evidence | Concern (no change) |
| N Nits | No elevation rule for Nits |

**Reasoning.** N=2 from distinct personas mirrors the multi-persona agreement signal already documented in orchestrator-design-notes §1 — two independent perspectives reaching Concern on the same Evidence is structurally equivalent to "high-confidence Concern," which on related Evidence compounds into a fabricated-claim risk the marketer will act on. Per-finding Concerns from the same persona do NOT compound: this would re-introduce the cross-finding reasoning the persona prompts explicitly forbid.

**Edge case it handles.** Brand-Voice Concern on `Description: "monetary value"` prose leak + Data-Trust Concern on `Description: data-source fidelity` — both on the same description property, different phrases. Token-overlap rules them NOT related (different phrase-tokens). No elevation. Correct — these are two independent concerns about the same description, neither alone elevates.

**Edge case it does NOT handle.** Cross-persona Concerns on **adjacent but technically distinct** Evidence (e.g., Concern on field X + Concern on field Y where X and Y interact). The default does not auto-elevate these because token-overlap fails. The orchestrator output should preserve both Concerns visibly; human reviewer (or panel-summary clustering in a future iteration) can recognize the pattern. Documented as Phase 2 refinement target.

---

### 1c. Output shape

**Default: structured markdown panel review with 4 sections.**

```markdown
# Panel Review — <PR title / SHA / date>

## Summary
- Total findings: <N> across <K> personas (out of 4)
- Panel-level severity: Block × <count>, Concern × <count>, Nit × <count>
- AI-reviewer-suggested findings: <count> (high-leverage — origin stamped from PR review history)
- No-findings sentinel personas: [<list>]
- Personas that failed to respond: [<list>]   (timeout / malformed — see §4 Run health)

## Findings (sorted: Block first, then by multi-persona agreement, then by persona order)

### Finding <id>: <persona's short title>

- **Panel severity:** Block | Concern | Nit
- **Persona severity:** Block | Concern | Nit (from <persona-id>)
- **Compound flags:** <e.g. "Block + Concern compounded from {brand-voice, data-trust}" | "2× Concern elevated to Block" | "single-persona finding">
- **Multi-persona agreement:** <count> personas flagged related Evidence ({<persona-ids>})
- **Origin:** implementer-authored | ai-reviewer-suggested | human-reviewer-suggested | unknown
- **Evidence:** <verbatim from persona>
- **Framework citation:** <verbatim from persona>
- **Issue:** <verbatim from persona>
- **Recommendation:** <verbatim from persona; if compound, append "Related concerns from <other-persona>: <Recommendation>">

## Context-gap routing

| Case | Source persona | Target dimension | Resolution |
|---|---|---|---|
| A overlap | brand-voice | Q5 | merged into Data-Trust Finding <id> |
| B need-context | funnel-stage | Q6 sibling-tool description | unresolved — flagged for human reviewer |
| C out-of-lane | hypothesis-driven | Q4 decision-class | coverage hole (no panel owner) — see §4 |

## Run health

- Persona dispatch: brand-voice ✓ (24s), funnel-stage ✓ (31s), hypothesis-driven ✓ (8s, sentinel), data-trust ✓ (33s)
- Origin-stamping: <ran / skipped — reason>
- Coverage holes (Case C routed to no-owner dimension): <list with run-count vs threshold>
```

**Reasoning.** Markdown matches persona output style (the orchestrator runs in a Claude Code session and surfaces this to the user — markdown renders directly in terminal/IDE). Preserves verbatim Issue/Recommendation/Evidence from each persona because rewriting them at orchestrator layer would contaminate the per-persona signal the panel design protects. Panel-severity and persona-severity both shown so the elevation logic is auditable post-hoc.

**Edge case it handles.** A finding flagged by 3 personas (e.g., synthetic Q7 violation that also touches Q5 ConfidenceTier requirements) — appears once as a single panel finding with `multi-persona agreement: 3 ({hypothesis-driven, data-trust, ...})`, the three personas' Issue/Recommendation bodies are merged via "Related concerns from <persona>:" suffix on the lead persona's Recommendation.

**Edge case it does NOT handle.** JSON / machine-readable side channel for tooling. Phase 2 may add this if a CI integration needs it. v1 is markdown-only.

---

## Risk 2 — Failure modes

### 2a. Subagent timeout

**Default: per-persona timeout 120 seconds; skip-with-flag on timeout; do NOT fail entire run.**

Concrete behavior:
- Each Agent tool call has a 120s soft timeout. Prior dispatch durations: 8s (sentinel), 23s, 24s, 31s, 33s, 40s (synthetic 3-finding Q7). 120s = 3× worst observed; gives headroom for slower model loads without dragging the orchestrator latency past a reasonable PR-review SLA.
- On timeout: orchestrator emits `Personas that failed to respond: [<persona-id> — timeout 120s]` in the Summary section + skips that persona's findings + still runs synthesis on the remaining personas' outputs.
- Three-out-of-four persona panel reviews are still useful; zero-persona panel reviews are not. Skip-with-flag maximizes salvageable signal.

**Edge case it handles.** Slow model load (cold-start) on one persona's dispatch; other three return in 30s; orchestrator emits a 3-persona panel with timeout note.

**Edge case it does NOT handle.** All 4 personas time out (likely Agent tool infrastructure outage, not panel issue). Orchestrator emits `Panel: all personas timed out. Re-run when infrastructure stable.` and exits non-zero. No partial review possible.

---

### 2b. Malformed output detection

**Default: regex-detect, retry once, then skip-with-flag.**

Malformed = output does NOT match either of:
- One or more `### Finding [N]:` headers followed by the 5-field structured template (Severity / Evidence / Framework citation / Issue / Recommendation). Tolerate trailing `Context gap:` lines.
- The persona's exact no-findings sentinel string (e.g., `No Q3 or Q1(c) findings.`).

Empty response, refusal patterns ("I cannot complete this task", "I apologize but…"), or partial template (e.g., Finding header with missing Framework citation field) → malformed.

Behavior:
1. Detect malformation via regex check on response body.
2. Retry the same persona dispatch ONCE with the same prompt.
3. If retry also malformed → skip-with-flag in Summary: `Personas that failed to respond: [<persona-id> — malformed after retry]`.

**Reasoning.** Subagent occasional bad output is a known LLM failure mode; one retry is cheap insurance. Persistent malformation across retries signals a persona-prompt issue (escalate to panel-design layer, not transient noise). Two retries don't materially improve recovery rate over one — diminishing returns.

**Edge case it handles.** Subagent hits a context-window edge or transient model burp; retry succeeds. Net cost: one extra dispatch (~30s).

**Edge case it does NOT handle.** Persistent malformation rooted in prompt ambiguity (e.g., persona prompt regression). Orchestrator skips + flags; panel-design layer must inspect the malformed output manually to diagnose. The retry doesn't fix prompt bugs; it fixes transient LLM bugs only.

---

### 2c. All-sentinel response

**Default: emit `Panel: no in-scope findings across Q1 / Q2 / Q3 / Q1(c) / Q5 / Q6 / Q7.` + skip origin-stamping.**

Concrete behavior:
- If all 4 personas return their no-findings sentinel: emit the line above as the panel review body, no Findings section, Run health section still present.
- Origin-stamping operates on findings; no findings means nothing to stamp; origin-stamp step skipped with `Origin-stamping: skipped (no findings).`

**Reasoning.** Independent-of-findings origin signals (e.g., "this PR contains AI-reviewer-suggested commits the panel did not flag — possibly because they touch dimensions outside Q1-Q7") are valuable but architecturally distinct from finding-stamping. That feature can land as a separate panel surface in Phase 3 if real PRs reveal demand. v1 keeps the all-sentinel path simple.

**Edge case it handles.** Panel reviewing a tool description-only change with no field surface impact (e.g., wording cleanup PR) — all 4 personas sentinel, panel emits one-liner, marketer reviewer moves on without noise.

**Edge case it does NOT handle.** AI-reviewer-suggested commits in the PR that the panel did not flag because they hit Q4 (decision-class — out of panel scope). Currently invisible to the panel-level output. Phase 3 candidate if recurrence is observed.

---

## Risk 3 — Case C routing to no-owner dimensions

**Default: hybrid — log every Case C; surface "coverage hole pattern" on recurrence threshold breach.**

Concrete behavior:

1. **Per-run logging.** Each `Context gap (out-of-lane): … route to <dimension>` line is parsed and added to the Context-gap routing table in the panel output (see 1c output shape). Resolution column reads `coverage hole — no panel owner` for Case C targets that match no current persona's dimensions.
2. **Recurrence tracking.** Orchestrator maintains a persistent `notes/coverage-hole-log.md` (append-only, one row per Case C escalation): `<run-date> | <PR id / SHA> | <source-persona> | <target-dimension> | <Evidence excerpt>`.
3. **Recurrence threshold breach.** If a single target-dimension accumulates 3+ rows in the last 10 runs (or last 14 days if fewer than 10 runs exist), the orchestrator emits a `🚨 Coverage hole pattern detected` callout in the panel Summary section pointing the panel-design layer at the dimension. Example: `🚨 Coverage hole pattern detected: Q4 decision-class flagged by Case C in 3 of last 10 runs ({brand-voice ×2, funnel-stage ×1}). Consider adding a Q4 persona to the panel or expanding an existing persona's lane.`
4. **Resolution path.** Panel-design layer (this workspace) decides whether to add a new persona, expand an existing persona's lane, or document the dimension as explicitly out-of-scope. The orchestrator does not auto-resolve.

**Reasoning.** Log-only (option a) loses the longitudinal signal — a single Case C escalation is noise; recurring escalations are signal. Escalate-on-recurrence (option b) without logging individual cases makes audit harder. Hybrid captures both granularities at low cost (one append per Case C, one threshold check per run).

**Edge case it handles.** Hypothesis-Driven persona escalates Q4 decision-class as Case C in 3 successive PR reviews → orchestrator surfaces the pattern → panel-design layer decides Q4 persona is worth adding before Q4 violations actually ship.

**Edge case it does NOT handle.** Threshold tuning (3-of-10 vs 2-of-5 vs other) is a guess — refinement deferred to smoke-test data after first 3-5 real PR runs. The first runs will be sparse; threshold may need lowering temporarily to surface patterns earlier.

---

## Deferred to Phase 2 (implementation)

These are not design decisions — they are implementation specifics that should NOT block Phase 1 review:

- **State storage** for `coverage-hole-log.md` and origin-stamping cache. File-based append in this workspace is the obvious default; revisit if Phase 3 adds CI integration.
- **gh CLI invocation pattern** for origin-stamping (fetching PR review history, parsing Codex suggestion threads). Will require shape probing on a real PR.
- **Dispatch prompt template** — the exact prompt the orchestrator sends to each subagent (persona file path + diff + PR review history excerpt + dispatch instructions). Will be informed by the same template the panel-design dispatches used during validation.
- **Markdown rendering / output sink** — orchestrator emits to stdout in v1; later may write to a file path or POST to a code-review surface.
- **Smoke test plan** — first 3 real PRs to run the orchestrator against, with deliberate before/after comparison to manual review.

---

## Carry-forward to Phase 2 — minimum viable orchestrator surface

The orchestrator is a single function (in the orchestrator session — no separate process) with this signature shape:

```
review_pr(pr_ref) →
  fetch_pr_diff(pr_ref)
  fetch_pr_review_history(pr_ref)
  dispatch_personas_in_parallel(diff, history) → [persona_outputs × 4]   (with timeout + retry + skip-with-flag)
  parse_findings(persona_outputs) → [findings]
  merge_related_evidence(findings) → [merge_groups]
  apply_compound_severity(merge_groups) → [findings_with_panel_severity]
  stamp_origin(findings_with_panel_severity, history) → [findings_with_origin]
  parse_context_gaps(persona_outputs) → [gaps]
  route_gaps(gaps) → [routed_gaps, coverage_holes]
  check_coverage_hole_threshold(coverage_holes) → [pattern_alerts]
  render_markdown_panel_review(findings_with_origin, routed_gaps, pattern_alerts, run_health) → output
```

Each step is testable in isolation. Implementation cost estimate: 3-5 hours for v1 if the gh CLI pattern is straightforward; double if origin-stamping has to do fuzzy matching against PR comment thread shapes.

---

## Phase 1 → Phase 2 gate

Andy reviews this design doc. If approved, Phase 2 implements the orchestrator. If Andy wants changes to any default above, iterate Phase 1 before Phase 2 starts.

The 3 risks are resolved with explicit defaults. The deferred items are implementation specifics, not design decisions.
