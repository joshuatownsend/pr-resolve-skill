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

## `group_by(.user)` in step 1c is correct as written

**Decided:** 2026-08-12 · **Status:** closed

The PR-level reactions pipeline in step 1 of `skills/pr-resolve/SKILL.md` ends:

```
| group_by(.user) | map(max_by(.created_at))
```

**Decision: keep it as written. Do not add `sort_by(.user)`.**

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

### What would justify revisiting

- jq changing `group_by` to stop sorting internally — a breaking change to a
  documented builtin, so watch the jq changelog rather than a bot.
- `gh`'s embedded jq diverging from upstream jq here. Testable directly: re-run
  the transcript above and see whether `codex` comes back as two groups.
- The pipeline being ported to a different JSON tool whose `group_by` genuinely
  requires sorted input.

**Source:** §3.1 of `feedback/2026-08-12-field-report-pr-267.md`.
