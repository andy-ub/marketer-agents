---
title: Marketer-Panel Orchestrator — Runbook
status: v0.1 implementation
created: 2026-05-16
design-source: notes/orchestrator-design.md (Phase 1, approved)
gh-cli-dependency: optional with graceful degradation (origin-stamping skipped if gh unavailable)
---

# Marketer-Panel Orchestrator — Runbook

This is the runbook you (Claude) follow to execute a panel review. Read it once at start, then execute each step in order. Each step is testable in isolation. Output is markdown rendered to the user.

## Invocation contract

The user invokes the orchestrator by saying *"run the marketer panel on <X>"* or by `/marketer-panel <X>` if a slash command wrapping this runbook exists. `<X>` is one of:

- **PR-ref form:** `owner/repo#NUMBER` (e.g., `umbraco/umbraco.engage.ai#5`). Requires `gh` CLI installed + authenticated. Orchestrator fetches diff + review history.
- **Local form:** an absolute path to a diff file, OR a commit ref + path (`d483e05^ -- src/Umbraco.Engage.AI/Tools/GetGoalsTool.cs`). No `gh` needed. Origin-stamping degrades to `unknown` for all findings unless a PR-ref is *also* supplied.
- **Pasted form:** the user pastes diff text directly. Orchestrator reads it from chat context. Origin-stamping degrades to `unknown`.

If the input form is ambiguous, ask the user to clarify before starting. Do not guess.

## Pre-flight checks

Run these in parallel before step 1:

1. `gh --version` (PowerShell) — if exit code ≠ 0, set `origin_stamping_available = false`. Continue. Surface degradation in Run health section.
2. Confirm at least one persona file exists at `personas/0[1-4]-*.md`. If not, abort with `Panel: persona files missing — cannot dispatch.`
3. Confirm `notes/coverage-hole-log.md` is writable (create if absent — empty file with frontmatter is fine).

---

## Step 1 — fetch_pr_diff + fetch_pr_review_history

### PR-ref form

```bash
gh pr diff <NUMBER> --repo <owner/repo>           # → diff text
gh pr view <NUMBER> --repo <owner/repo> --json title,body,headRefName,baseRefName,reviewDecision,reviews,comments,commits
```

The JSON output is the input to origin-stamping (Step 5). Cache it in-session — do not refetch.

For the review history specifically: `gh pr view ... --json reviews,comments` gives reviewer comments. To find AI-reviewer suggestions (Codex, pr-review-toolkit), filter the comments/reviews where `author.login` matches a known bot pattern (`codex-bot`, `coderabbitai`, `claude-bot`, etc.) — initial bot list to maintain in `state/known-ai-reviewers.md` (Phase 2.5; for v1, treat any user with `[bot]` suffix or matching `*codex*` / `*claude*` / `*reviewer*` as `ai-reviewer-suggested`).

### Local / pasted forms

Use `git show <ref>:<path>` for commit-ref form, or read the diff file path directly. Skip the review-history fetch — set `origin_stamping_available = false` for this run.

### Output of Step 1

In-session variables (not files):
- `diff_text` — the unified diff or full-file pre-fix snapshot
- `pr_metadata` — JSON object from `gh pr view`, or `null`
- `origin_stamping_available` — bool

---

## Step 2 — dispatch_personas_in_parallel

Send 4 parallel `Agent` tool calls in a single message. Each call uses:

- `subagent_type`: `general-purpose`
- `description`: e.g. `Brand-Voice marketer reviews <pr-ref>`
- `prompt`: built from `orchestrator/persona-dispatch-template.md` with substitutions for `{persona-file-path}`, `{diff_text}`, `{pr_metadata_excerpt}`. See template for exact shape.

Persona file paths to dispatch (in panel order):

1. `personas/01-brand-voice-marketer.md`
2. `personas/02-funnel-stage-marketer.md`
3. `personas/03-hypothesis-driven-marketer.md`
4. `personas/04-data-trust-marketer.md`

### Timeout + retry policy (Risk 2a, 2b)

- **Timeout:** 120s per call. If a single Agent call exceeds 120s, mark that persona `timeout` and proceed with the other 3 outputs.
- **Malformed detection:** after each call returns, regex-check the response (see `orchestrator/regex-patterns.md` §1 + §2). If neither template nor sentinel matches → retry that persona ONCE with the same prompt.
- **Retry malformed:** if retry also malformed → mark that persona `malformed`.
- **All-4-failed:** if all 4 personas timeout or malformed → emit `Panel: all personas failed to respond. Re-run when infrastructure stable.` and exit non-zero (in-session: stop and surface to user).

