---
title: Origin-stamping subsystem spec
status: v0.1
created: 2026-05-16
addresses: concerns #2 + #3 from Phase 1 approval (failure path + origin assignment logic explicit)
used-by: orchestrator/orchestrator-prompt.md Step 5
gh-cli-dependency: required
---

# Origin-stamping subsystem

This subsystem assigns each panel finding an `origin` tag indicating who introduced the change the finding flags. Output classes:

| Class | Meaning |
|---|---|
| `implementer-authored` | The PR author introduced this change in their initial push. No reviewer suggestion preceded it. |
| `ai-reviewer-suggested` | An AI code reviewer (Codex, pr-review-toolkit, claude-bot, coderabbitai, etc.) suggested the change in a PR review comment; the PR author then accepted/applied it. |
| `human-reviewer-suggested` | A human reviewer (PM, lead, peer) suggested the change in a PR review comment; the PR author then accepted/applied it. |
| `unknown` | Origin could not be determined — either Evidence tokens didn't match any modified line in the PR, OR origin-stamping failed mid-run (gh CLI errors, rate limit, auth failure). The reason is appended to the class string: `unknown — origin-stamping failed (auth)`. |

The high-leverage class is `ai-reviewer-suggested`: a panel finding tagged this way means the panel caught a bug that an AI reviewer would have introduced. These are surfaced prominently in the panel summary.

---

## Inputs to the subsystem

From upstream orchestrator state (after Step 4):

- `findings` — list of finding records, each with `evidence_tokens` (set of identifier tokens, see regex-patterns.md §4) and `evidence_file` (normalized path).
- `pr_metadata` — JSON object from `gh pr view --json title,body,headRefName,baseRefName,reviewDecision,reviews,comments,commits`. Specifically the `commits`, `reviews`, and `comments` arrays.
- `origin_stamping_available` — bool. If false, subsystem is skipped (per orchestrator Step 5 graceful degradation).

---

## Origin assignment logic (concern #3 explicit spec)

For each finding, the subsystem runs these checks in order. **First match wins.**

### Check 1 — Find candidate commits

A candidate commit is any commit in `pr_metadata.commits` whose modified-line set contains at least one of the finding's `evidence_tokens` (using the same tokenization rules as Evidence parsing — see regex-patterns.md §4).

Fetch the commit's modified lines via:
```bash
gh api repos/<owner>/<repo>/commits/<sha> --jq '.files[] | {filename, patch}'
```

For each file's `patch` (unified-diff text), tokenize the added/modified lines (`+` lines, excluding the `+` itself and excluding `+++` header) and check for token-set intersection with `finding.evidence_tokens`.

If no candidate commits: **origin = `unknown`** (Evidence tokens don't match any modified line in this PR's commit history). Stop.

### Check 2 — AI-reviewer linkage

For each candidate commit, check whether the commit was made *in response to* an AI-reviewer suggestion. Heuristics, evaluated as OR (any match → AI-reviewer-suggested):

- **Commit body / message references an AI reviewer.** Search the commit's full message body for case-insensitive matches of: `codex`, `coderabbit`, `pr-review-toolkit`, `claude-bot`, `copilot-bot`, `[bot]`. If found → AI-reviewer-suggested. Surface the matched bot name in the origin tag: `ai-reviewer-suggested (codex)`.
- **Co-authored-by trailer points at a bot.** Search for `Co-authored-by: ...<...[bot]@...>` or `Co-authored-by: ...codex...` patterns. If found → AI-reviewer-suggested.
- **Commit replies to a bot review comment.** Cross-reference `pr_metadata.comments` and `pr_metadata.reviews`: find any comment/review where `author.login` matches a bot pattern (suffix `[bot]`, or login containing `codex`, `claude`, `coderabbit`, `copilot`) AND whose body text contains at least one of the finding's `evidence_tokens`. If such a comment exists AND its timestamp precedes the candidate commit's timestamp → AI-reviewer-suggested.

If any heuristic fires: **origin = `ai-reviewer-suggested`** (with bot name if available). Stop.

### Check 3 — Human-reviewer linkage

Same approach as Check 2 but for human reviewers:

- **Commit body / message references a human reviewer by login or full name.** Search `pr_metadata.reviews` and `pr_metadata.comments` for non-bot reviewers (any author whose login does NOT match a bot pattern). For each such reviewer's login + display name, check whether the candidate commit's message body contains that login or name. If found → human-reviewer-suggested with the reviewer's login: `human-reviewer-suggested (corne)`.
- **Co-authored-by trailer points at a non-bot reviewer who left a review comment.** If `Co-authored-by:` matches a reviewer login → human-reviewer-suggested.
- **Commit responds to a human review comment.** Find any review/comment from a non-bot user whose body contains at least one of `finding.evidence_tokens` AND whose timestamp precedes the candidate commit. If found → human-reviewer-suggested.

If any heuristic fires: **origin = `human-reviewer-suggested`** (with login if available). Stop.

### Check 4 — Fallback to implementer-authored

If candidate commits exist (Check 1 passed) but neither AI-reviewer nor human-reviewer linkage was found (Checks 2 and 3 failed): the change was introduced by the PR author without a reviewer prompting it. **origin = `implementer-authored`**.

