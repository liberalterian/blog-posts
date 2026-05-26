# Building Claude Code Skills: A Technical Primer and Engineering Walkthrough

*Everything from the architecture of the skills system to writing, structuring, and composing production-grade skills for real software engineering workflows.*

---

## Introduction

If the previous guide on CLAUDE.md was about giving Claude persistent *knowledge* of your project, this one is about giving it persistent *capability*. These are different things, and the distinction matters.

A CLAUDE.md file answers: *what should Claude always know?* A skill answers: *how should Claude execute a specific class of task when asked?*

The canonical example: your CLAUDE.md tells Claude your project uses PostgreSQL with Prisma, follows a specific migration naming convention, and requires tests for all schema changes. A `db-migration` skill tells Claude the exact *procedure* — check for conflicting migrations, run the generator, review the output, verify reversibility, update the seed file, and write a test. CLAUDE.md is ambient knowledge. A skill is an executable protocol.

This guide covers the full progression:

1. **The architecture** — how skills work under the hood, progressive disclosure, and the loading model
2. **Anatomy of a skill** — SKILL.md structure, YAML frontmatter fields, and the directory layout
3. **Skill taxonomy** — the three patterns every engineering skill falls into
4. **Walkthroughs** — building four real software engineering skills from scratch
5. **Composition and advanced patterns** — multi-file skills, bundled scripts, composite skills, and team-scale distribution

By the end you will be able to design, write, and deploy skills that meaningfully change how Claude Code operates inside your engineering workflows. Let's get into it.

---

## Part 1 — How Skills Actually Work

### The Context Cost Problem

Before skills existed, extending Claude Code meant one of two things: stuffing everything into an ever-growing CLAUDE.md, or re-explaining complex procedures in every prompt. Both approaches are expensive. A 5,000-line CLAUDE.md is loaded in full at every session start — you pay the context cost whether or not any of it is relevant to the current task. And re-pasting procedures manually defeats the purpose of having an AI assistant.

Skills solve this with a mechanism called **progressive disclosure**: a tiered loading system where Claude only pays the context cost for knowledge it actually needs.

### Progressive Disclosure: The Three Tiers

When a Claude Code session starts, skills load in stages:

**Tier 1 — Metadata only (always loaded):** The `name` and `description` fields from every installed skill's YAML frontmatter are injected into the system prompt as a compact `<available_skills>` list. A skill at this tier costs roughly 20–50 tokens of context. You can have 50 skills installed and the baseline cost is negligible.

**Tier 2 — SKILL.md body (loaded on activation):** When Claude determines the current task matches a skill's description, it uses a bash read to pull the full SKILL.md content into the context window. This is where your instructions, workflows, examples, and references live.

**Tier 3 — Supporting files and scripts (loaded on demand):** Additional markdown files, reference documents, and scripts bundled in the skill directory are never loaded automatically. Claude reads them explicitly when the SKILL.md instructions direct it to, or runs scripts and receives only their output. Large reference files — full API schemas, database documentation, extensive examples — carry zero context cost until actually accessed.

This architecture is why a well-designed skill library does not slow Claude down. Skills are not loaded into context; they are available as a library the agent pulls from when the task demands it.

### How Activation Works

Claude decides whether to invoke a skill based entirely on the `description` field in the frontmatter. Every session, the available skills list is part of Claude's system context. When you give Claude a task, it scans that list and decides whether any skill description matches.

This has a critical implication: **the description is the skill's trigger.** It is not marketing copy. It is a routing condition. Write it accordingly.

Skills can also be invoked explicitly as slash commands: `/skill-name [arguments]`. This bypasses the description-matching entirely and loads the skill unconditionally — useful for procedural tasks you want to run deliberately rather than have Claude auto-detect.

### Skills vs. CLAUDE.md vs. Slash Commands

This distinction comes up constantly. Here is the clean version:

| | **CLAUDE.md** | **Skills** | **Slash Commands** |
|---|---|---|---|
| **When loaded** | Every session, always | On demand when relevant | When explicitly invoked |
| **Good for** | Universal facts, conventions, constraints | Procedural workflows, domain-specific tasks | Reproducible multi-step procedures |
| **Context cost** | Full file, every session | Metadata only until triggered | Zero until invoked |
| **Version controlled** | Yes (project-level) | Yes | Yes |
| **Supports files/scripts** | No (import syntax only) | Yes | Partially |

