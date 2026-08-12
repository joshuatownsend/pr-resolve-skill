# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repository is

A Claude Code **skill** repository. Alongside this file, `README.md`, `DECISIONS.md`, and the card in `assets/`, it holds one skill:

- `skills/pr-resolve/SKILL.md` — the `pr-resolve` skill: frontmatter declaring `name`, plus a `description` whose trigger phrases are what drive automatic invocation, followed by a numbered procedure the agent follows.

A skill is markdown, not executable code: YAML frontmatter declaring `name` and `description`, then the body as the instructions. The directory name matches the frontmatter `name`.

## It is also a plugin and its own marketplace

The repo root doubles as the plugin, and `.claude-plugin/marketplace.json` lists it with `"source": "./"`. That self-reference is deliberate and verified — `claude plugin validate` resolves it — so do not "fix" it into a `plugins/<name>/` subdirectory without re-validating. This matches the layout of the sibling `transcript-review-skill` repo; keep them consistent.

Validate manifests with `claude plugin validate .claude-plugin/marketplace.json --strict`. The plugin manifest emits one warning — that `CLAUDE.md` at the plugin root is not shipped as context — which is expected and correct to ignore: this file is contributor guidance, not something plugin users should receive. Because of it, `--strict` fails on the plugin manifest alone; validate the marketplace manifest strictly instead.

Three files open with the same one-sentence summary — the skill's frontmatter, `plugin.json`, and the marketplace entry — and then diverge: the frontmatter appends the trigger phrases that drive invocation, and the marketplace entry adds a second sentence of detail. Keep that shared opening sentence in sync; do not try to make the whole fields identical.

## No build, lint, or test tooling

There is no `package.json`, test runner, linter, or CI config. Do not look for npm scripts or a test command to run — none exist. Changes are validated by reading the skill and by invoking it against a real PR.

## Commands inside SKILL.md target *other* repositories

The commands in `skills/pr-resolve/SKILL.md` (`gh pr view <num> --comments`, `npm run typecheck`, the test suite, `gh` push/merge steps) are steps the skill performs in whatever repository it is invoked from. They are **not** development commands for this repository, and `npm run typecheck` will fail here. When editing the skill, evaluate those commands against the target-repo context they run in, not against this repo.

## Editing the skill

The whole file is agent-facing prose — wording is behavior. When widening the `description`, prefer phrasings observed in real transcripts over invented ones. Keep steps imperative, ordered, and specific about what to verify (the existing body asks for an exact test pass count and gates merging on user approval). Preserve those verification and approval gates unless explicitly asked to change them, since they are what keeps the skill from merging unverified work.

**Check `DECISIONS.md` before acting on a review finding against this repo, especially one a bot has raised before.** It records closed decisions — what was rejected, and what new information would justify reopening it. Some findings here are wrong and recur; the file exists so they get declined once rather than every time. When you decline a finding on grounds that will apply again, add an entry.