### Raw-output preservation (audit-trail requirement)

After each Agent call returns (including retries; including timeouts and malformed responses), write the **verbatim** subagent response to:

```
eval-runs/raw/<YYYY-MM-DD>-<input-id>-<persona-id>-run<N>.md
```

Where:
- `<YYYY-MM-DD>` is the run date.
- `<input-id>` derives from invocation: `pr<NUMBER>` for PR-ref form (e.g., `pr006`), or the short commit-SHA for local-commit form, or `local-diff-<HHMM>` for pasted form.
- `<persona-id>` is one of `brand-voice`, `funnel-stage`, `hypothesis-driven`, `data-trust`.
- `<N>` is the run-within-this-dispatch-session counter (1 if this is a single-run dispatch; 1/2 for a consistency-check pair).

The raw file contains: the verbatim response text, prepended with a 3-line YAML header (`persona`, `run-date`, `dispatch-status: ok | timeout | malformed`). No analysis, no classification — just the artifact. If a persona timed out or returned malformed text, save the (possibly empty) response alongside the status; preserve failure artifacts identically.

This preserves the source artifact so independent reviewers can audit reasoning-quality claims against verbatim subagent text, not against the builder's classification of that text. Without raw preservation, every Reasoning-quality claim in a summary archive is unfalsifiable except by re-dispatch (which is non-deterministic).

### Output of Step 2

In-session variables:
- `persona_outputs` — dict: `{persona_id: response_text | "timeout" | "malformed"}`
- `failed_personas` — list of persona_ids that did not produce output

On-disk artifacts (4-8 files per dispatch session, depending on run-count + retries): `eval-runs/raw/<...>.md` per the schema above.

---

## Step 3 — parse_findings + parse_context_gaps

For each persona response in `persona_outputs` (skip failed personas):

### 3a. Detect sentinel

Match against the persona-specific sentinel (see `orchestrator/regex-patterns.md` §2):
- `No Q3 or Q1(c) findings.` (Brand-Voice)
- `No Q2 or Q6 findings.` (Funnel-Stage)
- `No Q7 findings.` (Hypothesis-Driven)
- `No Q1 or Q5 findings.` (Data-Trust)

If sentinel matches: record `persona_findings[persona_id] = []` and continue.

### 3b. Extract findings

If not sentinel: parse with the finding-template regex (`orchestrator/regex-patterns.md` §1). Each finding produces an in-session object:

```
{
  finding_id: "<persona_id>-<n>",
  persona_id: <persona_id>,
  severity: "Block" | "Concern" | "Nit",
  evidence: <raw string>,
  evidence_tokens: <list of identifier tokens, lowercased, ≥3 chars, stopwords removed>,
  evidence_file: <extracted file path, if present>,
  framework_citation: <raw string>,
  reasoning: <raw markdown body>,
  issue: <raw markdown body>,
  recommendation: <raw markdown body>,
}
```

### 3b-bis. Extract Considered-but-not-flagged section (persona-level, optional)

After all findings in a persona's response, scan for an optional `## Considered but not flagged` heading. If present, parse the subsequent bullet list — each bullet has format `` - `<element>` — <reason> ``. Record per-persona:

```
{
  persona_id: <persona_id>,
  considered_but_not_flagged: [
    {element: <raw>, reason: <raw>},
    ...
  ]
}
```

If absent, the persona simply has no `considered_but_not_flagged` entry. Empty header with no bullets → malformed (retry).

### 3c. Extract Context-gap lines

After the last `### Finding N:` block (or instead of one, if a persona emitted sentinel-with-gap which is allowed): scan for `Context gap (overlap|need-context|out-of-lane):` lines. For each:

```
{
  persona_id: <source>,
  case: "A" | "B" | "C",
  target_dimension: <raw string after the case prefix>,
  evidence_or_note: <raw remainder of the line>,
}
```

### Output of Step 3

- `findings` — flat list of all findings across all personas (each finding has `reasoning` field)
- `considered_but_not_flagged` — dict: `{persona_id: [{element, reason}, ...]}` for personas that emitted the section
- `context_gaps` — flat list of all gap escalations

---

## Step 4 — merge_related_evidence + apply_compound_severity

### 4a. Related-Evidence grouping (Risk 1a — title-anchor heuristic, post Bug §1 fix)

