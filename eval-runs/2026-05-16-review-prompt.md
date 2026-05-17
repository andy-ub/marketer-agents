# Review prompt — persona #1 v0.1 (Brand-Voice Marketer)

Copy everything below the `---` line into a fresh Claude Code session at `D:/source/work/marketer-agents`. The session has read access to all referenced paths.

---

I'm building a manual marketer-review agent panel for Engage.AI PRs. The panel is 4 distinct persona prompts (markdown files) that review code diffs from a marketer's perspective. The orchestrator dispatches each persona as an independent subagent via the `Agent` tool, then synthesizes findings.

I've just drafted persona #1 (Brand-Voice Marketer — owns Q3 generic-vs-imposed and Q1(c) Stone-vs-Opinion dimensions) and run one live-dispatch validation against a known failure case from PR #5 (`MonetaryValue` rename). I want a structural review of the persona design and the validation result before I scale to persona #2.

**Don't write code or draft persona #2.** This is a review-only request. Give me a verdict and a short list of concrete changes (if any) to the persona file before I proceed.

## Files to read

Read these in order:

1. `D:/source/work/marketer-agents/README.md` — workspace overview + panel design.
2. `D:/source/work/marketer-agents/personas/01-brand-voice-marketer.md` — the persona prompt (v0.1).
3. `D:/source/work/marketer-agents/eval-inputs/pr-005-value-monetaryvalue.md` — the eval case the persona must catch, including explicit PASS/FAIL criteria.
4. `D:/source/work/marketer-agents/eval-runs/2026-05-16-persona-01-v0.1-pr005-monetaryvalue.md` — the live-dispatch output + my verdict (3/4 PASS, criterion #4 reclassified as orchestrator-level).
5. (Optional, for deeper grounding) `D:/source/work/engage-workspace/notes/engage-design-patterns.md` §2 — Stone vs Opinion shape variants (the canonical OSS pattern referenced in the persona).

## Context the persona file does not state explicitly

- The panel uses **subagent dispatch**: each persona is the role brief for a fresh Claude session spawned via `Agent`. No shared memory between personas. The orchestrator (a Claude Code session) sends the diff to each subagent in parallel and synthesizes findings.
- **Dedupe is the orchestrator's job, not the persona's.** Personas review independently. Multi-persona agreement on the same Evidence raises confidence — that's a synthesis-layer signal, not something individual personas track.
- The Severity / Evidence / Framework citation fields are structured metadata (neutral wording). The Issue and Recommendation bodies must preserve **the persona's voice** — different personas should sound different even within the same finding template.
- Framework citation is **mandatory per finding**. The orchestrator downgrades uncited findings as "could be generic LLM nitpick."
- This is persona #1 of 4. Personas #2–#4 (Funnel-Stage, Hypothesis-Driven, Data-Trust) will be drafted only after persona #1 passes validation, so the persona file structure that emerges here becomes the template.

## Questions I want you to answer

1. **Calibration-anchor leakage.** The persona file ends with a "Calibration anchor — the canonical case" section that contains the exact expected finding for the `MonetaryValue` rename — i.e., the same case the validation eval tests. The subagent reproduced the calibration finding almost verbatim. It also independently caught a second issue (the `monetary value` prose leak in the tool `Description`) that was NOT in the calibration anchor — which suggests it actually applied the framework, not just echoed. But: is the calibration anchor a methodological problem? Specifically: does the anchor effectively pre-cook the eval result, and should it be removed (or replaced with an abstract analogue) before this persona is used on novel diffs? If you recommend keeping it, justify why; if you recommend removing/replacing, propose the replacement.

2. **Voice preservation.** Read Findings 1 and 2 in the validation output. Do they sound like a senior brand-voice marketer briefing an implementer, or do they sound like a generic LLM review? Cite specific sentences as evidence either way. If voice is weak, point to where the persona prompt would need to push harder (anti-tropes? specific phrasings to require? examples of bad-voice findings to negate?).

3. **Severity discipline.** The persona uses Block / Concern / Nit with concrete examples. The validation output marked Finding 1 as Block and Finding 2 as Concern. Are those calls correct given my severity rubric? Is the rubric tight enough to prevent overclaim, or do you see a failure mode where future findings will drift toward Block?

4. **Criterion #4 split decision.** The eval case requires the reviewer to surface "the failure was introduced by an AI reviewer (not the implementer)." The persona missed this because the signal lives in PR review history, not the code diff. My verdict (option A) is: the persona should produce diff-grounded findings only, and the orchestrator should stamp AI-reviewer-introduced as metadata from PR review history when reviewing real PRs. Is that the right split? Or should the persona prompt include a heuristic for spotting AI-reviewer-introduced patterns (overly verbose naming, "for LLM clarity" rationale shape) even without PR history access? Argue for one option.

5. **Scope discipline / "stay in lane."** The persona prompt explicitly lists dimensions it does NOT review (Q1 IsInvalid, Q2 IsInverted, Q5 evidence tier, Q6 funnel, Q7 hypothesis). The eval input (`GetGoalsTool.cs` pre-fix) also has a Q1 IsInvalid and Q2 IsInverted issue. The persona correctly did NOT flag those. Is the "stay in lane" instruction robust enough, or is there a phrasing that would tempt the persona to drift (e.g., a Q1 issue that overlaps semantically with Q3)? If you see overlap risk, point to it.

6. **Reusability for personas #2–4.** Assuming you'd approve persona #1 (with or without changes), would you reuse this file as the structural template for personas #2–4? Identify which sections are persona-specific (Identity / Framework / Audit dimensions / Worked example / Calibration anchor) vs which are panel-shared (Severity rubric / Output format / Constraints / Reference). For the panel-shared sections, do they belong in a separate `personas/_shared.md` that each persona references, or stay inlined per persona (the embedded-self-contained constraint)?

## Output format I want

Structured markdown response with a section per numbered question above. Lead each section with a one-line verdict (e.g., `### Q1 — Calibration leakage: remove anchor`), then 3–6 sentences justifying. At the end, a `## Overall verdict` section: `Ship persona #1 as v0.1` / `Iterate to v0.2 with these changes:` / `Significant rework needed:`. Be direct. I'd rather hear "this is wrong because X" than diplomatic hedging.
