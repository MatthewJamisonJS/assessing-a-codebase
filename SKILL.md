---
name: assessing-a-codebase
description: "Use whenever AI sessions keep re-deriving codebase knowledge from scratch — symptoms include sessions starting slow, AI re-explaining domain concepts, requests to create rules files, set up .claude/rules/, make sessions arrive warm, document the repo for AI, set up persistent context, or reduce onboarding overhead. Also use when onboarding to a new codebase and wanting permanent context files so future sessions start informed. Do NOT use for one-time analysis, targeted feature questions, updating existing rules files, or writing a README."
---

# Assessing a Codebase

## Core Principle

Acquire the minimum context needed to understand a codebase from five irreducible perspectives — structural, historical, domain, behavioral, and operational — using the cheapest available source for each gap.

The goal is **tiered persistent memory**:

- `.claude/rules/` — **hot memory**: concise navigation files (architecture, domain, conventions). Always loaded. Scannable in seconds. Point to deeper knowledge; don't duplicate it.
- `.claude/rules/references/` — **cold memory**: on-demand deep specs for complex subsystems. Created only when a topic is too large for a hot-memory pointer.

A session loading these files should answer any architectural, domain, or conventions question with zero tool calls. The benchmark is the proof.

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

## Step 2: Deterministic Analysis (Near-Zero Cost)

These steps are mechanical. Run them all — they provide ground truth that cannot be safely inferred from source code.

**If prerequisites are missing:** shallow git clone (< 50 commits) → skip the tribal-knowledge analysis, focus on schema + deps. `gh` not authenticated → read `git log` commit bodies instead of `gh pr list`. No git at all → skip the two git sub-sections entirely. No expert available → skip Step 4, note unresolved gaps in the files.

### Critical path identification (tribal knowledge locator)

```bash
# Most-changed files = where tribal knowledge concentrates
git log --oneline -100 | wc -l  # confirm depth available
git log --format="%H" | head -100 | xargs -I{} git diff-tree --no-commit-id -r --name-only {} | sort | uniq -c | sort -rn | head -30
```

The top files here are where tacit knowledge lives. Proposals and expert interviews should focus on these areas.

### Commit rhythm and style

```bash
git log --oneline -60          # active areas, rhythm, message format
git shortlog -sn --no-merges   # who built what — who to interview
git log --oneline --after="6 months ago" -- <top-file-from-above>
```

### Schema facts (authoritative — never infer from application code)

```bash
# Find the schema file
find . -name "structure.sql" -o -name "schema.rb" -o -name "*.prisma" -o -name "schema.sql" 2>/dev/null | head -5

# Run on whatever is found:
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

# Changelog — the project's own narrative of what changed and why
cat CHANGELOG.md 2>/dev/null | head -120
```

Read the most recent proposals and 5–10 PR descriptions. They capture design intent, failure modes, and alternatives considered — information that disappears into source code and is never recovered.

### Environment and linting

```bash
cat .env.example .env.sample 2>/dev/null          # external integrations, feature flags
cat .rubocop.yml .eslintrc* pyproject.toml 2>/dev/null | head -80  # machine-readable conventions
ls Procfile docker-compose.yml fly.toml .github/workflows/ 2>/dev/null  # deployment shape
```

---

## Step 3: Parallel Agent Dispatch

Dispatch all three agents **in the same message**. Each follows the Pareto-efficient pattern: semantic search to identify structure → targeted dependency reads → report findings. Do not let agents roam.

**Before dispatching:** substitute `[path]` with `$ARGUMENTS` if the user provided a path, otherwise `.`. Substitute `[top files from git analysis]` and `[N most-changed model files from git analysis]` with the actual top 8 paths from Step 2's git output.

**Agent 1 — Architecture & Structure**

```
In the codebase at [path]:

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

Phase 3 — Return: flat list with directory paths and specific findings. 
No prose. No speculation beyond what the files show.
```

