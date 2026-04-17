---
name: assessing-a-codebase
description: Creates .claude/rules/ context files (architecture.md, domain-glossary.md, conventions.md) that persist codebase knowledge across AI sessions. Use this skill whenever someone asks to: create rules files, set up .claude/rules/, make AI sessions "arrive warm", document the repo for AI, set up persistent/working context, stop sessions from re-deriving architecture, or reduce session onboarding overhead. Also use when onboarding to a new codebase and wanting permanent context files so future sessions start informed. Do NOT use for: one-time codebase analysis, answering questions about specific features, updating existing rules files, or writing a README.
---

## Invocation

Invoke as `/assessing-a-codebase` (uses current directory) or `/assessing-a-codebase /path/to/repo`.

**Target path: `$ARGUMENTS`** — if empty, use the current working directory for all steps. If a path was provided, all commands, agent dispatches, and file writes resolve against it.

---

## Core Principle

Acquire the minimum context needed to understand a codebase from five irreducible perspectives — structural, historical, domain, behavioral, and operational — using the cheapest available source for each gap.

The goal is **tiered persistent memory**:

- `.claude/rules/` — **hot memory**: concise navigation files always loaded at session start. Scannable in seconds. **Hard limit: 150 lines per file.** Files over this limit get ignored.
- `.claude/rules/references/` — **cold memory**: on-demand deep specs for complex subsystems. Only created when a topic is too large for a hot-memory pointer.

A session loading these files should answer any architectural, domain, or conventions question with zero tool calls. The benchmark.md written in Step 5 is the proof.

---

## Step 1: Assess Existing Knowledge (Free)

Check what already exists before acquiring anything:

```bash
ls .claude/rules/ 2>/dev/null
cat README.md CLAUDE.md 2>/dev/null
find . -maxdepth 3 -type d \( -name "adr" -o -name "decisions" -o -name "rfcs" -o -name "proposals" -o -name "docs" \) 2>/dev/null
```

Read everything you find. Map which perspectives are already covered vs. missing. Only proceed with acquisition for what's missing.

---

## Edge Cases — Resolve Before Step 2

Run these checks. They determine which parts of Step 2 are safe to execute.

**No git history or non-git repo:**
```bash
git log --oneline -5 2>/dev/null || echo "NO_GIT"
```
If `NO_GIT`: skip all `git log`, `git shortlog`, and `xargs git diff-tree` commands in Step 2. Point agents at source files directly instead of most-changed-file lists.

**Shallow git history (fewer than 20 commits):**
```bash
git rev-list --count HEAD 2>/dev/null
```
If < 20: skip most-changed-files analysis. Focus domain and conventions agents on entry-point files instead.

**Monorepo:**
```bash
ls packages/ apps/ services/ 2>/dev/null
```
If multiple sub-packages exist: scope all steps to the relevant sub-package path. Note cross-package dependencies explicitly in `architecture.md`.

**No schema file:**
```bash
find . -name "structure.sql" -o -name "schema.rb" -o -name "*.prisma" -o -name "schema.sql" 2>/dev/null | head -3
```
If nothing found: skip schema grep in Step 2. Domain model knowledge comes from model/entity source files — flag this in `domain-glossary.md`.

---

## Step 2: Deterministic Analysis (Near-Zero Cost)

These steps are mechanical. Run the ones that apply given the edge-case checks above.

### Critical path identification

```bash
# Confirm depth before running — requires at least 20 commits
git log --oneline -100 | wc -l
git log --format="%H" | head -100 | xargs -I{} git diff-tree --no-commit-id -r --name-only {} | sort | uniq -c | sort -rn | head -30
```

The top files are where tacit knowledge lives.

### Commit rhythm, style, and dead code signals

```bash
git log --oneline -60          # active areas, rhythm, message format
git shortlog -sn --no-merges   # who built what
git log --oneline --after="6 months ago" -- <top-file-from-above>

# Dead code / abandoned paths: find files/dirs not touched in 12+ months
git log --after="$(date -v-12m +%Y-%m-%d 2>/dev/null || date -d '12 months ago' +%Y-%m-%d)" \
  --name-only --pretty=format: | grep -v '^$' | sort -u > /tmp/active_files.txt
# Any top-level directory NOT in active_files is a dead code candidate
```

### Schema facts (authoritative — never infer from application code)

```bash
# Run on whatever schema file was found in the edge-case check
grep -rE "(deleted_at|destroyed_at|archived_at|discarded_at)" <schema-file>  # soft-delete
grep -rE '"type"' <schema-file>                                               # STI / discriminators
grep -rE "(tenant_id|org_id|account_id|workspace_id)" <schema-file>          # tenancy
grep -rE '"[a-z_]+_type"' <schema-file>                                       # polymorphic
```

### Dependency manifest

Read the lock/manifest file (Gemfile, package.json, go.mod, requirements.txt, Cargo.toml). Extract: auth stack, job system, search layer, ORM, any private/internal packages not on public registries.