The practical rule: if it is a fact Claude should hold in every session, it belongs in CLAUDE.md. If it is a procedure Claude should follow when doing a specific type of task, it is a skill. If it is a workflow you want to trigger explicitly by name, it is a slash command — though as of the current Claude Code version, commands and skills have largely converged, with skills being the recommended path since they support additional features.

---

## Part 2 — Anatomy of a Skill

### Directory Structure

A skill is a directory. At minimum, it contains one file:

```
.claude/skills/
└── my-skill/
    └── SKILL.md
```

A more complete skill might look like:

```
.claude/skills/
└── db-migration/
    ├── SKILL.md              ← required; instructions + frontmatter
    ├── CHECKLIST.md          ← referenced by SKILL.md, loaded on demand
    ├── REFERENCE.md          ← detailed reference, loaded on demand
    └── scripts/
        └── check-conflicts.sh  ← executable script, output only in context
```

Supporting files carry no context cost until Claude reads them. This means you can bundle exhaustive reference material — complete API schemas, large example libraries, detailed style guides — without bloating every invocation.

### SKILL.md Structure

Every SKILL.md has two parts: a YAML frontmatter block, and a Markdown body.

````markdown
---
name: db-migration
description: Create and validate database migrations using Prisma. Use when adding
  or modifying database schema, creating new tables, or altering existing columns.
allowed-tools: Bash, Read, Write
---

# Database Migration Workflow

## Pre-migration checks
Before generating anything, run:
```bash
bash ${CLAUDE_SKILL_DIR}/scripts/check-conflicts.sh
```

## Steps
1. Create the migration: `npx prisma migrate dev --name <descriptive-name>`
...
````

The frontmatter configures *how* the skill runs. The body tells Claude *what* to do.

### YAML Frontmatter Fields

These are the fields you will actually use in engineering contexts:

**`name`** *(string, max 64 chars)* — The human-readable skill name. Also becomes the slash command: a skill named `db-migration` is invocable as `/db-migration`. Technically optional (falls back to folder name), but always set it explicitly — renaming a folder should not silently break your commands.

**`description`** *(string, max 200 chars)* — The activation trigger. Claude uses this to decide when to invoke the skill. Make it specific about *what the skill does* and *when to use it*. First-person, present tense: *"Create and validate database migrations"* not *"You can use this to create migrations."*

**`allowed-tools`** *(list)* — Restricts which tools Claude can use during this skill's execution. Useful for read-only analysis skills where you do not want Claude making file changes, or destructive operations where you want explicit control. Example: `allowed-tools: Read, Grep` for a code review skill.

**`disable-model-invocation`** *(boolean)* — Set to `true` to prevent Claude from auto-triggering this skill. It can only be called explicitly with `/skill-name`. Use for destructive operations (deployments, database drops, mass refactors) that should never activate automatically.

**`context: fork`** *(string)* — Runs the skill in an isolated subagent, keeping its verbose execution out of your main conversation context. Useful for long-running procedural skills where you want a clean main thread.

**`$ARGUMENTS` and `$0`, `$1`, ...`$N`** — Skills accept positional arguments when invoked as slash commands. `/deploy staging v1.2.3` makes `$0 = staging`, `$1 = v1.2.3`, and `$ARGUMENTS = staging v1.2.3`. Use these to make skills parameterizable.

**`${CLAUDE_SKILL_DIR}`** — A special variable that expands to the skill's directory path at runtime. Always use this when referencing bundled scripts or files, so the skill resolves correctly regardless of where it is installed.

### Where to Install Skills

Skills can live in three locations:

**`~/.claude/skills/`** — Personal skills, available across all your projects on this machine. Invisible to teammates. The right place for personal workflow preferences, debugging protocols, and utilities you always want available.

**`.claude/skills/`** (in a project repo) — Project-scoped skills, version-controlled and shared with the team via git. The right place for project-specific workflows: migration procedures, deployment checklists, code generation patterns.