**Agent 2 — Domain Model**

```
In the codebase at [path]:

Phase 1 — Identify the domain layer: find all model/entity files.

Phase 2 — Targeted reads: read the [N most-changed model files from git analysis]
and the central domain model. Find:
- The central entity (what everything else relates to)
- Any entity with a non-obvious table name (STI, shared tables — name the actual table)
- Entities with BOTH a soft-delete column AND an archive column (these serve distinct purposes)
- For soft-delete: does the model use a gem (is_paranoid, acts_as_paranoid) or raw columns directly? The behavior differs — be explicit
- Lifecycle callbacks that auto-create other entities
- Callbacks that write denormalized data onto other models (look for after_create/after_update/after_commit writing to associations — these are caches, not sources of truth)
- Safe query helpers for models with ambiguous FK arrangements (e.g., a junction with person1_id/person2_id — is there a class method that handles the ambiguity?)
- Delegation or forwarding methods that change which entity is "active" for an operation
- Cross-module dependencies: which modules depend on which others in non-obvious ways?
- How does tenant/org scoping propagate through the domain layer?

Phase 3 — Return: entity names, file paths, and specifically what is non-obvious 
about each. Skip obvious things.
```

**Agent 3 — Conventions in Practice**

```
In the codebase at [path]:

Phase 1 — Identify pattern sources: the linting config, one representative 
file from each of: controllers, models, views/templates, tests.

Phase 2 — Read 8-10 files (prioritize the most-changed files from this list: 
[top files from git analysis]). Find:
- What required base classes, includes, or mixins appear in every [model/controller/test]?
- How is authorization checked in practice? Show file:line example of the correct pattern.
- How are bulk operations written? Show file:line example. Is there any post-bulk-insert requirement (e.g., cache-busting calls that ActiveRecord callbacks would have triggered)?
- What cross-module boundaries exist that aren't obvious from the directory structure?
- What test helpers and factories exist? What directories hold what test types?
- What is set up or disabled in test_helper? (job queues faked, auditors disabled, HTTP stubs, etc.)
- What commit message format does git log show?

Phase 3 — Return: findings as file:line — what it shows. No prose. 
Flag anything that looks like an anti-pattern but repeats consistently 
(it's probably canonical here).
```

---

## Step 4: Expert Interview (Tacit Knowledge Only)

This step is the primary source of "things that look right but are wrong" — the distinctions that won't appear in any file read. Agents find what the code shows; the expert fills in what the code hides. Do not skip this lightly.

Read all agent findings before asking. Interview only what the artifacts couldn't answer.

