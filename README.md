# assessing-a-codebase

A [Claude Code skill](https://docs.anthropic.com/en/docs/claude-code/skills) that produces a 4-file cold-start context package for any codebase — so every AI session starts already knowing the architecture, domain, and conventions instead of re-deriving them from scratch.

## What it produces

Running `/assessing-a-codebase` writes four files to `.claude/rules/`:

| File | Answers |
|---|---|
| `architecture.md` | What is this system and where does everything live? |
| `domain-glossary.md` | What do these domain terms mean — and what are the footguns? |
| `conventions.md` | How do we build things here? (auth, testing, placement, patterns) |
| `benchmark.md` | What did these files eliminate, and how do you verify they work? |

Each file has a hard 150-line limit. The goal is navigation, not encyclopedia — every line either prevents a mistake or replaces a question to a teammate.

## How it works

1. **Assess existing knowledge** — reads README, CLAUDE.md, existing rules files before acquiring anything new
2. **Deterministic analysis** — git history, schema grep, dependency manifest, PR descriptions
3. **Parallel agent dispatch** — three scoped agents for architecture, domain model, and conventions
4. **Expert interview** — asks targeted questions about footguns, confusing naming, and tacit knowledge
5. **Write memory files** — hot memory (`.claude/rules/`) with optional cold memory (`.claude/rules/references/`)
6. **Self-check** — line count, correctness, redundancy, signal
7. **Cleanup** — removes duplicated structural facts from CLAUDE.md

## Install

Copy `SKILL.md` into your Claude Code skills directory:

```bash
mkdir -p ~/.claude/skills/assessing-a-codebase
curl -o ~/.claude/skills/assessing-a-codebase/SKILL.md \
  https://raw.githubusercontent.com/MatthewJamisonJS/assessing-a-codebase/main/SKILL.md
```

Or clone the repo:

```bash
git clone https://github.com/MatthewJamisonJS/assessing-a-codebase ~/.claude/skills/assessing-a-codebase
```

## Usage

In any Claude Code session, from the root of a codebase:

```
/assessing-a-codebase
```

Or with an explicit path:

```
/assessing-a-codebase /path/to/repo
```

Claude will run the full analysis and write the rules files directly into the target repo's `.claude/rules/` directory.

## What makes this different from just asking Claude to document a codebase

A one-off "document this codebase" request produces output that disappears when the session ends. This skill produces **persistent, structured files** that Claude Code loads automatically at the start of every future session — so you never pay the cold-start cost again.

The `benchmark.md` file verifies this: it includes verification questions that should be answerable with zero tool calls in a fresh session.

## Evals

The `evals/` directory contains the test cases used to develop this skill, including assertions that verify the output quality. Pass rate: **83% with skill vs 20% without** across 3 test scenarios (warm-start, explicit-files, onboarding).