**Organization/plugin level** — Enterprise installations can distribute skills organization-wide through Claude Code's plugin/organization settings. All engineers get the same baseline skills without any per-repo setup.

---

## Part 3 — The Three Skill Patterns

In practice, almost every useful engineering skill falls into one of three patterns. Recognizing which pattern you need before writing saves significant iteration.

### Pattern 1: Encoded Preference

Claude already knows *how* to do the task. The skill encodes *how you want it done* — your team's specific opinions, guardrails, and quality bar.

**Use when:** The cognitive work is about enforcing standards, not providing novel knowledge. Code review checklists, commit message formatting, naming convention enforcement, behavioral guardrails.

**Structure:** Almost entirely Markdown instructions. No scripts needed. Often short — a well-written Encoded Preference skill is 50–150 lines.

**Example use cases:**
- *"Review this PR using our team checklist"* — loads a skill encoding your specific review criteria
- *"Write a commit message"* — loads a skill encoding your team's conventional commits format with project-specific scope types
- *"Refactor this function"* — loads a skill with your team's explicit rules about complexity limits, naming, and documentation requirements

The key insight: Claude does not need to be taught *how to review code*. It needs to be taught *what your team cares about in a review*. That is pure Encoded Preference territory.

### Pattern 2: Procedural Workflow

A multi-step process with a specific sequence, conditional branches, and state that must be tracked across steps. Claude needs to know the protocol, not just the preference.

**Use when:** The task has a defined start, middle, and end. Deployments, database migrations, release workflows, new service scaffolding, incident response runbooks.

**Structure:** Ordered step-by-step instructions, explicit decision branches, bundled scripts for deterministic operations, possibly a config file for environment-specific values.

**Example use cases:**
- *"Deploy to staging"* — sequence: run tests, build, tag, push, update infra, smoke test, notify
- *"Create a new API endpoint"* — sequence: schema validation, handler, service layer, DB query, integration test, documentation entry
- *"Cut a release"* — sequence: version bump, changelog, tag, build artifacts, publish, notify stakeholders

The defining characteristic: the order matters, and skipping steps produces incorrect results. Skills encode the correct sequence so it is reproducible and not dependent on whoever happens to remember the whole procedure.

### Pattern 3: Knowledge Injection

The skill primarily provides domain-specific reference material that Claude cannot reliably hold in working memory — large API schemas, complex domain models, proprietary system documentation, framework internals.

**Use when:** The task requires accurate knowledge of a system, schema, or domain that would be error-prone to rely on Claude's training data for — especially internal systems or heavily version-specific APIs.

**Structure:** Lightweight SKILL.md that directs Claude to reference files. The real content lives in supporting markdown files, loaded on demand. Often paired with scripts that can query live data (current DB schema, current API spec, current feature flags).

**Example use cases:**
- *"Write a query against our analytics schema"* — skill provides the actual schema, table definitions, and join conventions
- *"Implement a feature using our internal auth library"* — skill provides the library's actual API surface and usage patterns
- *"Debug this infrastructure issue"* — skill provides your service topology, common failure modes, and runbook links

The distinction from Procedural: Knowledge Injection is primarily about *what Claude knows*, not *what Claude does*. The workflow is relatively open; the constraint is accuracy of reference material.

---

## Part 4 — Walkthroughs: Four Engineering Skills

Let's build real skills. These are not toy examples — each one is immediately usable in a production engineering context.

### Walkthrough 1: Pull Request Review (Encoded Preference)

**Goal:** Every PR review Claude produces should follow the team's specific checklist, in priority order, with explicit pass/fail verdicts.

**Why a skill, not CLAUDE.md?** Code review is not something Claude should do with every response. It is a specific task that activates when you ask for a review. Loading review instructions universally wastes context. Load them precisely when needed.

**Directory structure:**
```
.claude/skills/
└── pr-review/
    ├── SKILL.md
    └── CHECKLIST.md
```