Ask:
- "What 3–5 domain terms regularly confuse people new to this codebase?"
- "What looks like an anti-pattern but is the right call here — and why?"
- "Is anything named one thing but architecturally another?" (shared tables, STI, polymorphic behavior that doesn't match naming)
- "What's the canonical way to [check permissions / scope a query / enqueue a job]?"
- "What would you warn someone not to do — things that look right but silently break something?" (e.g. soft-delete via gem vs. raw columns — the behavior at query time differs)
- For top-changed files from git analysis: "Why does [file] change so frequently?"

Document answers verbatim. These fill the gap between what exists and what's known.

Route into Step 5: domain concepts → domain-glossary.md; "looks like anti-pattern but is right here" and canonical patterns → conventions.md; why-files-change-frequently and architectural intent → architecture.md.

If no expert is available, skip this step. Add `<!-- UNVERIFIED: expert interview skipped -->` at the top of each file so future sessions know the tacit-knowledge layer is missing.

---

## Step 5: Write the Memory Files

### Hot memory (`.claude/rules/`) — Navigation, not encyclopedia

**architecture.md** — Answer "what is this system and where does everything live?":  
Language/framework/version, database, frontend tiers (active / legacy / never touch + routing rule between them), namespace structure, tenancy pattern + enforcement layer, job systems (primary vs. legacy + their directories), auth stack (each library's role), complete soft-delete model list from schema grep, internal packages with one-line purposes, test directory map, active roadmap.

**domain-glossary.md** — Answer "what do these words mean?":  
One paragraph per concept where the answer to *"what would someone get wrong?"* is non-trivial. Always include: non-obvious relationships between central entities, STI models with their actual table name, entities that auto-create others in callbacks, what key delegation methods actually return, explicit "A vs B vs C" for easily confused concepts.

**conventions.md** — Answer "how do we build things here?":  
View layer hierarchy + where new views go, authorization (correct AND wrong pattern in a code block — the difference is often subtle), handler/controller structure, model conventions (required includes, bulk pattern, what not to use), service object / command class decision criteria, test directory map, route organization, commit message format from git log.

**Date stamp:** First line of each hot-memory file: `<!-- Last verified: YYYY-MM-DD -->`

**Rule:** If a developer could derive it by reading one obvious file, omit it. Every line must either prevent a mistake or replace a question to a teammate.

### Cold memory (`.claude/rules/references/`) — Only when needed

Create a reference file when a topic is genuinely too complex for a hot-memory pointer — a deep authorization spec, a finance module architecture, a complex STI hierarchy. Link to it from the relevant hot-memory file. Don't create cold memory speculatively.

---

## Step 6: Quality Audit (Parallel)

Dispatch all three agents **in the same message**.

**Agent A — Correctness**

```
Read architecture.md, domain-glossary.md, and conventions.md from .claude/rules/.
Cross-check:
- Soft-delete column list vs. the schema grep output from Step 2
- Every directory path mentioned — verify it exists
- Entity relationships described — verify against model files

Return: file:line — the claim — why it's wrong or unverifiable. Nothing else.
```

**Agent B — Signal**

```
Read the same three files.
Flag:
- Passages reducible by 30%+ without information loss
- Facts a developer would still need to look up (not actionable as written)
- Anything derivable from one obvious file (README, schema, linting config)

Return: file:line — the entry — why it's low signal. Nothing else.
```

**Agent C — Redundancy**

```
Read the same three files.
Flag:
- The same fact stated in two or more files
- Glossary entries that duplicate conventions.md content

Return: file — what's duplicated — where the canonical home should be. Nothing else.
```

Apply findings. Note false positives and skip.

---

## Step 7: Benchmark

Fresh session, four questions:

1. **Domain:** "What is [central term] and how does it relate to [adjacent term]?"
2. **Authorization:** "How do I add a permission check for [action] on [entity]?"
3. **Placement:** "Where does a new [view / worker / test] go?"
4. **Conventions:** "What commit message format does this project use and how are bulk inserts written?"

**Pass:** All four answered correctly with zero tool calls. Failure = gap in the files. Find it, fill it, retest.

---

## Step 8: Cleanup

Remove structural facts now covered by rules files from any `CLAUDE.md` — duplicates become contradictions.

Write a memory entry: "`[project name]` has `.claude/rules/` files (architecture.md, domain-glossary.md, conventions.md) — verified [date]. Do not re-derive topics they cover."

---

## Step 9: Keep Files Current

Not a re-run — targeted updates only.

After any significant architectural change, run:

```bash
git log --oneline --after="<last-verified-date>"
```

If the diff touches areas the rules files describe: re-run Steps 1–2 for the affected area, update only the changed lines, and update the date stamp. No other steps needed unless the benchmark fails.

**Triggers:** major dependency change, new module, auth refactor, team-reported stale answer.

---

## Decision Table

| Before doing this... | Check whether this already answers it |
|---|---|
| Dispatching an agent to explore | Does schema grep + git log + deps already answer it? |
| Reading source files for patterns | Does linting config already state them? |
| Asking the expert about domain | Do proposals / ADRs / PR descriptions already document it? |
| Writing a glossary entry | Is it already in a proposal or README? |
| Creating a cold-memory reference file | Is a hot-memory pointer sufficient? |