---

## Worked example — PR #5 MonetaryValue rename

For the canonical Brand-Voice Finding 1 against pr-005-value-monetaryvalue:

- `evidence_tokens` includes `monetaryvalue`, `monetary`, `value`, `goalresult`.
- Walk `pr_metadata.commits`:
  - The commit that introduced `MonetaryValue: goal.Value` (Story 03 initial push commit, e.g., `97a1ec9 feat: add engage_get_goals AI tool`) — tokens `monetaryvalue`, `monetary`, `value`, `goalresult` all appear in modified lines. **Candidate.**
- Check 2 — AI-reviewer linkage:
  - Commit message body: scan for `codex`, `[bot]`, etc. — search the PR review history for prior comments from codex or another bot.
  - If Codex left a review comment suggesting `Value → MonetaryValue` *before* commit `97a1ec9`, AND the comment body contains `MonetaryValue` or `Value` — match. **origin = `ai-reviewer-suggested (codex)`**.
- If the Codex suggestion was inlined in a thread that the orchestrator can fetch via `gh pr view --json comments`, the timestamp check confirms order.

This is the highest-leverage origin class: the panel catches a bug an AI reviewer would have shipped if not for downstream human review.

For the IsInverted + IsInvalid findings, candidate commits are the same Story 03 initial push (which OMITTED both fields). No prior bot suggestion existed for the omission (the omission was a copy-paste, not a positive suggestion). Therefore: **origin = `implementer-authored`** for both. The PR review flagged them post-hoc; the change to add them is in commit `d483e05`, which is the fix, not the originating commit.

---

## Failure path (concern #2 explicit spec)

If any `gh` CLI invocation fails during stamping (non-zero exit, timeout, JSON parse error, rate limit):

1. Mark all findings not yet stamped (those processed after the failure point) as `origin = "unknown — origin-stamping failed (<error-class>)"`.
2. Mark findings stamped before the failure with their assigned origin (preserve partial work).
3. Set `run_health.origin_stamping = "failed (<error-class>) — N of M findings stamped before failure"`.
4. Do NOT fail the run. Continue to Step 6.

Error classes to surface:

- `auth` — gh CLI returned `gh: To authenticate, please run …`
- `rate-limit` — gh CLI returned `API rate limit exceeded`
- `network` — gh CLI returned connection-timeout or DNS errors
- `json-parse` — `gh ... --json ...` produced non-parseable output
- `not-found` — PR or commit not found (404)
- `permission` — repository or PR access denied (403)
- `other` — any unclassified gh CLI error

Surface the error class in the per-finding origin string AND the run_health line. Marketer reviewers reading the panel output should be able to see at a glance that origin data is unreliable for this run and decide whether to manually verify the highest-severity findings.

---

## Caching strategy (Phase 2 implementation specific)

Per-run cache of:
- `gh pr view ... --json ...` output (one call per run)
- `gh api repos/.../commits/<sha>` output (one call per candidate commit, but check candidates lazily — many findings will share candidate commits, so memoize by SHA)

In-session memoization is sufficient for v1. Persistent cache (across runs) is Phase 3 optimization if real-PR runs show repeated identical PRs (unlikely — each PR is fresh).

---

## Known limitations (Phase 2.5 backlog)

1. **Bot identifier list is hardcoded.** `codex`, `coderabbit`, `pr-review-toolkit`, `claude-bot`, `copilot-bot`, `[bot]` suffix. New AI reviewers (or rebranded ones) won't be detected until the list is updated. Mitigation: maintain `state/known-ai-reviewers.md` once smoke-test data shows real-world distribution.
2. **Token overlap is heuristic, not semantic.** Two unrelated commits that happen to mention the same token will both flag as candidates. Manual audit in first 3 runs (per smoke-test attention point #3 in orchestrator-prompt.md) is the verification path.
3. **Timestamp ordering relies on author timestamps.** If a reviewer leaves a suggestion and the PR author rebases/cherry-picks, timestamps may shift. Mitigation: also check `committed_date` and prefer the later of the two.
4. **No support for inline suggestion comments with the `suggestion` markdown block** (`gh`'s GitHub-specific "Apply suggestion" feature). These produce explicit commits but the comment body may not contain the changed code's tokens — the suggestion block contains them in a separate field. Phase 2.5 enhancement: parse `suggestions` arrays in comment payloads if `gh pr view --json reviews` exposes them.

---

## Test plan (deferred to Phase 3 smoke test)

Validate origin assignment on real PR #5 first:

1. Run orchestrator against PR #5.
2. Manually compare assigned origins to ground truth:
   - MonetaryValue findings → expect `ai-reviewer-suggested` (Codex round-1 suggestion is documented in `eval-inputs/pr-005-value-monetaryvalue.md` line 31).
   - IsInverted + IsInvalid findings → expect `implementer-authored` (omissions, no prior bot suggestion).
3. If miscategorization rate is high (>20%): tighten heuristics before second smoke-test PR.

If PR #5 itself is the smoke-test target, the eval-inputs already document expected origin per finding — origin-stamping validation can run against this ground truth.