**Background.** The original `≥1 shared Evidence-backtick token after stopword filter` heuristic over-merged in the 2026-05-17 smoke test (18/21 pairs falsely merged — see [`eval-runs/2026-05-17-orchestrator-smoke-test-pr005.md`](../eval-runs/2026-05-17-orchestrator-smoke-test-pr005.md) Bug §1). Root cause: omission-class findings cite full record declarations as Evidence context, so token noise from unrelated fields appears in every such finding. Andy's initial extended-stopword + ≥2-token proposal did not address this — `MonetaryValue` token appears in both BV-F1's Evidence (rename finding) and FS-F1's Evidence (record declaration showing IsInverted missing). Fix: switch primary signal from Evidence-backtick tokens to **finding-title tokens**, with Evidence and framework-citation as fallback signals.

**Algorithm v2:**

For each unordered pair of findings `(f1, f2)`:

1. If `f1.evidence_file != f2.evidence_file` → **not related**, stop.

2. Compute anchor overlap: `shared_anchor = f1.anchor_tokens ∩ f2.anchor_tokens` (per [`regex-patterns.md`](regex-patterns.md) §4 aggressive stopword list).
   - If `len(shared_anchor) ≥ 1` → **related**, stop.

3. Fallback: framework-citation agreement.
   - Parse `f1.framework_citation` and `f2.framework_citation` for dimension labels (`Q1`, `Q1(c)`, `Q2`, `Q3`, `Q5`, `Q6`, `Q7`).
   - If the dimension sets share a label AND `shared_context = f1.context_tokens ∩ f2.context_tokens` has `len(shared_context) ≥ 2` → **related**.

4. Otherwise → **not related**.

Build merge groups via union-find: each finding belongs to exactly one group; groups of size 1 contain a single finding, groups of size ≥2 indicate related-Evidence overlap.

**Why title-anchor primary:** the finding title names what the finding is *about* (the specific field, the specific imposition, the specific omission). Two findings about the same field will both name it in their titles. Two findings about different fields on the same record will have non-overlapping titles even when their Evidence-backtick context overlaps.

**Why framework-citation fallback:** prevents two findings on the same line of code with the *same* framework dimension from missing each other when title vocabulary varies (e.g., one persona says "promotes to currency" while another says "imposes monetary semantic" — both are Q3 generic-vs-imposed on the same Evidence; context-token check provides the same-evidence confirmation).

**Documented deviation from Phase 1 spec:** Phase 1 specified Evidence-backtick tokens with ≥1-token threshold. Phase 1 design + Andy's Phase 2-approval iteration didn't anticipate the Evidence-quote-the-whole-record pattern that omission-class findings use. Bug §1 fix changes the primary signal from Evidence tokens to title tokens. Smoke-test data drove the change; archive [`eval-runs/2026-05-17-orchestrator-smoke-test-pr005-post-fix.md`](../eval-runs/2026-05-17-orchestrator-smoke-test-pr005-post-fix.md) documents the precision/recall delta.

### 4b. Compound severity (Risk 1b)

For each merge group:
- Count distinct persona-ids among the group's findings.
- Apply the severity matrix:

| Configuration | Panel-level severity |
|---|---|
| 1 finding (singleton group) | unchanged |
| any Block in group | Block; fold other findings' Recommendations into the Block finding via "Related concerns from {persona-ids}: …" |
| ≥2 distinct-persona Concerns, no Block | **Panel-level Block** (elevation); preserve per-finding Concern severity; tag `compound-severity: 2× Concern elevated to Block` |
| All Nits, or single-persona-multiple-findings | unchanged |

Annotate each finding with:
- `panel_severity` — post-elevation
- `compound_flag` — string explaining the elevation reason, or empty
- `multi_persona_agreement` — count of distinct personas in this group

### Output of Step 4

`findings` enriched with `panel_severity`, `compound_flag`, `multi_persona_agreement`, `merge_group_id`.

---

## Step 5 — stamp_origin

### Graceful degradation gate (concern #2 from Phase 1 review)

- If `origin_stamping_available == false` (no gh OR no PR ref): set `finding.origin = "unknown"` for all findings. Set `run_health.origin_stamping = "skipped (no PR ref or gh unavailable)"`. Skip to Step 6.

### Origin-stamping subsystem (gh-CLI based)

Iterate over each finding. For each:

1. Pull `evidence_tokens` from the finding.
2. For each commit in `pr_metadata.commits`:
   - Diff that commit (`gh api repos/<owner>/<repo>/commits/<sha> --jq '.files'`).
   - If any modified line in the commit contains a token from `evidence_tokens` → record commit as a candidate origin.
