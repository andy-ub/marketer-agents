# Marketer Agent Panel

Manual marketer-review agent panel for Engage.AI PRs — 4 distinct AI personas that review code diffs from a marketer's perspective, catching the class of domain-semantic bugs that human reviewers caught in PR #5 (Story 03 `engage_get_goals`).

## Why

Code review by general-purpose LLMs (Codex, pr-review-toolkit) misses domain-semantic bugs and sometimes actively introduces them — see [`eval-inputs/pr-005-value-monetaryvalue.md`](eval-inputs/pr-005-value-monetaryvalue.md), where Codex suggested renaming `Value` → `MonetaryValue` and the implementer accepted it. The panel pattern adds a marketer's perspective as an independent check.

## Approach

**Manual party-mode (BMAD-inspired, framework-free).** Each persona is a markdown file containing a self-contained system prompt — personality, embedded framework, audit dimensions, severity rubric, output format. When reviewing a PR, the orchestrator (a Claude Code session) dispatches each persona as a subagent in parallel via the `Agent` tool, collects 4 perspectives, and synthesizes findings.

No framework dependency — pure prompt engineering + manual orchestration.

## Layout

```
marketer-agents/
├── personas/             # 4 .md persona prompts (one per role)
├── eval-inputs/          # 3 PR #5 eval cases (copied from engage-workspace, frozen baseline)
├── eval-runs/            # Outputs from panel review sessions (gitignored later)
└── README.md
```

## Personas

| # | Role | Audit dimensions | Status | Validation |
|---|---|---|---|---|
| 1 | Brand-Voice Marketer | Q3 generic-vs-imposed, Q1(c) Stone-vs-Opinion | v0.3 shipped | Catches `pr-005-value-monetaryvalue` 3/3 persona-level (criterion #4 stamped at orchestrator layer per eval addendum) |
| 2 | Funnel-Stage Marketer | Q2 polarity, Q6 funnel-stage attribution + cross-tool handoff | v0.2 shipped | Catches `pr-005-is-inverted` 4/4 + bonus Q6 handoff catch |
| 3 | Hypothesis-Driven Marketer | Q7 hypothesis echo | v0.2 shipped | 2-run negative + synthetic positive (no Q7 in PR #5 set); synthetic eval lifted to `eval-inputs/synthetic-q7-segments-tool.md`; v0.2 re-dispatch (`eval-runs/2026-05-16-persona-03-v0.2-redispatch.md`) confirms framework prose drives Recommendation after Worked Example A fix-portion stripped (anchor-leak mitigation) |
| 4 | Data-Trust Marketer | Q1 repository filter policy, Q5 evidence tier (forward-looking) | v0.1 shipped | Catches `pr-005-is-invalid` 4/4 + bonus Q1 data-source fidelity catch; Q5 forward-looking per Andy decision (no synthetic eval — trigger criterion in `notes/orchestrator-design-notes.md` §5) |

## Validation gate

Original commitment: panel must catch all 3 PR #5 eval cases when fed the pre-fix code. **Closed 2026-05-16** — persona #4 dispatch against `pr-005-is-invalid` was the final criterion.

Per-persona validation requirements before drafting persona N+1: persona N must catch its target eval case via live subagent dispatch (or pass a 2-run negative + synthetic positive validation if no PR #5 case covers its dimension — see persona #3).

Validation results per persona live in `eval-runs/` with verdicts and PASS/FAIL detail.

**Panel-level final review** (cross-cutting integration audit, 2026-05-16): consistency / voice differentiation / anchor-leak / orchestrator-readiness / severity calibration audited together. One iteration required (persona #3 anchor-leak mitigation v0.1 → v0.2 + lift synthetic Q7 to `eval-inputs/`). **Panel is shipping-ready; orchestrator phase may proceed.** Architectural risks captured for orchestrator phase: merge mechanics, failure-mode handling, Case C routing to no-owner dimensions.

## Orchestrator

**Phase 1 design** (2026-05-16, approved): [`notes/orchestrator-design.md`](notes/orchestrator-design.md) — defaults on 3 architectural risks (merge mechanics, failure modes, Case C routing).

**Phase 2 implementation** (2026-05-16): [`orchestrator/`](orchestrator/) — runbook + dispatch template + parsing reference + origin-stamping subsystem spec. Invocation guide in [`orchestrator/README.md`](orchestrator/README.md).

- Optional `gh` CLI dependency: required only for origin-stamping (Step 5). Graceful degradation if absent — all findings tagged `origin: unknown`, panel review still produced.
- State files: append-only [`notes/coverage-hole-log.md`](notes/coverage-hole-log.md) (Case C routing), optional per-run archive in `eval-runs/`.

**Phase 3 smoke test** (pending): first 3 real-PR runs validate token-overlap heuristic, Case C noise rate, origin-stamping precision. Attention points documented at the bottom of [`orchestrator/orchestrator-prompt.md`](orchestrator/orchestrator-prompt.md).

## Orchestration constraints

- Each persona reviews independently. No cross-persona references.
- Dedupe / merge is the orchestrator's job, not the persona's. Multi-persona agreement on the same Evidence raises confidence.
- Every finding must cite a framework dimension — uncited findings are downgraded by the orchestrator.

## References

- Engage.AI tools: `D:/source/work/umbraco.engage.ai/src/Umbraco.Engage.AI/Tools/`
- Engage design patterns: `D:/source/work/engage-workspace/notes/engage-design-patterns.md`
- Per-entity Q1 matrix: `D:/source/work/engage-workspace/notes/domain-rules.md`
- Audit rule v1.1: `~/.claude/projects/d--source-work-engage-workspace/memory/feedback_domain_field_audit.md`
- Research workspace: `D:/source/work/marketer-agent-research/`
