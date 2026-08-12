---
name: pr-resolve
description: Resolve all PR review comments, verify, and merge
---

Resolve every review finding on a PR, verify, and merge. Bot reviewers signal
findings and verdicts through three different channels — gather all three, every
round. `gh pr view --comments` alone is not sufficient and will show an empty
result while findings exist.

## 1. Gather findings from all three sources

Run all three. `--paginate` matters: a busy PR silently truncates otherwise.

```bash
# a. Inline review comments
gh api repos/{owner}/{repo}/pulls/<n>/comments --paginate \
  --jq '.[] | {user: .user.login, path, line, body}'

# b. Review bodies — findings hide in <details> blocks even when the review
#    summary claims it generated nothing
gh api repos/{owner}/{repo}/pulls/<n>/reviews --paginate \
  --jq '.[] | {user: .user.login, state, submitted_at, body}'

# c. PR-level reactions (note: issues/, not pulls/ — reactions are issue-scoped)
gh api repos/{owner}/{repo}/issues/<n>/reactions --paginate \
  --jq '[.[] | {user: .user.login, content, created_at}]
        | group_by(.user) | map(max_by(.created_at))'
```

**A Copilot review saying "generated no new comments" is not a clean verdict.**
Read the full review body regardless — findings are routinely suppressed inside
`<details>` blocks while the summary line reports nothing.

## 2. Reaction semantics

- 👀 (`eyes`) = **currently reading**. Not a result. Keep waiting.
- 👍 (`+1`) = **reviewed, no suggestions**. This is the clean verdict.

Codex **swaps** its reaction rather than adding one — the 👍 from the previous
round disappears when 👀 appears. So never ask "has this bot ever thumbs-upped
this PR." Ask: **is this bot's latest reaction a 👍, and is it newer than my most
recent push?** A stale 👍 from an earlier commit is indistinguishable from a
fresh one except by timestamp. The `group_by | max_by` in step 1c returns only
the latest reaction per user, which is the one to test.

## 3. Fix

Fix each finding using a subagent, grouping related fixes into a single commit.

## 4. Verify

Run `npm run typecheck` and the full test suite. Report the exact pass count.

## 5. Push, and record the push time

There is no push timestamp in the GitHub API — capture it locally, at the moment
you push. This value is the freshness floor for step 6.

```bash
git push && PUSH_TIME=$(date -u +%Y-%m-%dT%H:%M:%SZ) && echo "$PUSH_TIME"
```

Fallback if the push time was not captured: the HEAD commit's `committedDate`
(`gh pr view <n> --json commits`). This floor may be early if the commit was
authored well before it was pushed — prefer the captured value.

## 6. Wait for a fresh verdict

After **every** push, not just the first. A verdict is one of:

- a new inline comment, **or**
- a submitted review, **or**
- a 👍 whose `created_at` is later than `PUSH_TIME`

and explicitly **not** 👀. ISO-8601 UTC timestamps compare correctly as plain
strings, so `created_at > $PUSH_TIME` is a valid test.

Re-run step 1 to poll. If a verdict brings new findings, return to step 3 and
repeat the whole loop. Only proceed once every reviewer has issued a fresh, clean
verdict. If no verdict arrives after a reasonable wait, report the wait and ask
the user how to proceed — do not treat silence as approval.

**Do not use a re-review request as a completion signal.** `POST
repos/{owner}/{repo}/pulls/<n>/requested_reviewers` returns 200 and leaves
`requested_reviewers` empty for the Copilot bot, so a successful response does
not confirm a review was queued.

## 7. Summarize

Post a summary comment to the PR: findings resolved, source of each, and the
exact test pass count from step 4.

## 8. Merge

Only if the user approves: squash-merge, using `--admin` if branch protection
blocks.
