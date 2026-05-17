# Marketer Agent Panel

Multi-persona AI panel for reviewing code from a marketer's perspective. Four specialized personas dispatched in parallel by an orchestrator, each catching a distinct class of domain-semantic bug that general-purpose code review misses. Calibrated against real failure modes surfaced in [Umbraco Engage.AI](https://github.com/umbraco/umbraco.engage.ai) tool PRs.

---

## Why this exists

General-purpose LLM code review (Codex, pr-review-toolkit, etc.) is good at idiomatic correctness but blind to **what the code will make a marketer believe**. The canonical failure: in Umbraco Engage.AI PR #5, an AI reviewer suggested renaming `Goal.Value` (a generic configurable decimal) to `MonetaryValue` "for LLM clarity." The implementer accepted the suggestion. The rename was a semantic lie — marketers configure `Goal.Value` as money for some goals, scores for others, weights for others, zero-for-tracking for others. With the field renamed `MonetaryValue`, an LLM reading the response would total the goals and confidently report "$X in pipeline" when the column is a mix of dollars and scores. The marketer commits budget against fabricated currency arithmetic.

That bug shipped past automated review. It was caught at human PR review and reverted in commit `d483e05`. The pattern — **AI reviewer introduces a domain-semantic bug, automated review misses it, the marketer pays the cost** — is the failure class this panel is calibrated against.

The panel runs four marketer personas in parallel: each owns 1-2 audit dimensions, each speaks in the voice of their marketing specialty, each catches a different way wire-format language can mislead an LLM that narrates results to a marketer. See [`eval-inputs/pr-005-value-monetaryvalue.md`](eval-inputs/pr-005-value-monetaryvalue.md) for the canonical worked example.

---

## How it works

```
                ┌────────────────────────────────┐
  PR diff   ──> │  Orchestrator (Claude Code)    │
  + optional    │  • parse input form            │
  PR metadata   │  • fetch diff + review history │
                │  • dispatch 4 personas ║parallel│
                └─────────────┬──────────────────┘
                              │
        ┌──────────┬──────────┼──────────┬──────────┐
        ▼          ▼          ▼          ▼          ▼
  Brand-Voice  Funnel-Stage  Hypothesis  Data-Trust  (sentinel)
  Q3 + Q1(c)   Q2 + Q6       Q7          Q1 + Q5
        │          │          │          │
        └──────────┴────┬─────┴──────────┘
                       ▼
              ┌────────────────────────┐
              │  Orchestrator (cont.)  │
              │  • merge related-Ev    │
              │  • compound severity   │
              │  • origin-stamp (gh)   │
              │  • render markdown     │
              └────────────────────────┘
                       ▼
                 Panel Review
```

- **4 personas** = standalone Claude Code system prompts in [`personas/`](personas/). Each persona is self-contained: framework, audit dimensions, severity rubric, voice instructions, output template, escalation channel. Subagents read the persona file as their role brief and review the diff independently — no shared memory between them.
- **Orchestrator** = a runbook ([`orchestrator/orchestrator-prompt.md`](orchestrator/orchestrator-prompt.md)) that Claude follows step-by-step in an interactive session. Dispatches 4 personas in parallel via the `Agent` tool, parses findings, merges related Evidence (title-anchor heuristic), elevates compound severity, optionally stamps origin from PR review history, renders markdown.
- **Audit dimensions** are derived from `feedback_domain_field_audit.md` v1.1 — a 7-question framework that emerged from PR #5 retrospective + a 9-source OSS survey of marketer-agent codebases.
- **Validation gate**: each persona must catch its target eval case in [`eval-inputs/`](eval-inputs/) via live subagent dispatch before shipping.

---

## Quick example

Excerpt from a panel review of the PR #5 pre-fix code:

```markdown
### Finding 1: MonetaryValue rename promotes configurable decimal to currency
*Semantic merge group MV — Block + Block + Concern across {brand-voice, data-trust}*

- **Panel severity:** Block
- **Multi-persona agreement:** 2 personas flagged related Evidence
- **Framework citation:** Q3 generic-vs-imposed semantic, Q1(c) Stone-vs-Opinion
- **Issue:** This rename teaches the LLM to call configurable amounts "money."
  Marketers configure Goal.Value as currency for some goals, raw counts for others,
  priority scores for ranking, weights for scoring models. When the LLM totals
  the response and reports "$1,840 in pipeline this quarter," it's fabricating
  dollar arithmetic over a column that mixes dollars, scores, and weights.
- **Recommendation:** Revert to generic `Value`. Add tool-description sentence
  explaining configurability: "Value is a marketer-configured decimal; may be
  monetary, count, score, weight, or zero. Do not assume currency."
```

Full smoke-test output: [`eval-runs/2026-05-17-orchestrator-smoke-test-pr005-phase2.5-complete.md`](eval-runs/2026-05-17-orchestrator-smoke-test-pr005-phase2.5-complete.md).

---

## How to invoke

In a Claude Code session opened at this repo's root, ask Claude one of:

