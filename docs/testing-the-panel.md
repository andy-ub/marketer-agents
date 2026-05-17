# Testing the Panel

How to validate the marketer-agent panel against known-bug PRs. The methodology used in the PR #6 dry-run, documented so future-Andy + external readers can apply it to new PRs and judge panel quality honestly.

---

## 1. Why test the panel

The panel encodes marketer perspectives that human reviewers caught in PR #5 of `umbraco/umbraco.engage.ai`. That doesn't tell us how well it generalizes to new tools, new audit dimensions, or PR review patterns from a different reviewer. Testing builds calibrated confidence: every panel dispatch against a known-state PR with documented catches generates an evidence row about catch rate, false-positive rate, and persona-lane coverage. Without testing, the panel is theater — a confident-sounding markdown report with no ground-truth tie. The 5-step methodology below is the minimum bar for a real-PR application to be useful evidence.

---

## 2. Test methodology

```
1. Pick a known-bug code state.
   - Best source: pre-fix commit of a merged PR with documented catches in fix
     commits + review history.
   - Fetch via: git show <pre-fix-commit-ref>:<file-path>
   - Panel must be blind to PR review outcome — using post-fix state defeats the test.

2. Inject context-supplement block.
   - Schema mappings, interface declarations, prior-story decisions.
   - See orchestrator/persona-dispatch-template.md for the input shape.
   - Discipline: provide facts, NOT hints about suspected bugs (see Section 3).

3. Dispatch the panel.
   - Orchestrator dispatches 4 personas in parallel via the Agent tool.
   - Run twice. Drift on secondary findings is expected (intrinsic LLM sampling
     variance); core catches should be stable across runs.

4. Score findings against ground truth.
   - Ground truth: fix commits + PR review thread.
   - Match rubric: Section 4.
   - Track false positives and false negatives separately.

5. Aggregate over 3 PRs.
   - A single PR is a noisy signal — sampling variance + context-block quality +
     persona-prompt drift all confound a single measurement.
   - Threshold + escalation decisions live in
     [`development-workflow.md`](development-workflow.md) §"Empirical validation".
```

---

## 3. Context-engineering discipline

The context-supplement block is load-bearing — the PR #6 dry-run confirmed that personas reason heavily from injected schema context. That power cuts both ways: well-crafted context surfaces real bugs, biased context pre-cooks the findings.

**Good context block contains:**

- Schema references the code diff doesn't show — interface declarations, related-table foreign keys, enum value definitions, configuration types.
- Public-facing source-of-truth (GitHub URLs to public repos when applicable).
- Prior-story decisions where pattern transfer matters (e.g., *"Story 01 dropped `IsInvalid` because Campaigns repo auto-filters; verify whether the entity in this diff behaves the same way."*).
- Tabular mapping when the code diff hides a relationship — fact-table → metric mapping, dimension → key mapping.

**Biased context block contains:**

