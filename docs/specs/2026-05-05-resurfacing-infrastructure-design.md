# Resurfacing Infrastructure Design

**Date:** 2026-05-05
**Status:** Implemented in feat/resurfacing-infrastructure

## Context

`assessing-a-codebase` creates `.claude/rules/` files that prime Claude with codebase knowledge. These files are gitignored (not committed) and disappear silently on `git stash`, branch checkout, or repo switch — requiring a full re-assessment to recover them.

This design adds Step 9: Resurfacing Infrastructure, which solves this by caching rules files outside the working tree and installing a git-aware `resurface` shell command.

## Architecture Overview

Two new capabilities added at the end of assessment:

**Cache write (Step 9, assessment-time):**
- Derives project slug: `git rev-parse --show-toplevel | tr '/' '-'` — same derivation as Claude Code's own `~/.claude/projects/<slug>/memory/`
- Adds `.claude/rules/` to `.gitignore`
- Rsyncs `.claude/rules/` → `~/.claude/projects/<slug>/context-cache/`

**Shell function install (Step 9, assessment-time):**
- Detects shell via `basename "$SHELL"` → fish | zsh | bash
- Fish: writes `~/.config/fish/functions/resurface.fish` (safe overwrite — functions are separate files)
- Zsh/Bash: appends to rc file between sentinel comments; grep guard prevents duplicates
- Function is git-root-aware: auto-derives slug, rsyncs cache → working dir with `--ignore-existing`

## Key Design Decisions

**Slug derivation aligns with Claude Code:** `~/.claude/projects/<slug>/` is how Claude Code itself organizes per-project data. Using the same derivation means the `context-cache/` lives alongside auto memory — discoverable and consistent.

**`--ignore-existing` on pull:** `resurface` never overwrites locally-modified rules files. The cache is a restore point, not a source of authority over local edits.

**Sentinel comments on Bash/Zsh:** Running Step 9 twice should not duplicate the function block. Sentinel `# >>> resurface (claude-code) >>>` / `# <<< resurface (claude-code) <<<` lets the install script check before appending.

**Fish is a full overwrite:** Fish function files are one function per file. Overwriting is safe and idempotent.

## BDD Acceptance Scenarios

**Scenario 1 — First-time assessment:**
- Given: running assessing-a-codebase on `/Users/me/Code/my-app`
- When: Step 9 completes
- Then: `.claude/rules/` in `.gitignore`, cache at `~/.claude/projects/-Users-me-Code-my-app/context-cache/`, `resurface` available in terminal

**Scenario 2 — Rules lost after `git stash`:**
- Given: `.claude/rules/` was stashed
- When: developer runs `resurface` from anywhere inside the repo
- Then: missing rules files restored from cache; locally-modified files untouched (`--ignore-existing`)

**Scenario 3 — Multi-repo switching:**
- Given: `members` and `finance-app` both assessed
- When: `resurface` runs from inside `finance-app`
- Then: `finance-app`'s rules restored (different slug, different cache); `members` cache untouched

**Scenario 4 — No cache exists:**
- When: `resurface` runs before any assessment
- Then: `"resurface: no cache found for -Users-me-Code-my-app — run assessing-a-codebase first"`

**Scenario 5 — Shell detection:**
- Given: `$SHELL = /opt/homebrew/bin/fish`
- Then: `~/.config/fish/functions/resurface.fish` created/updated; no changes to zshrc/bashrc

**Scenario 6 — Local edits preserved:**
- Given: developer has locally modified `conventions.md`
- When: `resurface` runs
- Then: `conventions.md` NOT overwritten

**Scenario 7 — Idempotent install:**
- Given: Step 9 already ran once (sentinel exists in `~/.zshrc`)
- When: Step 9 runs again
- Then: no duplicate function block, no duplicate gitignore entry

## Anthropic Docs Findings (synthesized into content upgrades)

Source: https://code.claude.com/docs/en/memory

- **200-line target per file** — longer files reduce adherence per Anthropic docs → added to Step 5
- **`paths` frontmatter** — rules files can be scoped to specific file paths for conditional loading → added to Step 5
- **`~/.claude/rules/`** — user-level rules load before project rules, across all projects → added to Step 5
- **`.claude/rules/` should be gitignored** — these are personal AI-generated context, not team docs → added to Step 8
- **Project slug derivation** — Claude Code derives `~/.claude/projects/<slug>/` from git root; our cache uses the same pattern for consistency
