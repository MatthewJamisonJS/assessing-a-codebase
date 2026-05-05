# assessing-a-codebase

A [Claude Code skill](https://docs.anthropic.com/en/docs/claude-code/skills) that runs a one-time analysis of any codebase and writes persistent context files so every future session already knows the architecture, domain, and conventions — no re-deriving from scratch.

## The problem it solves

Every Claude Code session starts cold. Ask it something architectural and it reads half the repo to get oriented. Do that ten times a day and you're paying the same onboarding cost on repeat.

This skill runs once, writes three files to `.claude/rules/`, and Claude loads them automatically from then on. Session starts warm. Questions get direct answers.

## What it writes

```
.claude/rules/
├── architecture.md     # What is this system, where does everything live
├── domain-glossary.md  # What do the domain terms mean, what are the footguns
└── conventions.md      # How do we build things here
```

Hard cap: 200 lines per file. The goal is navigation, not documentation — every line either prevents a mistake or replaces a question to a teammate. If a topic needs more depth, it goes in `.claude/rules/references/` and gets a pointer.

## Staying warm across repo switches and git clean

`.claude/rules/` is gitignored — it's personal context, not team documentation. That means it disappears on a fresh clone, `git clean -fdx`, or `git stash --all`. The skill handles this: on first run it caches the rules to `~/.claude/projects/<repo>/context-cache/` and installs a `resurface` command in your shell (Fish, Zsh, or Bash — auto-detected, no manual config edits).

```bash
# rules files gone after git clean or switching machines?
resurface
# → restores .claude/rules/ from cache in seconds
```

`resurface` is git-root-aware. Run it from anywhere inside the repo. It never overwrites files you've locally edited.

## Install

```bash
mkdir -p ~/.claude/skills/assessing-a-codebase
curl -o ~/.claude/skills/assessing-a-codebase/SKILL.md \
  https://raw.githubusercontent.com/MatthewJamisonJS/assessing-a-codebase/main/SKILL.md
```

Or clone:

```bash
git clone https://github.com/MatthewJamisonJS/assessing-a-codebase ~/.claude/skills/assessing-a-codebase
```

## Usage

From the root of any repo:

```
/assessing-a-codebase
```

Or point it at a path:

```
/assessing-a-codebase /path/to/repo
```

The full run takes 10–20 minutes on a large codebase. Run it once, get `resurface` for free, and you're done.

## How it works

The skill works in ten steps:

1. **Check existing knowledge** — reads README, CLAUDE.md, any existing rules before touching anything
2. **Deterministic analysis** — git history, schema grep, dependency manifest, recent PR descriptions
3. **Three parallel agents** — architecture & structure, domain model, conventions in practice
4. **Expert interview** — asks targeted questions about footguns, confusing naming, tacit knowledge the code won't show
5. **Write the rules files** — architecture, domain-glossary, conventions (≤200 lines each)
6. **Quality audit** — three parallel agents checking correctness, signal, and redundancy
7. **Benchmark** — fresh session, four questions, should pass with zero tool calls
8. **Cleanup** — removes duplicated structural facts from CLAUDE.md
9. **Resurfacing setup** — caches rules, detects shell, installs `resurface`
10. **Maintenance** — targeted update process for when the codebase changes

## Evals

`evals/` has the test cases used to develop this skill. Pass rate: **83% with skill vs 20% without** across warm-start, explicit-files, and onboarding scenarios.