### Written intent (highest-signal, lowest-cost)

```bash
# ADRs and proposals — design intent before implementation erases it
find . -type d \( -name "adr" -o -name "decisions" -o -name "proposals" -o -name "rfcs" \) 2>/dev/null

# Recent PR descriptions — tradeoffs and rejected alternatives
gh pr list --state merged --limit 15 --json number,title,body 2>/dev/null

# Changelog — the project's own narrative
cat CHANGELOG.md 2>/dev/null | head -120
```

### Environment and linting

```bash
cat .env.example .env.sample 2>/dev/null
cat .rubocop.yml .eslintrc* pyproject.toml 2>/dev/null | head -80
ls Procfile docker-compose.yml fly.toml .github/workflows/ 2>/dev/null
```

---

## Step 3: Parallel Agent Dispatch

Dispatch all three agents **in the same message**. Each follows a scoped pattern: identify structure → targeted reads → report findings. Keep agents tightly scoped — unlimited file exploration fills context without proportional gain.

**Agent 1 — Architecture & Structure**

```
In the codebase at [resolved target path from $ARGUMENTS or cwd]:

Phase 1 — Identify structure: list the top-level directories and the test
directory structure. What is the namespace or routing configuration?

Phase 2 — Targeted reads: read the dependency manifest, the main router or
routes config, and one representative controller/handler. Find:
- All active vs. legacy frontend tiers (which is touched, which is never touched)
- Background job system (primary vs. legacy) and which directories each uses
- Auth and authorization stack — each library's role
- Service/command/interactor layer if it exists — location and decision criteria
- Internal or private packages — what does each do?
- Multi-tenancy: how is org/tenant scoping enforced and at which layer?
- Dead code: any top-level directories, files, or packages that appear abandoned
  (no commits in 12+ months, or explicitly replaced by something newer)

Phase 3 — Return: flat list with directory paths and specific findings.
No prose. No speculation beyond what the files show.
Flag dead code explicitly with the label DEAD CODE.
```

**Agent 2 — Domain Model**

```
In the codebase at [resolved target path from $ARGUMENTS or cwd]:

Phase 1 — Identify the domain layer: find all model/entity files.

Phase 2 — Targeted reads: read the [N most-changed model files from git analysis,
or entry-point model files if no git history] and the central domain model. Find:
- The central entity (what everything else relates to)
- Any entity with a non-obvious table name (STI, shared tables — name the actual table)
- Entities with BOTH a soft-delete column AND an archive column (these serve distinct purposes — document both)
- Lifecycle callbacks that auto-create other entities
- Delegation or forwarding methods that change which entity is "active" for an operation
- Cross-module dependencies: which modules depend on which others in non-obvious ways?
- How does tenant/org scoping propagate through the domain layer?
- FOOTGUNS: patterns that look correct but silently break something — e.g., using a
  library-internal column for app logic, calling a method that bypasses callbacks,
  accessing JSONB directly vs. via an accessor. For each: name it, show the wrong
  pattern vs. the right one.

Phase 3 — Return: entity names, file paths, and specifically what is non-obvious
about each. Skip obvious things. Mark footguns with FOOTGUN label.
```

**Agent 3 — Conventions in Practice**

```
In the codebase at [resolved target path from $ARGUMENTS or cwd]:

Phase 1 — Identify pattern sources: the linting config, one representative
file from each of: controllers, models, views/templates, tests.

Phase 2 — Read 8-10 files (prioritize the most-changed files from this list:
[top files from git analysis, or entry-point files if no git history]). Find:
- What required base classes, includes, or mixins appear in every [model/controller/test]?
- How is authorization checked in practice? Show file:line of the CORRECT pattern and
  the WRONG pattern that looks similar (the subtle difference is what new devs get wrong).
- How are bulk operations written? Show file:line example.
- What cross-module boundaries exist that aren't obvious from the directory structure?
- What test helpers exist? What directories hold what test types?
- What commit message format does git log show?
- Top 3 "looks right but silently breaks" patterns — things the codebase consistently
  guards against (update_column, direct hash access instead of accessor, etc.).

Phase 3 — Return: findings as file:line — what it shows. No prose.
Flag anti-patterns that repeat consistently (they're probably canonical here, not mistakes).
```

---

## Step 4: Expert Interview (Tacit Knowledge Only)

**When available:** Read all agent findings before asking. Interview only what the artifacts couldn't answer.