**`SKILL.md`:**
```markdown
---
name: pr-review
description: Review a pull request or code change against team engineering standards.
  Use when asked to review, audit, check, or give feedback on code changes.
allowed-tools: Read, Grep, Bash
---

# PR Review Protocol

You are performing a structured code review. Follow this process exactly.

## Step 1: Understand the change
Read the diff in full before commenting on anything. If files are not provided,
ask for them. Do not begin the review until you have seen the actual changes.

## Step 2: Apply the checklist
Work through the checklist at CHECKLIST.md. For each category, provide:
- A verdict: ✅ PASS | ⚠️ CONCERN | ❌ FAIL
- Specific line references for any issues found
- Concrete suggestions, not vague advice

## Step 3: Produce the review
Structure your output as:

### Summary
One paragraph: what this change does and your overall verdict.

### Category Results
One section per checklist category, with verdict and findings.

### Required Changes
Numbered list of blockers. Empty if none.

### Suggestions
Non-blocking improvements, each with a rationale.

Do not soften language. "This will cause a race condition in concurrent requests"
is more useful than "This might potentially have concurrency implications."
```

**`CHECKLIST.md`:**
```markdown
# Review Checklist

## 1. Correctness (blockers)
- [ ] Logic handles all stated edge cases
- [ ] Error paths are explicitly handled, not silently swallowed
- [ ] No off-by-one errors in loops or slice operations
- [ ] Concurrent access is safe (locks, atomics, or documented as non-concurrent)

## 2. Security
- [ ] No secrets, credentials, or PII in code or comments
- [ ] Input is validated before use
- [ ] SQL queries use parameterization (no string interpolation)
- [ ] Auth checks are present on all new endpoints

## 3. Test Coverage
- [ ] Happy path is tested
- [ ] At least one error/failure case is tested
- [ ] New public functions have tests
- [ ] Tests are not tautological (they can actually fail)

## 4. Maintainability
- [ ] Variable and function names are self-documenting
- [ ] No function exceeds 40 lines without a comment explaining why
- [ ] Complex logic has inline comments
- [ ] No duplication of existing utilities

## 5. Performance
- [ ] No N+1 query patterns
- [ ] No synchronous I/O on hot paths
- [ ] Memory allocations in loops are flagged for review
```

**How to invoke:**
- Explicitly: `/pr-review` then paste or reference the diff
- Automatically: *"Review this function for me"* — Claude auto-activates from the description match

---

### Walkthrough 2: New API Endpoint (Procedural Workflow)

**Goal:** Every new endpoint follows the same structure — validation, handler, service, repository, test, documentation. No steps skipped, no conventions improvised.

**Directory structure:**
```
.claude/skills/
└── new-endpoint/
    ├── SKILL.md
    ├── templates/
    │   ├── handler.ts.template
    │   ├── service.ts.template
    │   └── test.ts.template
    └── CONVENTIONS.md
```

**`SKILL.md`:**
```markdown
---
name: new-endpoint
description: Scaffold a new REST API endpoint following team conventions. Use when
  creating a new route, handler, or API resource from scratch.
allowed-tools: Read, Write, Bash
---

# New Endpoint Workflow

Arguments: `/new-endpoint <HTTP_METHOD> <route-path> <resource-name>`
Example: `/new-endpoint POST /users/invite user-invite`

## Pre-flight
1. Read CONVENTIONS.md for naming and file placement rules
2. Check the route does not already exist: `grep -r "$0 $1" src/routes/`
3. If a conflict is found, stop and report it. Do not overwrite existing routes.

## Files to Create

Create each file in order. Do not move to the next until the current one compiles.

### 1. Request/Response Schema (`src/schemas/$2.schema.ts`)
- Use Zod for all validation
- Export `${ResourceName}RequestSchema` and `${ResourceName}ResponseSchema`
- All fields have explicit `.describe()` calls for OpenAPI generation

### 2. Handler (`src/handlers/$2.handler.ts`)
- Follows the template at templates/handler.ts.template
- Validates request with schema before any logic
- Returns typed response object, never raw data
- All errors thrown as typed exceptions (see CONVENTIONS.md §3)

### 3. Service (`src/services/$2.service.ts`)
- Business logic only — no HTTP context, no request/response objects
- Follows the template at templates/service.ts.template
- Pure functions preferred; side effects documented in JSDoc

### 4. Repository query (`src/repositories/$2.repository.ts`)
- Prisma query only — no business logic
- All queries typed with Prisma's generated types
- Named exports, no default exports

### 5. Route registration (`src/routes/index.ts`)
- Add route in alphabetical order within its HTTP method group
- Format: `router.$0('$1', validate($2RequestSchema), $2Handler)`

### 6. Integration test (`tests/integration/$2.test.ts`)
- Follows the template at templates/test.ts.template
- Minimum: one success case, one validation failure, one auth failure
- Uses the test database, not production

## Post-creation
Run: `npm run typecheck && npm test -- $2`

If either fails, fix before declaring done. Do not leave broken tests.
```

