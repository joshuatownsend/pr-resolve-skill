---
name: pr-resolve
description: Resolve pull request review comments from bot reviewers, verify, and merge. Use when the user says "resolve PR comments", "review PR comments", "address the PR feedback", "fix the review comments", "check for bot comments", "check again for bot findings", "any new comments on the PR", "did Codex/Copilot review it", "codex has commented", "another round of bot review", "merge once the bots are clean", or "merge on green".
---

Resolve every review finding on a PR, verify, and merge. Bot reviewers signal
findings and verdicts through three different channels — gather all three, every
round. `gh pr view --comments` alone is not sufficient and will show an empty
result while findings exist.

**Steps 1–2 only read. Step 3 onward mutates.** Several of the phrases that
invoke this skill are questions — "any new comments on the PR", "did Codex
review it", "check for bot comments". When the request is a question, gather,
report what you found, and stop. Go on to fix, verify, push, and comment only
once the user asks for the findings to be resolved, or has already authorized
it. A status question must never turn itself into a push.

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

### Empty is not clean

Nothing found and nobody heard from is not a clean PR — it is a PR no one has
looked at yet. Before treating any gather as clean, confirm each expected
reviewer has engaged with the **current** head: an inline comment, a submitted
review, or a reaction dated after the head push. Work out who to expect from the
reviewers already active on this PR, or from the most recently merged PR in the
repo when this is the first round.

If no reviewer has engaged with the current head, the round is not clean — it has
not happened yet. Wait and re-poll per step 6 instead of proceeding. Silence is
the one signal that never means approval.

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

**Default to fixing in this session.** Most findings arrive with a file, a line,
and a diagnosis — the expensive part is already done, and a fresh subagent would
only re-derive it. Delegation buys a clean context, not less effort; if the
diagnosis is already in the finding, it buys nothing.

Delegate only when the work would flood this session's context:

- diagnosis needs wide reading — tracing call paths, hunting usages, or reading
  files the finding does not name
- the finding is a symptom and the cause is not yet located
- the fix spans many files, or repeats mechanically across them

Subagents edit the working tree directly and report what changed. Review the
diff and keep commit responsibility here, so related fixes still group into a
single commit. Run delegations in parallel only when the fixes are expected to
touch disjoint files; serialize when unsure.

### Choosing the subagent model

Start at **Sonnet** — the right size for a bounded fix that needs real
exploration, which is most delegated work. Depart from it only for a reason:

- **Haiku** — bulk mechanical edits, fully specified, where volume rather than
  difficulty is why the work is leaving this session
- **Opus** — genuine design judgment, or subtle correctness and concurrency bugs
- **Fable** — only with explicit user permission; ask before spawning one

Escalate on evidence, not prediction — difficulty is hard to judge before the
work is attempted. A wrong or incomplete result retries **one tier up**, not
again at the same tier with a better prompt.

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

Applies after every push, not just the first — though *How many rounds* below
decides when blocking on the result is actually warranted. A verdict is one of
the following, **posted by someone other than you**:

- a new inline comment, **or**
- a submitted review, **or**
- a 👍 whose `created_at` is later than `PUSH_TIME`

and explicitly **not** 👀. ISO-8601 UTC timestamps compare correctly as plain
strings, so `created_at > $PUSH_TIME` is a valid test.

**Your own activity is not a verdict.** Replying to a review thread — which step
3 tells you to do when you decline a finding — creates a `COMMENTED` review *and*
an inline comment under your own login, both newer than `PUSH_TIME`. Read
literally, your reply satisfies the test above, and the loop exits believing a
reviewer responded when nothing but your own comment happened. Exclude yourself
by login on every channel, reactions included. Only you are excluded — a human
reviewer's comment or review is as much a verdict as a bot's.

Re-run step 1 to poll, then apply the exclusion:

```bash
ME=$(gh api user --jq .login)

gh api repos/{owner}/{repo}/pulls/<n>/reviews --paginate \
  --jq ".[] | select(.submitted_at > \"$PUSH_TIME\") | select(.user.login != \"$ME\")
        | {user: .user.login, state, submitted_at, body}"

gh api repos/{owner}/{repo}/pulls/<n>/comments --paginate \
  --jq ".[] | select(.created_at > \"$PUSH_TIME\") | select(.user.login != \"$ME\")
        | {user: .user.login, path, line, body}"
```

Keep `body` in both, for the reason step 1 gives. Where the expected reviewers
are all bots, `select(.user.type == "Bot")` is an equivalent filter —
`.user.type` is `"Bot"` for `copilot-pull-request-reviewer[bot]` and `"User"`
for humans — but it also drops human reviewers, so prefer the login exclusion
unless you have established the roster is bots only.

