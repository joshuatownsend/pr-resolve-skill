# Decisions

Closed decisions, recorded so they stay closed.

The `pr-resolve` skill prescribes treating a re-raised declined finding as "a
known disagreement, not new information" (see *How many rounds* in
`skills/pr-resolve/SKILL.md`), but that rule needs somewhere to keep the
disagreement. A verdict that lives only in a PR thread is one the next agent
re-litigates, and then the one after that.

Each entry records the decision, when it was made, what was rejected, and what
new information would justify reopening it. An entry is closed, not permanent —
it holds until its stated condition is met.

---

## `null` needs no guard in the step 6 timestamp filters

**Decided:** 2026-08-12 · **Status:** closed

**Decision: do not add a null guard to `select(.submitted_at > $PUSH_TIME)`.**

### What was rejected

> `submitted_at` is null/missing for `PENDING` reviews [...] jq will error when
> comparing null to a string, which would break the polling loop on PRs with
> pending reviews present.

The premise is false — jq has a total ordering across types (`null` < `false` <
`true` < numbers < strings < arrays < objects), so the comparison yields `false`
rather than raising. Checked against `gh`'s embedded jq:

```console
$ gh api rate_limit --jq '[{"a":null},{"a":"2026-01-01T00:00:00Z"}]
                          | map({val: .a, gt: (.a > "2025-01-01T00:00:00Z")})'
[{"gt":false,"val":null},{"gt":true,"val":"2026-01-01T00:00:00Z"}]
$ echo $?
0
```

The null row is dropped, exit code 0, no error. That is also the **correct**
outcome on the merits: a `PENDING` review is one nobody has submitted, so it must
not count as a verdict. A guard would add ceremony to reach the same result.

### What would justify revisiting

- jq changing its cross-type ordering — re-run the command above and check for a
  non-zero exit.
- The filter being rewritten to do arithmetic or string operations on
  `submitted_at`, where `null` genuinely would raise.

**Origin:** raised by Copilot in a suppressed-comment block, 2026-08-12. This is
the third bot claim in one PR asserting false jq semantics, after `group_by`
adjacency and `--slurp`. **Test any jq-semantics claim against `gh`'s embedded jq
before acting on it** — all three took one command to disprove.

---

## Inline code spans may wrap across a newline

**Decided:** 2026-08-12 · **Status:** closed

**Decision: do not reflow prose to keep an inline code span on one line.**

### What was rejected

> The inline-code span for the example filter is split across a newline [...]
> GitHub-flavored Markdown doesn't render multi-line inline-code reliably, so
> this can show up as raw backticks and reduce readability.

The premise is false. CommonMark converts a line ending inside a code span to a
space, and GitHub implements it. Checked against GitHub's own renderer:

```console
$ gh api markdown -f mode=gfm -f text='A filter ending `group_by(.user) |
map(max_by(.created_at))` therefore returns a user once per page.'
<p>A filter ending <code class="notranslate">group_by(.user) | map(max_by(.created_at))</code> therefore returns a user once per page.</p>
```

One clean `<code>` element, no raw backticks. Nothing to fix.

### Why it matters more here than in most repos

`SKILL.md` is read by an agent as raw text far more often than it is rendered,
so hard-wrapping prose at a fixed width is the priority and a code span that
happens to straddle the wrap is not a defect. Reflowing to satisfy this would
also imply the constraint is real, which is the same trap as the `sort_by`
entry below.

### What would justify revisiting

- GitHub's renderer changing this behavior — re-run the command above and check
  whether the `<code>` element still comes back intact.
- The file gaining a consumer that renders it with a non-CommonMark parser.

**Origin:** raised by Copilot in a suppressed-comment block, 2026-08-12.

---

## `group_by(.user)` in step 1c is correct as written

**Decided:** 2026-08-12 · **Status:** closed

The PR-level reactions pipeline in step 1 of `skills/pr-resolve/SKILL.md` ended:

```
| group_by(.user) | map(max_by(.created_at))
```

**Decision: `group_by` needs no `sort_by(.user)`. Do not add one.**

The line itself was removed the same day, for an unrelated defect — see
*Amendment* below. The decision is recorded against the **claim**, not the line:
the claim is false wherever it is raised, and it has been raised twice.

### What was rejected

Copilot has raised this twice — once inline, then again in a round-2 review body
after the first was declined, the second phrasing sharper and equally wrong:

> In jq, `group_by` only groups adjacent items, so if reactions aren't already
> sorted by user this can yield multiple groups per user and the
> `max_by(.created_at)` result won't reliably be the latest reaction per user.

This is false. jq's `group_by` sorts internally — it is defined as
`_group_by_impl(map([f]))`, and the implementation sorts before it groups. "Only
groups adjacent items" describes `uniq`, which genuinely does require pre-sorted
input; that is the likely source of the confusion.

### Evidence

Run against `gh`'s embedded jq — the implementation the skill's commands actually
use — with one user deliberately interleaved, so an implementation that only
grouped adjacent items would split it into two groups:

```console
$ gh api rate_limit --jq '[{"user":"codex","created_at":"2026-01-01T00:00:00Z"},
                           {"user":"copilot","created_at":"2026-01-02T00:00:00Z"},
                           {"user":"codex","created_at":"2026-01-03T00:00:00Z"},
                           {"user":"aaa","created_at":"2026-01-04T00:00:00Z"}]
                          | group_by(.user)'
[[{"created_at":"2026-01-04T00:00:00Z","user":"aaa"}],
 [{"created_at":"2026-01-01T00:00:00Z","user":"codex"},
  {"created_at":"2026-01-03T00:00:00Z","user":"codex"}],
 [{"created_at":"2026-01-02T00:00:00Z","user":"copilot"}]]

$ # ...same input | group_by(.user) | map(max_by(.created_at))
[{"created_at":"2026-01-04T00:00:00Z","user":"aaa"},
 {"created_at":"2026-01-03T00:00:00Z","user":"codex"},
 {"created_at":"2026-01-02T00:00:00Z","user":"copilot"}]
```

Both `codex` entries grouped together despite being non-adjacent in the input,
and `max_by` returned the later one — precisely the behavior step 1c relies on.

### Why not add `sort_by` anyway

It would be harmless, and it would stop the finding recurring. It would also
assert a bug that does not exist: a defensive no-op tells every future reader the
pipeline is fragile, and invites the next editor to protect it further. The
skill's reaction handling is load-bearing enough that it should read as
deliberate.

### Amendment, 2026-08-12 — a real defect in the same line, found the same day

Codex raised a **different** claim against this pipeline, and was explicit that
the `sort_by` rejection above should stand. It was right, and the line is gone:

> `gh api --help` documents that with `--paginate`, "Each page is a separate JSON
> array or object" [...] so the embedded jq expression groups and selects a
> maximum independently on every page.

Verified — `--jq` runs once per page, emitting one result per page:

```console
$ gh api "repos/OWNER/NAME/commits?per_page=2" --paginate --jq 'length'
2
2
2
2
1
```

So `group_by(.user) | map(max_by(.created_at))` aggregated *per page*, and on a
PR with more than one page of reactions returned a user once per page — the
latest-per-user guarantee failing silently. Step 1c is now a streaming
per-item filter, with the picking left to the agent.

**The suggested fix does not work, and this is the part worth remembering.**
Codex proposed `--slurp`. `gh` rejects it outright alongside `--jq`:

```console
$ gh api "repos/OWNER/NAME/commits?per_page=2" --paginate --slurp --jq 'length'
the `--slurp` option is not supported with `--jq` or `--template`
```

and `--slurp` on its own returns pages still nested (`[[...],[...]]`), needing
`add` from a jq that is no longer in the pipeline. Piping to standalone `jq` is
not portable — it is absent on the maintainer's Windows machine. **Decline any
future finding that proposes `--slurp` with `--jq`; the incompatibility is
enforced by `gh`'s argument parser, not a matter of style.**

The lasting rule went into the skill itself: keep every `--jq` filter streaming,
because under `--paginate` any aggregating filter silently computes per page.

### What would justify revisiting

- jq changing `group_by` to stop sorting internally — a breaking change to a
  documented builtin, so watch the jq changelog rather than a bot.
- `gh`'s embedded jq diverging from upstream jq here. Testable directly: re-run
  the transcript above and see whether `codex` comes back as two groups.
- The pipeline being ported to a different JSON tool whose `group_by` genuinely
  requires sorted input.

**Origin:** raised by Copilot on a live PR and declined, twice, on 2026-08-12.
The field report covering that run is kept local rather than committed; the
evidence that matters is reproduced in full above, so this entry stands on its
own.