**`CONVENTIONS.md`:**
```markdown
# API Conventions Reference

## File naming
- kebab-case for all files: `user-invite.handler.ts`
- Folder structure mirrors resource hierarchy

## Error handling (§3)
All handlers use typed exception classes from `src/lib/errors.ts`:
- `ValidationError` — 400, with field-level details
- `AuthorizationError` — 403
- `NotFoundError` — 404
- `ConflictError` — 409
- Never throw raw `Error` objects in handlers

## Response format
All responses use the envelope from `src/lib/response.ts`:
{ success: true, data: T, timestamp: string }
{ success: false, error: { code: string, message: string }, timestamp: string }

## Authentication
All endpoints require authentication unless decorated with @public.
Use the `requireAuth` middleware from `src/middleware/auth.ts`.
```

This skill illustrates the core principle: the procedure is encoded once and executed consistently. No engineer has to remember the six-file sequence. No new team member has to ask. The skill is the documentation.

---

### Walkthrough 3: Database Migration (Procedural + Safety Guards)

**Goal:** Migrations follow a consistent procedure, with explicit safety checks, and a hard stop for destructive operations.

**`SKILL.md`:**
```markdown
---
name: db-migration
description: Create, validate, and apply Prisma database migrations. Use when
  modifying database schema, adding tables, altering columns, or adding indexes.
allowed-tools: Read, Write, Bash
---

# Database Migration Protocol

## Safety rules (non-negotiable)
- NEVER run `migrate deploy` — that is a production-only CI/CD operation.
- NEVER drop columns or tables without explicit user confirmation. Ask first.
- If a migration is irreversible (data loss, column removal), label it clearly
  and ask the user to confirm before generating.

## Step 1: Pre-migration check
```bash
npx prisma migrate status
```
If any migrations are in a "pending" state you did not create, stop and report.
Do not generate new migrations on top of an inconsistent state.

## Step 2: Schema change
Make the requested changes to `prisma/schema.prisma`.

Naming rules:
- Migration names: `<verb>_<noun>` in snake_case
  Examples: `add_user_roles`, `alter_products_add_sku`, `create_audit_log`
- Column names: snake_case
- Table names: plural snake_case (`user_sessions`, not `UserSession`)

## Step 3: Generate migration
```bash
npx prisma migrate dev --name <migration-name> --create-only
```
The `--create-only` flag generates the SQL without applying it. Review before proceeding.

## Step 4: Review the generated SQL
Read the migration file. Verify:
- [ ] The SQL matches the intended schema change
- [ ] No unexpected table drops or column removals
- [ ] Indexes are created CONCURRENTLY if the table has data
- [ ] Backfill logic is present if adding a NOT NULL column to an existing table

## Step 5: Apply locally
```bash
npx prisma migrate dev
npx prisma generate
```

## Step 6: Update seed file
If the migration touches any table used in `prisma/seed.ts`, update it.
Run `npx prisma db seed` to verify it still works.

## Step 7: Write a migration test
Add a test to `tests/migrations/` that:
1. Applies the migration to a blank schema
2. Verifies the resulting structure
3. Applies the rollback (if one exists)

## Done
Report: migration name, files changed, whether seed was updated, whether reversible.
```

The safety rules block at the top acts as an internal guardrail. For truly destructive operations, add `disable-model-invocation: true` to require explicit invocation — never automatic.

---