3. Classify the finding's origin by inspecting candidate commits and PR review history:
   - **`ai-reviewer-suggested`** — at least one candidate commit references (in body, title, or co-author) a known AI-reviewer bot identifier (`codex`, `claude`, `coderabbitai`, `pr-review-toolkit`, or any user with `[bot]` suffix). OR the change was made in response to a review comment from a bot user.
   - **`human-reviewer-suggested`** — at least one candidate commit references (in body, title, or `Co-authored-by:` trailer) a non-bot reviewer who submitted a PR review comment touching the same Evidence tokens. Most common: PM/lead inline comments.
   - **`implementer-authored`** — candidate commits exist, but no AI-reviewer or human-reviewer linkage found; change was introduced by the PR author in their initial push.
   - **`unknown`** — no candidate commits found (Evidence tokens don't match any modified line in the PR), OR gh CLI returned errors mid-stamping (see failure path below).

### Origin-stamping failure path (concern #2 from Phase 1 review)

If gh CLI calls fail partway (timeout, auth error, rate limit):
- Set `finding.origin = "unknown — origin-stamping failed (<error>)"` for any findings not yet stamped.
- Set `run_health.origin_stamping = "failed (<error>)"`.
- Continue to Step 6. Do not fail the run.

### Output of Step 5

`findings` enriched with `origin`. `run_health.origin_stamping` set to one of: `success`, `skipped (...)`, `failed (...)`.

---

## Step 6 — route_gaps + check_coverage_hole_threshold

### 6a. Route gaps

For each gap in `context_gaps`:

- **Case A (overlap):** Check if any finding in `findings` has Evidence-token overlap with the gap's `evidence_or_note`. If yes, resolution = `merged into <finding_id>`; append a "Related concern from {source_persona}: <gap details>" note to that finding. If no, resolution = `unresolved overlap — no matching finding`.
- **Case B (need-context):** resolution = `flagged for human reviewer`. No automatic resolution.
- **Case C (out-of-lane):** Check the target_dimension against owned-dimension table:

| Dimension | Owning persona |
|---|---|
| Q1 (filter policy) | data-trust |
| Q1(c) (Stone-vs-Opinion) | brand-voice |
| Q2 (polarity) | funnel-stage |
| Q3 (generic-vs-imposed) | brand-voice |
| Q5 (evidence tier) | data-trust |
| Q6 (funnel-stage attribution) | funnel-stage |
| Q7 (hypothesis echo) | hypothesis-driven |
| Q4 (decision class) | **NO OWNER — coverage hole** |
| anything else | **NO OWNER — coverage hole** |

For owned dimensions: if the owning persona did NOT produce a finding on the gap's Evidence, mark `unresolved — owner-persona did not catch`. If the owning persona DID produce a finding on related Evidence → mark `merged — owning persona already flagged`.

For NO-OWNER dimensions: mark `coverage hole`, append to `notes/coverage-hole-log.md`.

### 6b. Coverage-hole log append

Append one line per Case C `coverage hole` to `notes/coverage-hole-log.md`:

```
<run-date YYYY-MM-DD> | <pr-ref or "local-diff"> | <source-persona> | <target-dimension> | <evidence excerpt — first 100 chars>
```

Create the file with a frontmatter header on first append:

```markdown
---
title: Coverage-hole log
purpose: Append-only audit trail of Case C out-of-lane escalations pointing at dimensions no panel persona owns. Used by check_coverage_hole_threshold (Step 6c).
---
```

### 6c. Threshold check (Risk 3)

For each unique `target_dimension` in the log:
- Count rows where `run-date` is within the last 10 runs OR last 14 days (whichever is *shorter* — for early runs both numbers are small).
- If count ≥ 3 → emit a `pattern_alert` for this dimension.

### Output of Step 6

- `findings` further enriched (Case A merges)
- `routed_gaps` — list of gap records with resolution
- `pattern_alerts` — list of dimensions exceeding threshold

---

## Step 7 — render_markdown_panel_review

Emit the panel review using this exact structure (Risk 1c). Replace placeholders. Sort Findings by:
1. Panel-severity (Block first, then Concern, then Nit).
2. Within same severity, by `multi_persona_agreement` descending.
3. Within same agreement count, by persona order (1 → 4).

```markdown
# Panel Review — <pr_ref or local-diff identifier> — <YYYY-MM-DD>

## Summary

- Total findings: <N> across <K of 4> personas
- Panel-level severity counts: Block × <count>, Concern × <count>, Nit × <count>
- AI-reviewer-suggested findings: <count>
- No-findings sentinel personas: [<list>]
- Personas that failed to respond: [<list>]
<if pattern_alerts non-empty:>
- 🚨 **Coverage hole pattern alert(s):**
  - `<dimension>` flagged by Case C in <count> of last <window> runs ({<source-personas>}). Consider adding a dedicated persona or expanding existing lane.
<end if>

## Findings

<for each finding in sorted order:>

### Finding <id>: <persona's short title>

- **Panel severity:** <Block|Concern|Nit>
- **Persona severity:** <Block|Concern|Nit> (from <persona-id>)
- **Compound flags:** <compound_flag or "single-persona finding">
- **Multi-persona agreement:** <count> personas flagged related Evidence ({<persona-ids>})
- **Origin:** <implementer-authored | ai-reviewer-suggested | human-reviewer-suggested | unknown[ — <reason>]>
- **Evidence:** <verbatim from persona>
- **Framework citation:** <verbatim from persona>
- **Reasoning:** <verbatim from persona>
- **Issue:** <verbatim from persona>
- **Recommendation:** <verbatim from persona>
  <if related-concerns folded in:>
  *Related concerns from <other-persona>:* <other-persona's Recommendation>
  <end if>

<end for>

## Persona deliberation — Considered but not flagged

For audit transparency: elements personas evaluated in-lane but chose NOT to flag.

<for each persona that emitted a section:>

### <persona-id>

- `<element>` — <reason>
- ...

<end for>

(if no persona emitted this section, omit the entire `## Persona deliberation` section)

## Context-gap routing

| Case | Source persona | Target | Resolution |
|---|---|---|---|
<for each routed_gap:>
| <A overlap | B need-context | C out-of-lane> | <source_persona> | <target_dimension> | <resolution> |
<end for>

## Run health

- Persona dispatch:
  - brand-voice: <ok (<duration>) | timeout | malformed>
  - funnel-stage: <...>
  - hypothesis-driven: <...>
  - data-trust: <...>
- Origin-stamping: <success | skipped (<reason>) | failed (<reason>)>
- Coverage holes logged this run: <count>
```

### Special-case outputs

- **All-sentinel run:** emit only the Summary header line + `Panel: no in-scope findings across Q1 / Q2 / Q3 / Q1(c) / Q5 / Q6 / Q7.` + Run health section. Skip Findings + Context-gap sections entirely.
- **All-personas-failed run:** emit `Panel: all personas failed to respond. Re-run when infrastructure stable.` + Run health detail showing each persona's failure mode. Exit non-zero.

### Optional archive

If `archive` flag is set or user requests: write the rendered output to `eval-runs/<YYYY-MM-DD>-orchestrator-<pr-ref or local>.md` alongside this in-session display.

The archive MUST include a `## Raw subagent outputs` section near the top, linking to every raw file written in Step 2's preservation phase:

```markdown
## Raw subagent outputs

- [brand-voice run 1](raw/2026-05-17-pr006-brand-voice-run1.md)
- [brand-voice run 2](raw/2026-05-17-pr006-brand-voice-run2.md)
- [funnel-stage run 1](raw/2026-05-17-pr006-funnel-stage-run1.md)
- ...
```

This is a hard requirement, not optional. The archive is the builder's classification of the artifacts; the raw files are the artifacts. Both must be discoverable in the same place for independent review to be meaningful.

---

## Smoke-test attention points (concern #1 from Phase 1 review)

After first 3 real-PR runs, audit for these patterns. Surface findings to the panel-design layer:

1. **Semantic-overlap-without-token-overlap.** Count findings on the same entity/record where token-overlap heuristic correctly identifies them as NOT related but a human reviewer would group them. If this pattern occurs in ≥2 of first 3 runs → escalate to Phase 3 panel-summary clustering work.
2. **Case C noise rate.** Count Case C escalations per run. If average >2 per run → personas are over-escalating; tighten the "trust other personas" default in persona prompts.
3. **Origin-stamping precision.** Manually verify `ai-reviewer-suggested` and `human-reviewer-suggested` classifications on first 3 runs against actual PR review history. If miscategorization rate >20% → revisit the classification logic in Step 5.

These audits do NOT block orchestrator operation — they generate Phase 3 backlog items.
