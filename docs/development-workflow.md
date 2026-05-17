# Development Workflow — Where the Panel Fits

## TL;DR

The panel is a single pre-push review step for AI tool PRs. Runs once per PR at Step 5, takes 5-10 minutes. Does NOT touch other workflow steps. Validated empirically over the first 3 PRs — escalation criteria in Section 6.

---

## The 8-step flow

```
1. Planning           — apply v1.1 domain audit (existing memory rule)
2. Implementation     — code + unit tests pass
3. pr-review-toolkit  — 3 agents (existing): code-reviewer / silent-failure-hunter / type-design-analyzer
4. Codex review       — style, edge cases, redundancy (existing per memory rule)
5. Panel self-review  — invoke orchestrator (NEW — this repo)
6. UI verification    — Playwright + Copilot (existing, when tool surfaces have UI)
7. Push + link PR     — push branch, link to ADO work item (existing)
8. Human review       — PM/lead final gate (existing)
```

Each step catches a different failure class. The panel slots between automated style/code review (Steps 3-4) and human business-context review (Step 8). It covers the gap where the wire-format names, types, and descriptions of a tool surface make claims the underlying data does not back — the failure class that produced PR #5's `MonetaryValue` rename bug and PR #6's site-level-metric-on-per-page-tool bug.

---

## When to invoke the panel

Apply panel for PRs that:

- Add a new AI tool (`AITool` attribute on a new class).
- Change tool result record fields (add / remove / rename / retype).
- Change a tool's `Description` string (LLM-facing wording).
- Change argument descriptions (LLM-facing wording on the input contract).

Skip panel for PRs that:

- Pure code refactor with no tool-surface change.
- Test-only changes.
- Comment / typo / docs.
- Build / CI / config.

Rule of thumb: **if the diff changes what the LLM will read or narrate to a marketer, run the panel**. If it doesn't, skip.

---

## How to invoke

Open Claude Code at this repo's root. Tell it: *"Run the marketer panel on `<commit-ref or branch-ref or pasted-diff>`."* The orchestrator dispatches 4 personas in parallel, returns a markdown panel review in ~60-90 seconds. Input forms (PR-ref / local commit / pasted diff) and output structure are in [`../HANDOFF.md`](../HANDOFF.md).

When dispatching against new tool surfaces, **include source-of-truth schema context** for the Engage Core types the tool touches — fact table names, FK relationships, enum values, configuration types. The PR #6 dry-run ([`../eval-runs/2026-05-17-orchestrator-pr006-pre-fix-dry-run.md`](../eval-runs/2026-05-17-orchestrator-pr006-pre-fix-dry-run.md)) confirmed this context is load-bearing: without it, the panel reasons from field names alone and may miss schema-level conflicts.

---

## Reviewer layer positioning

| Layer | Tool | What it catches |
|---|---|---|
| Engineering tier | pr-review-toolkit (3 agents) | Code quality, error handling, type design |
| Style / edge tier | Codex | Style consistency, edge cases, redundancy |
| **Domain tier (this panel)** | **Marketer panel (4 personas)** | **Domain-semantic bugs at the tool surface — what the wire format will make a marketer believe** |
| Final gate | Human reviewer (PM / lead) | Business intent, domain expertise, downstream impact |

The panel encodes the marketer-perspective-on-source-code pattern that human review caught post-Codex-acceptance in PR #5 and PR #6. Complementary to other layers — run them as usual.

---

## Empirical validation + escalation criteria

The panel is in validation phase over the first 3 AI tool PRs. After each PR, fill the **Outcome tracking** section in that PR's `eval-runs/` archive.

Aggregate over 3 PRs:

- **≥80% of human-reviewer catches replicated by panel + 0 production bugs in panel dimensions** → panel proven. Continue Step 5 only.
- **50-80% replicated** → iterate persona prompts. Tighten false-positive surfaces, sharpen lane boundaries, calibrate severity rubrics. Stay at Step 5.
- **<50% replicated OR a production bug surfaces post-merge in a panel dimension** → escalate to Step 1 design panel. The panel reviews the design *before* implementation, not just after — adds ~30 min per PR (Step 1 + Step 5 dispatch).

The Step 1 escalation expands the panel's role from "catch bugs in implementation" to "catch design flaws before implementation locks them in." Only triggered if Step 5 alone proves insufficient.

Current state (1 of 3 PRs measured — PR #6 dry-run):
- Replication rate: 100% (2 of 2 domain-semantic catches; engineering-refactor item correctly out-of-scope).
- Production bugs in panel-dimension: n/a (dry-run on already-fixed PR).
- False positive rate: ~6%.

On track to **"continue Step 5 only"** verdict pending 2 more PR runs.

---

## Break-even

| Side | Estimate |
|---|---|
| Cost | ~5-10 min per AI tool PR (orchestrator dispatch + reading panel output) |
| Value per catch | ~30 min human-reviewer back-and-forth avoided + ~1 day push/fix cycle avoided |
| Break-even rate | 1 catch per 3-4 PRs |

PR #5 had 3 catches in 1 PR. PR #6 had 2 catches in 1 PR. Above break-even rate so far; revisit after 3 measured PRs.
