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

**v0.1 weekend experiment.** Validated against the three PR #5 eval cases (MonetaryValue / IsInverted / IsInvalid) + one synthetic Q7 case. Three smoke-test cycles run against PR #5 pre-fix code; core catches stable across all 6 dispatches.

Real-PR validation (against an actual not-yet-merged PR) pending. See [`HANDOFF.md`](HANDOFF.md) for cold-start invocation and current Phase 2.5 status. `HANDOFF.md` is internal-handoff style; this README is external-onboarding style.

---

## License

[MIT](LICENSE). © 2026 Andy Vu Viet.

---

## Contributing

PRs welcome. See [`CONTRIBUTING.md`](CONTRIBUTING.md) for the persona structural template and validation-gate requirements. Short version: **new personas need a passing eval case before merge**.

---

## Author

Built by [Andy Vu Viet](https://github.com/andy-ub) as a weekend experiment while shipping Umbraco Engage.AI tools. The panel exists because [PR #5 in `umbraco/umbraco.engage.ai`](https://github.com/umbraco/umbraco.engage.ai/pull/5) caught three domain-semantic bugs that automated review missed; the panel is an attempt to systematize that catch.