### Walkthrough 4: Release Workflow (Procedural + Arguments + Scripts)

This one demonstrates argument passing and bundled scripts.

**Directory structure:**
```
.claude/skills/
└── release/
    ├── SKILL.md
    └── scripts/
        ├── check-branch.sh
        ├── bump-version.sh
        └── generate-changelog.sh
```

**`SKILL.md`:**
```markdown
---
name: release
description: Cut a versioned release. Use when creating a new release, bumping
  version numbers, or preparing a release branch.
allowed-tools: Read, Write, Bash
disable-model-invocation: true
---

# Release Workflow

Invocation: `/release <version-type> [<version-number>]`
- `version-type`: `patch`, `minor`, `major`, or `exact`
- `version-number`: required if version-type is `exact` (e.g., `2.1.0`)

Examples:
- `/release patch`         → bumps patch version automatically
- `/release exact 3.0.0`  → sets version to 3.0.0

## Pre-flight checks
```bash
bash ${CLAUDE_SKILL_DIR}/scripts/check-branch.sh
```
Verifies you are on `main`, the branch is clean, and CI is passing.
If any check fails, stop immediately and report the failure. Do not proceed.

## Step 1: Determine version
```bash
bash ${CLAUDE_SKILL_DIR}/scripts/bump-version.sh $0 $1
```
This prints the new version number. Confirm with the user before proceeding.

## Step 2: Generate changelog
```bash
bash ${CLAUDE_SKILL_DIR}/scripts/generate-changelog.sh
```
Generates a draft changelog from commits since the last tag. Review the output,
clean up machine-generated noise, and ask the user to approve before writing.

## Step 3: Write changelog entry
Prepend to `CHANGELOG.md`:
```
## [X.Y.Z] - YYYY-MM-DD
### Added
### Changed
### Fixed
### Breaking Changes (if any)
```

## Step 4: Commit and tag
```bash
git add package.json CHANGELOG.md
git commit -m "chore: release vX.Y.Z"
git tag -a vX.Y.Z -m "Release vX.Y.Z"
```

## Step 5: Push
```bash
git push origin main --tags
```

## Done
Report: version released, changelog summary, tag pushed.
Remind the user that CI/CD handles artifact building and publishing.
```

Key details: `disable-model-invocation: true` because releases should never happen automatically. `${CLAUDE_SKILL_DIR}` resolves script paths correctly regardless of install location. `$0`/`$1` pass the version type and number to the bump script.

---

## Part 5 — Advanced Patterns

### Multi-File Reference Skills (Knowledge Injection at Scale)

For skills that primarily provide reference material, the progressive disclosure architecture becomes genuinely powerful. Consider a skill for an internal analytics platform:

```
.claude/skills/
└── analytics-query/
    ├── SKILL.md               ← ~50 lines; directs Claude to reference files
    ├── SCHEMA.md              ← full table definitions, loaded on demand
    ├── COMMON_PATTERNS.md     ← pre-validated query patterns, loaded on demand
    ├── GLOSSARY.md            ← domain terminology, loaded on demand
    └── scripts/
        └── validate-query.py  ← runs EXPLAIN ANALYZE, returns output only
```

The SKILL.md is intentionally minimal:

```markdown
---
name: analytics-query
description: Write, review, or optimize queries against the internal analytics
  data warehouse. Use when working with analytics data, building reports, or
  investigating data quality issues.
allowed-tools: Read, Bash
---

# Analytics Query Assistant

Before writing any query, read SCHEMA.md for table definitions and relationships.
For common patterns, read COMMON_PATTERNS.md.
For domain terminology, read GLOSSARY.md.

After drafting a query, validate it:
```bash
python3 ${CLAUDE_SKILL_DIR}/scripts/validate-query.py "$query"
```

## Rules
- Always filter by `created_at` range — full table scans are blocked in production
- Exclude test accounts: `WHERE account_type != 'test'`
- Use CTEs for readability on queries over 30 lines
- Add a comment block at the top explaining what each query computes
```

The schema file might be hundreds of lines. None of that enters context until Claude actually needs it. This is the core benefit of the multi-file model — bundle exhaustive reference material with zero upfront cost.