Ask:
- "What 3–5 domain terms regularly confuse people new to this codebase?"
- "What looks like an anti-pattern but is the right call here — and why?"
- "What would you warn someone not to do — things that look right but silently break something?"
- "Is anything named one thing but architecturally another?" (shared tables, STI, polymorphic behavior that doesn't match naming)
- For top-changed files from git analysis: "Why does [file] change so frequently?"

**When no human is available:** Mine PR descriptions and commit messages for the same signals. Look for "don't use X", "note:", "important:", "gotcha" language in comments and PR bodies.

Document answers. These fill the gap between what exists and what's known.

---

## Step 5: Write the Memory Files

### Hot memory (`.claude/rules/`) — Navigation, not encyclopedia

**Hard limit: 150 lines per file.** If you're over, cut in this order:
1. Generic facts derivable from reading one obvious file
2. Verbose explanations compressible to a single line
3. Less-critical subsystems (move to `references/` or omit)

Every line must either **prevent a mistake** or **replace a question to a teammate**. If it does neither, cut it.

---

**`architecture.md`** — Answers: "what is this system and where does everything live?"

Required sections:
- Language/framework/version, database, frontend tiers (active / legacy / never touch + routing rule between them)
- Namespace structure, tenancy pattern + enforcement layer
- Job systems (primary vs. legacy + their directories and queue config)
- Auth stack (each library's role)
- Complete soft-delete model list from schema grep
- Internal packages with one-line purposes
- Test directory map
- **Known Gaps / Tech Debt**: dead code directories, commented-out features, unfinished APIs, anything that "looks active but isn't" — this section prevents wasted effort on code that will be deleted

---

**`domain-glossary.md`** — Answers: "what do these words mean?"

Required sections:
- One paragraph per concept where *"what would someone get wrong?"* is non-trivial
- Non-obvious relationships between central entities
- STI models with their actual table name
- Entities that auto-create others in callbacks
- Explicit "A vs B vs C" for easily confused concepts
- **Footguns**: patterns that look correct but silently break something. For each: wrong pattern → right pattern → why. These are the highest-value lines in the file. Examples: using a library-internal column for app logic; accessing JSONB directly instead of via store_accessor; calling a method that bypasses callbacks.

---

**`conventions.md`** — Answers: "how do we build things here?"

Required sections:
- View layer hierarchy + where new views go
- Authorization: show BOTH the correct pattern AND the wrong-looking-similar pattern in code blocks
- Handler/controller structure
- Model conventions: required includes, bulk pattern, and at least one explicit "never do X, use Y instead" with reason
- Service object / command class decision criteria
- Test directory map + auth helpers
- Route organization
- Commit message format from git log

---

### Cold memory (`.claude/rules/references/`) — Only when needed

Create a reference file when a topic is genuinely too complex for a hot-memory pointer. Link to it from the hot-memory file. Don't create cold memory speculatively.

---

**`benchmark.md`** — Write this as the fourth output file, immediately after the other three. You already have everything you need — no new reads required.

```markdown
# Context Benchmark — [Repo Name]

## What these files eliminate

High-cost derivations a cold session would otherwise spend multiple reads on:

| Fact | Source without rules files |
|---|---|
| [non-obvious architectural fact] | [file(s) that reveal it] |
| [non-obvious auth/session design decision] | [file(s)] |
| [non-obvious domain relationship or constraint] | [file(s)] |
| [non-obvious job/queue design rationale] | [file(s)] |
| [footgun or anti-pattern] | [file(s)] |

## Verification questions

Run these in a fresh session. All three should be answered with zero tool calls.

1. **Domain**: [central term] — what is it and how does it relate to [adjacent term]?
   Expected: [answer]

2. **Authorization**: How do I scope a query / add a permission check for [action] on [entity]?
   Expected: [correct pattern]

3. **Placement**: Where does a new [view / worker / test] go?
   Expected: [directory]

## Intentionally excluded

These are NOT covered — they are too volatile or too granular to be worth warm-starting:
- [list of things deliberately omitted and why]
```

The "intentionally excluded" section is as important as the rest: it prevents future maintainers from bloating the rules files with noise that will eventually get ignored.

---

## Step 6: Self-Check Before Writing

Before writing any file, run this check inline (no agents needed):

1. **Line count**: Draft each file mentally — if any section is hitting 40+ lines, it's probably a candidate for `references/` or a cut.
2. **Correctness**: Verify any directory path, class name, or enum value you're about to write exists on disk. `ls` or `grep` it — don't trust memory.
3. **Redundancy**: If the same fact appears in two files, pick the right owner and remove the duplicate.
4. **Signal**: Every line either prevents a mistake or replaces a question to a teammate. If neither, cut it.

---

## Step 7: Cleanup

Remove structural facts now covered by rules files from any `CLAUDE.md` — duplicates become contradictions over time. Note that the rules files exist, when they were last verified, and which codebase they cover.

---

## Decision Table

| Before doing this... | Check whether this already answers it |
|---|---|
| Dispatching an agent to explore | Does schema grep + git log + deps already answer it? |
| Reading source files for patterns | Does linting config already state them? |
| Asking the expert about domain | Do proposals / ADRs / PR descriptions already document it? |
| Writing a glossary entry | Is it already in a proposal or README? |
| Creating a cold-memory reference file | Is a hot-memory pointer sufficient? |
| Running git analysis commands | Did the edge-case checks show < 20 commits or no git? |
| Adding a line to any rules file | Does it prevent a mistake or replace a question to a teammate? |