- *"I suspect X might be wrong"* — pre-cooks a finding the panel was meant to surface.
- Quoted fix-commit messages, PR review comments, or human-reviewer suggestions — leaks the answer.
- Phrasing that matches expected catch wording — biases persona language and inflates voice quality scores.
- Information the panel would only have post-hoc (e.g., runtime error messages from a deployment that hasn't happened yet at PR-review time) — invalidates the blind test.

**Worked example — Good (PR #6 dry-run context, abbreviated):**

> *"The Engage Core analytics layer provides per-page metrics via these fact tables: FctPageview (has `pageId` FK), FctPageSessions (has `pageId` FK), FctSession (NO `pageId` FK — site-wide), FctUser (NO `pageId` FK — site-wide). Metric → fact table mappings: Metric.sessions → FctSession ✗ no pageId FK, breaks page-level joins; Metric.pageSessions → FctPageSessions ✓ joins on pageUrl dimension."*

Provides schema facts. Doesn't say *"the current code uses Metric.sessions which is broken."* The panel must connect the schema facts to the code itself.

**Worked example — Biased (DO NOT inject):**

> *"⚠️ The current implementation uses `Metric.sessions` and `Metric.users` on a per-page query — this is a runtime SQL bug because those fact tables have no `pageId` FK. Look for this in the diff."*

Tells the panel what to find. A catch produced from this context is not evidence of panel value — it's evidence the panel can parrot context.

---

## 4. Match scoring rubric

| Score | Definition | PR #6 example |
|---|---|---|
| **Full match** | Same source-code element flagged with comparable severity reasoning to ground-truth fix | Brand-Voice F1 flagged `Sessions` / `Users` columns on `TopPageResult` as imposing per-page semantics the data layer cannot back — same Evidence, Block severity, matches `95fb3d4` fix |
| **Partial match** | Same element flagged but different severity OR different framework citation than ground truth would suggest | Data-Trust F2 Run 1 flagged timezone-UTC mismatch as Concern; ground-truth fix commit `96c8544` treated it as ship-blocker. Run 2 escalated to Block — partial-match drift |
| **Different element flagged** | Panel raised a related concern on a different surface than ground truth | Funnel-Stage F2 Run 1 flagged `AvgTimeOnPage` polarity (engagement vs friction ambiguity) — legitimate Q2 concern, not in actual PR review, related-but-different from GT-1 metric routing |
| **Missed** | Element not flagged at all | GT-2a (reusable `PeriodVocabulary` description const + LLM clarification gate) was an engineering refactor — outside panel domain dimensions, correctly missed |

A "Missed" score is not always a failure — if the missed element is engineering-quality (out of panel scope), it's correctly missed. Mark the score with the reason.

---

## 5. Aggregation + escalation

A single PR is sampling-variance-bound. Aggregate over 3 PRs in the validation period, then apply the thresholds documented in [`development-workflow.md`](development-workflow.md) §"Empirical validation + escalation criteria":

- **≥80% human-reviewer replication + 0 production bugs in panel dimension** → panel proven. Continue Step 5 only.
- **50-80% replicated** → iterate persona prompts (false-positive trimming, severity rubric tightening, persona lane sharpening). Stay at Step 5.
- **<50% replicated OR a production bug surfaces post-merge in a panel dimension** → escalate to Step 1 design panel (review the design before implementation, not just after).

After each validation PR, fill the **Outcome tracking** section in that PR's `eval-runs/` archive (see PR #6 dry-run for the template).

---

## 6. When to skip testing

**Skip per-PR test when:**

- 3+ measurements already aggregated with a clear verdict (panel proven or escalated — re-testing every PR is overhead beyond what the signal requires).
- PR is trivial: test-only, comment fix, build config, dep bump.
- Time-pressured release window — document the deferral in the PR description so the gap is auditable.

**Always test when:**

- First PR after a persona or orchestrator change (regression check — did the change preserve catch rate on PR #6 pre-fix code?).
- PR adding a new persona's lane (new persona needs an eval case per `CONTRIBUTING.md`; the validation gate is non-optional).
- PR after an escalation trigger fired (validate the escalation worked — Step 1 + Step 5 panel should now catch what Step 5 alone missed).

---

## 7. Honest limitations

**Context block leak.** The PR #6 dry-run context block contained the explicit metric → fact-table mapping that included the line *"Metric.sessions → FctSession ✗ no pageId FK, breaks page-level joins."* That sentence is one logical step away from GT-1's finding. The panel deserves partial credit for the GT-1 catch — it operationalized the schema fact into structured findings, framed it via multiple framework dimensions (Q3 + Q1(c) + Q6 + Q1), generated multi-persona agreement, and surfaced concrete marketer-impact narratives — but should not be credited as "found the bug from scratch." A future test should inject schema facts without the *"breaks page-level joins"* annotation and re-measure. GT-2b (timezone) catch was less context-leaked: the panel connected `ReportingTimeZone` existence in context to the code's `_timeProvider.GetUtcNow().UtcDateTime` usage independently, with no annotation hinting at the conflict.

**3-PR sample size.** Three PRs is the minimum viable signal — it identifies catastrophic miss rates and obvious calibration problems, but not subtle drift, persona-lane coverage holes that emerge across diverse code surfaces, or false-positive distributions across tool types. Real validation needs 10+ PRs over multiple sprints to baseline these. Do not over-claim panel value after 3 PRs. Treat the 3-PR threshold as "minimum bar to keep using the panel," not "panel is proven."

**Domain coverage gap.** The panel catches Q1-Q7 dimensions: Stone-vs-Opinion + filter policy + polarity + funnel attribution + hypothesis-echo + evidence tier + cross-tool handoff. Bugs outside this set — pure engineering correctness, performance, security, build configuration, test coverage — will not be caught. The panel is complementary to other review layers (pr-review-toolkit, Codex, human review per [`development-workflow.md`](development-workflow.md)), not a replacement. A PR where every bug is outside Q1-Q7 will produce a "No findings" panel — that is a correct result, not a panel failure.

**Reviewer signal vs panel signal.** Ground truth in these tests is "what the human reviewer caught" — but human reviewers also miss things. A panel finding flagged as "False Positive" because the human reviewer didn't surface it may actually be a genuine catch the reviewer missed. The honest fix is to surface ambiguous FPs to a second human reviewer before counting them as panel noise. Until that's done, the FP rate has an upward bias.
