# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repository is

A Claude Code **skill** repository. Alongside this file and `README.md`, it holds one skill:

- `pr-resolve/SKILL.md` — the `pr-resolve` skill: frontmatter declaring `name`, plus a `description` whose trigger phrases are what drive automatic invocation, followed by a numbered procedure the agent follows.

A skill is markdown, not executable code: YAML frontmatter declaring `name` and `description`, then the body as the instructions. The directory name matches the frontmatter `name`.

## No build, lint, or test tooling

There is no `package.json`, test runner, linter, or CI config. Do not look for npm scripts or a test command to run — none exist. Changes are validated by reading the skill and by invoking it against a real PR.

## Commands inside SKILL.md target *other* repositories

The commands in `pr-resolve/SKILL.md` (`gh pr view <num> --comments`, `npm run typecheck`, the test suite, `gh` push/merge steps) are steps the skill performs in whatever repository it is invoked from. They are **not** development commands for this repository, and `npm run typecheck` will fail here. When editing the skill, evaluate those commands against the target-repo context they run in, not against this repo.

## Editing the skill

The whole file is agent-facing prose — wording is behavior. When widening the `description`, prefer phrasings observed in real transcripts over invented ones. Keep steps imperative, ordered, and specific about what to verify (the existing body asks for an exact test pass count and gates merging on user approval). Preserve those verification and approval gates unless explicitly asked to change them, since they are what keeps the skill from merging unverified work.