### Composite Skills

Skills can orchestrate other skills, enabling composition of complex workflows from smaller, testable units. A `ship` skill that calls `pr-review`, `db-migration`, and `release` as sub-procedures:

```markdown
---
name: ship
description: Run the full pre-ship checklist: review changes, run migrations, cut release.
disable-model-invocation: true
---

# Ship Checklist

Execute each phase in order. Block if any phase fails.

## Phase 1: Code review
Apply the pr-review skill to all changes since the last release tag.
Do not proceed if any ❌ FAIL verdicts are outstanding.

## Phase 2: Migration check
Apply the db-migration skill to verify all pending migrations are valid
and have been tested locally. Confirm with the user before applying.

## Phase 3: Release
Apply the release skill with the version type the user specifies.
```

Claude understands what the referenced skills do from their descriptions, applies them in sequence, and produces well-organized output. Composite skills are most useful for high-stakes workflows where you want every sub-procedure to be auditable independently.

### Configuration Files for Dynamic Skills

For skills that need environment-specific values — which Slack channel, which deployment target, which Jira board — a `config.json` pattern works well:

```
.claude/skills/
└── standup/
    ├── SKILL.md
    └── config.json       ← created by user on first run; gitignored
```

```markdown
---
name: standup
description: Generate and post the daily engineering standup summary.
---

# Standup Generator

## Configuration
Read ${CLAUDE_SKILL_DIR}/config.json.
If the file does not exist, ask the user for:
- Slack channel name
- Team members to include
- Standup format preference (bullet, narrative, or table)

Then create config.json with their answers before proceeding.
The file is gitignored — it stores your local configuration.
```

The skill self-configures on first use and remembers settings thereafter. No manual setup required; no team credentials checked into version control.

### Letting Claude Write Your Skills

One genuinely useful pattern: complete a task the long way, then ask Claude to capture what it learned as a skill.

From Anthropic's own authoring guidelines: after completing a workflow, identify the context you provided that would be useful for similar future tasks, then ask Claude to create a skill capturing that pattern. Claude understands the SKILL.md format natively and produces properly structured output. You edit and refine from there — substantially faster than writing from scratch.

In practice: *"We just worked through that migration procedure together. Create a skill in `.claude/skills/db-migration/` that encodes this workflow for next time."*

---

## Part 6 — Quality and Maintenance

### Writing Descriptions That Actually Trigger

The description field is where most skill failures originate. A bad description means the skill never activates (too narrow), always activates (too broad), or activates at the wrong time (ambiguous). Follow these rules:

**Be specific about the task class, not the subject matter.** *"Create database migrations using Prisma"* is better than *"Work with databases."* The first is a task class; the second is a domain.

**Include activation phrases.** Think about how you actually ask for this work: *"Use when asked to review, audit, check, or give feedback on code changes"* anchors the skill to natural language patterns.

**First-person, present tense, active voice.** *"Scaffold a new REST API endpoint"* not *"This skill can be used to scaffold REST API endpoints."* Consistent grammatical structure helps Claude's description-matching.

**200 characters is the hard limit — stay under it.** If you cannot describe a skill's activation condition in 200 characters, the scope is probably too broad. Split it.

### SKILL.md Body Best Practices

**Keep the body under 500 lines.** Anthropic's official guidance recommends this ceiling. If your instructions exceed it, split content into reference files and link to them. A 200-line SKILL.md with three supporting files is better than a 600-line monolith.

**Lead with constraints and safety rules.** Put non-negotiable guardrails at the top. Claude reads sequentially; front-loading constraints means they are processed before any action steps.

**Be concrete, not aspirational.** *"Run `npm test` before declaring done"* is enforceable. *"Ensure quality"* is not.

**Leave room for judgment.** Over-specifying every micro-decision produces rigid, brittle skills. Encode the protocol and the constraints; leave Claude room to adapt to the specific situation. Give it the information it needs, not a script for every contingency.

### Skill Auditing Checklist

Before committing a new skill to the team repository:

- [ ] Description is under 200 characters and unambiguous as a trigger
- [ ] Name is kebab-case with no spaces
- [ ] `disable-model-invocation: true` is set for any destructive or irreversible operations
- [ ] `allowed-tools` is constrained appropriately (no Write access for read-only skills)
- [ ] `${CLAUDE_SKILL_DIR}` is used for all script and file path references
- [ ] Safety rules and hard stops are at the top of the body, not buried
- [ ] Supporting files are referenced by name so Claude knows they exist
- [ ] The skill has been manually tested: invoke it, verify Claude follows the protocol

---

## Part 7 — Reference

### Complete YAML Frontmatter Reference

```yaml
---
# Identity
name: skill-name                   # 64 char max; becomes /skill-name command
description: When and what         # 200 char max; the activation trigger

# Access control
allowed-tools: Read, Write, Bash, Grep   # Restrict available tools
disable-model-invocation: false          # true = explicit /command only

# Execution context
context: fork                      # Run in isolated subagent (clean main context)

# Argument documentation
argument-hint: "<type> [version]"  # Shown in autocomplete

# External dependencies
dependencies:
  - python3
  - jq
---
```

### Skill Type Decision Tree

```
Does Claude already know HOW to do this task?
│
├── YES → Does it need to follow YOUR team's specific process or quality bar?
│         │
│         ├── YES → Encoded Preference skill
│         └── NO  → No skill needed; just ask Claude directly
│
└── NO  → Does it require a specific ordered sequence of steps?
          │
          ├── YES → Procedural Workflow skill
          └── NO  → Does it require large or proprietary reference material?
                    │
                    ├── YES → Knowledge Injection skill
                    └── NO  → Add it to CLAUDE.md (it is a fact, not a procedure)
```

### Installation Quick Reference

```bash
# Create a project-scoped skill (shared via git)
mkdir -p .claude/skills/my-skill
touch .claude/skills/my-skill/SKILL.md

# Create a personal skill (all projects, not shared)
mkdir -p ~/.claude/skills/my-skill
touch ~/.claude/skills/my-skill/SKILL.md

# Verify Claude sees your skills (inspect loaded context)
/memory

# Invoke a skill explicitly with arguments
/my-skill arg1 arg2

# Ask Claude to generate a skill from a completed workflow
"Create a skill in .claude/skills/my-skill/ that captures this workflow."
```

---

## Conclusion

Skills are a solution to a specific engineering problem: how do you make procedural knowledge reproducible without paying the context cost of loading it universally?

The progressive disclosure architecture gives a clean answer. Metadata is cheap. Full instructions load only when the task demands them. Supporting files and scripts carry no cost until accessed. The result is a library of capabilities that scales gracefully — 50 skills in a mature project costs almost nothing when you are working on a task that only needs two or three of them.

The progression worth keeping in mind as you build out your skill library:

**Start with Encoded Preference.** Pick the workflow you re-explain most often — code review criteria, commit message format, PR description requirements — and encode it. This alone produces immediate, noticeable consistency improvement.

**Add Procedural skills for your most error-prone workflows.** New endpoint creation, database migrations, deployment procedures, release cuts — anything where skipping a step causes real problems. These pay off the first time they prevent a skipped test or an unreviewed migration.

**Build Knowledge Injection skills for your internal systems.** Your analytics schema, your internal API library, your infrastructure topology — these are exactly the cases where Claude's training data is useless and accurate reference material is everything.

**Compose skills into workflows as your library matures.** A `ship` skill that orchestrates review, migration, and release is only possible once those underlying skills exist and are reliable.

The skill library you end up with is a codification of your team's institutional knowledge. The procedures that live in people's heads, in documentation pages nobody reads, and in Slack threads that scroll off — distilled into executable protocols any engineer on the team can invoke.

That is a meaningful engineering artifact. Build it deliberately.

---

*Official references: [Extend Claude with Skills — Claude Code Docs](https://code.claude.com/docs/en/skills) · [Agent Skills Overview — Anthropic API Docs](https://platform.claude.com/docs/en/agents-and-tools/agent-skills/overview) · [Skill Authoring Best Practices](https://platform.claude.com/docs/en/agents-and-tools/agent-skills/best-practices)*