If a verdict brings new findings, return to step 3 and
repeat the loop. If no verdict arrives after a reasonable wait, report the wait
and ask the user how to proceed — do not treat silence as approval.

### Where each reviewer stands

Reviewers behave differently: some re-review on every push, others review once
per request and then stop. Do not model them by name — resolve each one's state
from the API:

- **Pending** — the reviewer appears in `reviewRequests`, or has a
  `review_requested` timeline event newer than its latest review. A review is
  queued; wait. (GitHub clears `reviewRequests` once the review lands, so the
  timeline is the durable record.)
- **In progress** — its latest reaction is 👀. Wait.
- **Fresh verdict** — a comment, review, or 👍 after `PUSH_TIME`. Consume it.
- **Done, not pending** — reviewed an older head, with no newer request. **It is
  not coming back on its own.** Never block on this state: a reviewer that acts
  only on request is finished, not slow. Blocking here hangs forever.

When a **substantive** round pushed changes after a once-per-request reviewer's
verdict, re-request that reviewer as part of the push — its stale verdict does
not cover the new code. After a cosmetic-only round, the stale verdict stands.

**Re-triggering Copilot requires GraphQL — the REST call silently does nothing.**
Copilot is a Bot, not a user, so `POST .../requested_reviewers` with
`reviewers: ["Copilot"]` returns **201 and discards the request**: no pending
entry, no timeline event, no review. It looks for a user of that name. Use:

```bash
gh api graphql -f query='mutation($pr:ID!,$bot:ID!){
  requestReviews(input:{pullRequestId:$pr, botIds:[$bot], union:true}){
    pullRequest{ reviewRequests(first:5){nodes{requestedReviewer{... on Bot{login}}}} } } }' \
  -f pr="<PR node id>" -f bot="<bot id>"
```

`union: true` keeps existing reviewers. Get the bot id from any earlier
`ReviewRequestedEvent` in the repo — query `timelineItems(itemTypes:
[REVIEW_REQUESTED_EVENT])` and read `requestedReviewer` — it is stable across
PRs. A real request shows up in `reviewRequests` *and* adds a timeline event;
if neither appears, it did not register. Confirm the outcome by watching for the
review itself, never by the call succeeding.

Codex re-reviews automatically on every push, and on a `@codex review` comment.

### How many rounds

Every push re-triggers the bots, so another round always happens — the only
decision is whether to block on it. Judge by what the round you just fixed
contained, since substantive edits are what introduce new defects:

- **Nothing, or cosmetic only** — docs, comments, naming, formatting: fix them,
  push, and proceed to step 7 without waiting. A gather that is clean or
  cosmetic-only from the start goes straight to step 7 — but only once it has
  passed *Empty is not clean* in step 1. An unreviewed PR is not a clean one.
- **Anything substantive** — correctness, security, data loss, API misuse, a
  test gap hiding a real bug: block on the next verdict and repeat from step 1.

Converged means the latest round raised no new **substantive** findings — not
that every remark was addressed. Declining a finding is legitimate; a bot that
re-raises something you declined is a known disagreement, not new information,
so treat it as converged and record it in the step 7 summary for the user to
arbitrate at the merge gate.

Stop and ask the user rather than pushing again when:

- a round re-raises something you already **fixed** — the fix did not take, and
  another round of the same will not help
- three substantive rounds pass without converging

Report what is still outstanding and let the user decide.

## 7. Summarize

Post a summary comment to the PR: findings resolved, source of each, and the
exact test pass count from step 4.

## 8. Merge

Requires the user's approval. A conditional grant already given — "merge once
the bots are clean", "merge on green", "merge once this round is clean" —
satisfies that gate, but only after its condition is **objectively verified**:
a converged round per step 6, CI actually green, whichever was named. Verify the
condition; never infer that it was met. If it is ambiguous, only partly met, or
rests on a signal you could not confirm, ask instead of merging.

Approval is scoped to one PR. A single session often takes several PRs through
this skill, and permission granted for one is not permission for the next.

Then squash-merge: `gh pr merge <n> --squash --delete-branch`.

**Never bypass branch protection or required reviews.** If the merge is blocked,
that is the protection working — report the specific blocker and stop. Approval
to merge is not approval to override the checks that approval was conditioned
on, and `--admin` is exactly the move that turns "merge on green" into a merge
on red. Use it only if the user, told what is blocking, explicitly asks for it.