**PR-ref form** (requires [`gh`](https://cli.github.com/) CLI installed and authenticated):
> Run the marketer panel on `umbraco/umbraco.engage.ai#5`

**Local commit form** (no `gh` needed):
> Run the marketer panel on `<commit-sha>^:src/Umbraco.Engage.AI/Tools/MyNewTool.cs`

**Pasted diff form** (paste code directly into chat):
> Run the marketer panel on this diff: \`\`\`csharp ... \`\`\`

Claude follows [`orchestrator/orchestrator-prompt.md`](orchestrator/orchestrator-prompt.md) end-to-end. Without `gh`, all findings tag `origin: unknown` and the panel still produces full review output. Typical wall time: 60-90 seconds.

---

## Repo layout

| Folder | Purpose |
|---|---|
| [`personas/`](personas/) | 4 persona prompts — each is a complete system prompt for one subagent role |
| [`orchestrator/`](orchestrator/) | Runbook + dispatch template + parsing reference + origin-stamping subsystem spec |
| [`eval-inputs/`](eval-inputs/) | Frozen eval cases (3 from real PR #5 + 1 synthetic Q7) — what each persona must catch |
| [`eval-runs/`](eval-runs/) | Audit trail of validation runs + smoke tests — frozen historical archive |
| [`notes/`](notes/) | Orchestrator design notes + coverage-hole log (Case C escalations) |
| [`HANDOFF.md`](HANDOFF.md) | Internal future-self handoff doc — invocation cheat sheet, Phase 2.5 status, backlog |
| [`CONTRIBUTING.md`](CONTRIBUTING.md) | How to add a persona / eval case |

Several persona and orchestrator docs reference `<engage-workspace>/` and `<research-workspace>/` paths — placeholders for the author's private notes / research workspaces that contain the original audit-rule definitions and OSS survey data. These are optional read-deeper references; the embedded framework in each persona file is sufficient for the panel to operate.

---

## Status

**v0.1 weekend experiment.** Validated against PR #5 catches during build (MonetaryValue / IsInverted / IsInvalid + one synthetic Q7 case) — three smoke-test cycles, core catches stable across 6 dispatches.

**First real-PR validation run completed against PR #6 pre-fix code (2026-05-17):** panel replicated **2 of 2 domain-semantic catches** from human review (metric routing + timezone-aware boundaries) with multi-persona agreement on the primary catch. Real-PR validation period continues over Story 25 + 1 more PR. See [Testing strategy](#testing-strategy) below + [`docs/testing-the-panel.md`](docs/testing-the-panel.md) for methodology, and [`HANDOFF.md`](HANDOFF.md) for cold-start invocation (internal-handoff style; this README is external-onboarding style).

---

## Testing strategy

The panel encodes marketer perspectives caught by human reviewers in past PRs. We validate it against new PRs the same way: pick a known-bug code state, inject schema context, dispatch the panel blind, score against ground truth.

**Methodology (4-step flow):**

1. **Pick a pre-fix commit** of a PR with documented review catches. Fetch via `git show <ref>:<file>`. Panel must be blind to PR review outcome.
2. **Inject context-supplement block** (schema references, interface declarations, prior-story decisions). Discipline: provide facts, NOT hints about suspected bugs.
3. **Dispatch panel** (4 personas in parallel via Agent tool). Run twice for consistency check.
4. **Score findings** against ground truth (fix commits + PR review history). Match rubric: full / partial / different element / missed.

Aggregate over 3 PRs for confidence. Escalation triggers documented in [`docs/development-workflow.md`](docs/development-workflow.md) §"Empirical validation + escalation criteria".

**Validation state (1 of 3 PRs measured):**

- PR #6 (`engage_get_top_pages`) dry-run, 2026-05-17: panel replicated **2 of 2 domain-semantic catches** in panel scope (metric routing + timezone-aware boundaries). Multi-persona agreement on metric routing (3 personas converged). One borderline false positive (~6% FP rate). However, **panel scope covers approximately 50% of PR #6's total concerns** — 2 of 4 surfaced bugs fall outside panel's Q1-Q7 dimensions (LLM-prompt-engineering + vocabulary expansion). See [complete bug ecology in archive](eval-runs/2026-05-17-orchestrator-pr006-pre-fix-dry-run.md#complete-bug-ecology) for the full picture.
- Story 25 (next): run #2 of 3 will land when the PR ships.

**Honest limitations:**

- The PR #6 context block contained explicit schema mappings that hinted at the metric-routing bug; panel deserves partial credit (multi-persona framing, structured findings) not full discovery credit. The timezone catch was less context-leaked — panel connected `ReportingTimeZone` existence to UTC usage independently.
- 3 PRs is minimum viable signal. Real validation needs 10+ PRs over multiple sprints to baseline drift, false positive distribution, and persona-lane coverage.
- Panel catches Q1-Q7 domain-semantic dimensions. Bugs outside (pure engineering correctness, performance, security) won't be caught. Complementary to other review layers, not replacement.

See [`docs/testing-the-panel.md`](docs/testing-the-panel.md) for full methodology details + context-engineering discipline + match scoring rubric.

---

## License

[MIT](LICENSE). © 2026 Andy Vu Viet.

---

## Contributing

PRs welcome. See [`CONTRIBUTING.md`](CONTRIBUTING.md) for the persona structural template and validation-gate requirements. Short version: **new personas need a passing eval case before merge**.

---

## Author

Built by [Andy Vu Viet](https://github.com/andy-ub) as a weekend experiment while shipping Umbraco Engage.AI tools. The panel exists because [PR #5 in `umbraco/umbraco.engage.ai`](https://github.com/umbraco/umbraco.engage.ai/pull/5) caught three domain-semantic bugs that automated review missed; the panel is an attempt to systematize that catch.
