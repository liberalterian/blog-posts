# Claude Code Custom Commands: A Technical Primer and Engineering Walkthrough

*From the anatomy of a command file to multi-step workflow design, argument passing, and building a library of reproducible engineering procedures.*

---

## Introduction

There is a class of engineering task that is not quite a skill and not quite a rule. It is a workflow — a named, reproducible, multi-step procedure that you want to invoke deliberately, by name, with specific inputs. Deploy to staging. Run the full pre-PR checklist. Scaffold a new microservice. Generate this week's incident summary.

Skills handle this for some cases, but skills are designed around automatic activation — Claude detects that a task matches the skill's description and invokes it. Sometimes that is what you want. Often it is not. Sometimes you want to type `/deploy staging` and have Claude execute a specific procedure, with no ambiguity about what is happening and no risk of the wrong thing triggering at the wrong time.

That is exactly what custom commands are for.

Claude Code's command system lives at `.claude/commands/`. Each file in that directory defines one command, invocable as a slash command in any Claude Code session. Commands are explicit, parameterizable, version-controlled, and shared with the team through the repository. They are the closest thing Claude Code has to a Makefile target — a named operation with defined inputs and a documented procedure.

This guide covers:

1. **The architecture** — how commands work, how they differ from skills, and the loading model
2. **Anatomy of a command** — file structure, argument handling, and what makes a good command
3. **Command taxonomy** — the four patterns every useful command falls into
4. **Walkthroughs** — building six real commands for a production engineering workflow
5. **Advanced patterns** — argument composition, conditional logic, cross-command workflows, and team distribution

By the end you will be able to design and deploy a command library that makes your most common engineering workflows one slash command away.

---

## Part 1 — How Commands Actually Work

### The Core Mechanic

A command file is a Markdown document that lives in `.claude/commands/<name>.md`. When you type `/<name>` in a Claude Code session, Claude reads that file and executes the instructions inside it as if you had typed them yourself — except more precisely, more consistently, and with any arguments you provided already substituted into place.

That is the entire mechanism. No frontmatter. No YAML. No special syntax beyond standard Markdown and a handful of argument variables. The simplicity is intentional — commands are meant to be approachable to write, easy to read, and straightforward to maintain.

When Claude Code starts, it scans `.claude/commands/` and makes all command files available as slash commands. The command name is derived from the filename: `deploy.md` becomes `/deploy`, `pre-pr-check.md` becomes `/pre-pr-check`. That scan happens at session start — new command files require a session restart to appear.

### Arguments

Commands accept positional arguments, accessed inside the file as `$ARGUMENTS` (the full argument string) or `$1`, `$2`, `$N` (individual positional arguments):

```
/deploy staging v1.2.3
```

Inside `deploy.md`:
- `$ARGUMENTS` → `staging v1.2.3`
- `$1` → `staging`
- `$2` → `v1.2.3`

Arguments are substituted as plain text. There is no type system, no validation at the command invocation level — validation is something you write into the command body itself. ("If `$1` is not one of `staging`, `production`, or `preview`, stop and report an error.")

### Commands vs. Skills: The Right Mental Model

Commands and skills are complementary, not redundant. The distinction is activation mode and intent.

| | **Commands** | **Skills** |
|---|---|---|
| **How invoked** | Explicitly: `/command-name` | Automatically (description match) or explicitly |
| **Best for** | Named, deliberate workflows | Task-class procedures that should auto-activate |
| **Argument passing** | First-class: `$1`, `$2`, `$ARGUMENTS` | Via slash invocation: `/skill-name arg` |
| **Frontmatter** | None required | YAML frontmatter with name, description, allowed-tools |
| **Auto-activation** | Never | Yes, when description matches the task |
| **Confirmation requirement** | Always explicit | Can be automatic |

The practical rule: if you want Claude to do something *automatically* when it detects a certain type of task, it is a skill. If you want something that you invoke *deliberately by name* with specific parameters, it is a command. Many workflows warrant both: a `db-migration` skill that auto-activates during migration work, and a `/migration-status` command for checking the current state on demand.

Commands are also preferred over skills when the procedure is destructive enough that you actively do not want it to trigger automatically. A `/purge-cache` command is safer than a skill that might fire when Claude detects the word "cache" in a conversation.

### Where Commands Live

```
.claude/commands/              ← project-scoped, shared via git (team commands)
~/.claude/commands/            ← personal, available across all projects
```

Project commands are the more important location. They are version-controlled, shared with the team, and tied to the specific project's infrastructure. Personal commands are for utilities that follow you across every project — your personal debugging protocol, your preferred standup format generator, your code review shorthand.

Commands in both locations are available simultaneously in any session. If a personal command and a project command have the same name, the project command takes precedence.

---

## Part 2 — Anatomy of a Command

### File Structure

A command file is a Markdown document with a descriptive name. There is no required structure — just instructions Claude will follow when the command is invoked. That said, a consistent format makes commands easier to write, read, and maintain.

A practical template:

```markdown
# Command Name

Brief one-line description of what this command does.

## Arguments
- `$1` — description of what this argument should be
- `$2` — description of what this argument should be (optional)

## Preconditions
Anything Claude should verify before starting. Fail loudly if preconditions are not met.

## Steps

### 1. First step
What to do. Commands to run if applicable.

### 2. Second step
...

## Completion
What to report when done.
```

None of these sections are mandatory. A simple command might just be a bulleted list of steps. A complex one might have branching logic, error cases, and conditional sections. Write what the procedure demands.

### Good Command Anatomy: What to Include

**A clear first line.** Claude reads the command file from the top. The first thing it reads sets the frame for everything that follows. "Deploy the application to a specified environment after running precondition checks" is a better opening than jumping straight into step 1.

**Explicit preconditions.** What must be true before this command can safely run? Correct branch, clean working tree, tests passing, required arguments present? List them and tell Claude to stop if they are not met. This is the difference between a command that fails safely and one that does something irreversible on the wrong environment.

**Ordered steps with clear exit conditions.** Commands are procedures. Procedures have order. Number your steps. Make clear what constitutes completion of each step. Make clear what should happen if a step fails.

**Argument validation.** If your command takes arguments, validate them at the top: "if `$1` is not one of the expected values, stop and list the valid options." Never assume the user passed valid arguments.

**A completion report.** What should Claude tell you when the command finishes? Specify this. "Report the deployed version, the target environment, and the URL of the deployment" is far more useful than letting Claude decide what is worth mentioning.

### What Not to Put in a Command

**Long reference material.** If a command needs a 200-line API schema to do its work, that schema belongs in a supporting file that the command reads, not inline in the command file itself. Keep command files focused on the procedure; use supporting files for reference material.

**Business logic that belongs in code.** A command that is doing complex branching, data transformation, or decision-making might be better implemented as a script that the command invokes. The command orchestrates; the script executes.

**Anything that should auto-activate.** If you are writing a command for something you want Claude to do automatically whenever it detects a relevant task, that is a skill. Use the right tool.

---

## Part 3 — Command Taxonomy

Commands fall into four practical patterns. Recognizing the pattern before writing saves significant iteration.

### Pattern 1: Environment Operations

Commands that operate on infrastructure, environments, or external services. The defining characteristics: they have environmental targets (staging, production, preview), they are irreversible or have lasting side effects, and they should never run automatically.

**Examples:** deploy, rollback, migrate-production, purge-cache, restart-service, scale-up

**What makes them work:** Strict precondition checks. Explicit environment targeting via `$1`. User confirmation before irreversible steps. Clear completion reporting including the state of the environment after the command runs.

### Pattern 2: Quality Gates

Commands that run a battery of checks before an important workflow step — pre-PR, pre-deploy, pre-release. The defining characteristic: they aggregate multiple individual checks into a single go/no-go verdict.

**Examples:** pre-pr-check, pre-deploy-check, validate-migration, security-scan

**What makes them work:** Exhaustive checklists with explicit pass/fail per item. A clear blocking/non-blocking distinction. An overall verdict at the end. Running these explicitly means nothing slips through because someone forgot to run one of the individual checks.

### Pattern 3: Scaffolding and Generation

Commands that generate boilerplate, scaffold new components, or initialize standard structures. The defining characteristic: they are parameterized by a name or type and produce a predictable set of files.

**Examples:** new-service, new-component, new-migration, generate-client, scaffold-test

**What makes them work:** Argument validation (the name must not already exist, must follow naming conventions). Step-by-step file creation with verification after each file. A completion report that lists every file created.

### Pattern 4: Reporting and Summarization

Commands that aggregate information from across the codebase or external systems and produce a structured report. The defining characteristic: they read without writing, and the output is designed to be shared with someone else.

**Examples:** incident-summary, weekly-report, dependency-audit, coverage-report, changelog-generate

**What makes them work:** Clear output format specification. Explicit scope definition (which time range, which services, which branches). A format that is immediately useful for the intended audience without further editing.

---

## Part 4 — Walkthroughs: Six Production Commands

### Walkthrough 1: Pre-PR Quality Gate

The most universally useful command in any team's library. It runs every check that should be green before a PR is opened, so the reviewer sees clean code and the CI pipeline does not surface anything that should have been caught earlier.

**`.claude/commands/pre-pr-check.md`:**
```markdown
# Pre-PR Quality Check

Run the full pre-PR checklist. All items must pass before opening a pull request.

## Arguments
None. Run from the branch you are about to open a PR for.

## Preconditions
Verify that:
1. We are not on `main` or `develop` — if so, stop with an error
2. The working tree is clean (no uncommitted changes) — if not, remind the user to commit or stash first

## Checks

Run each check in order. Report pass ✅ or fail ❌ for each item.

### 1. Type checking
```bash
npm run typecheck
```
Report: pass or fail with error count.

### 2. Linting
```bash
npm run lint
```
Report: pass or fail with error and warning counts.

### 3. Tests
```bash
npm test
```
Report: pass or fail with test count and coverage percentage.
If coverage drops below 80%, report it as a ❌ FAIL.

### 4. Build
```bash
npm run build
```
Report: pass or fail. If fail, show the first 20 lines of build output.

### 5. Security audit
```bash
npm audit --audit-level=moderate
```
Report: number of vulnerabilities by severity. Any HIGH or CRITICAL is a ❌ FAIL.

### 6. Branch naming convention
Verify the current branch name matches `feature/<description>` or `fix/<description>`.
If not, report a ⚠️ WARNING (not a blocker, but note it).

### 7. Uncommitted changes review
Run `git diff main...HEAD --stat` and report the files changed and lines added/removed.

## Verdict

After all checks:
- If any ❌ FAIL: "NOT READY — X issue(s) must be resolved before opening a PR."
  List the failing checks.
- If any ⚠️ WARNING and no ❌ FAIL: "READY WITH WARNINGS — review the warnings above."
- If all ✅ PASS: "READY — all checks passed. You are clear to open a PR."
```

This command takes under a minute to invoke and eliminates an entire category of "oops, I forgot to run tests" PR comments. Which is to say, it eliminates an entire category of passive-aggressive PR comments. Worth it for that alone.

---

### Walkthrough 2: Deploy to Environment

An environment operation command. Note the guard rails: the target environment must be explicitly specified, preconditions must pass, and the user must confirm before anything is pushed.

**`.claude/commands/deploy.md`:**
```markdown
# Deploy

Deploy the current branch to a specified environment.

## Arguments
- `$1` — target environment: `staging`, `preview`, or `production`

## Argument Validation
If `$1` is not one of `staging`, `preview`, or `production`:
Stop immediately and report:
"Invalid environment '$1'. Valid options are: staging, preview, production."

If `$1` is `production`:
Show a warning: "⚠️  You are deploying to PRODUCTION. This affects live users."
Ask for explicit confirmation before proceeding.

## Preconditions
Check each item. Stop and report if any fail:

1. Working tree is clean: `git status --porcelain` must return empty
2. Current branch is not `main` (for staging/preview) or IS `main` (for production)
3. Tests pass: `npm test -- --passWithNoTests`
4. Build succeeds: `npm run build`

## Deploy Steps

### 1. Tag the deployment
```bash
git tag -a "deploy-$1-$(date +%Y%m%d-%H%M%S)" -m "Deploy to $1"
```

### 2. Push to environment
```bash
# This triggers the CI/CD pipeline for the target environment
git push origin HEAD --tags
```

### 3. Monitor deployment
```bash
# Wait for deployment to complete (check CI status)
gh run watch --exit-status
```
If the deployment fails, report the failure and the URL of the failed run.

### 4. Smoke test (staging and preview only)
After a successful deployment to staging or preview, hit the health endpoint:
```bash
curl -f https://$1.myapp.com/health
```
Report the response status and the deployed version from the response body.

## Completion Report
Report:
- Environment deployed to
- Branch and commit SHA deployed
- Deployment timestamp
- Health check status
- URL of the deployed environment
```

---

### Walkthrough 3: Generate Changelog

A reporting command. This one aggregates commit history into a structured changelog entry, ready to paste into `CHANGELOG.md` with minimal editing.

**`.claude/commands/generate-changelog.md`:**
```markdown
# Generate Changelog

Generate a changelog entry for the commits since the last release tag.

## Arguments
- `$1` — the new version number (e.g., `1.4.2`). Optional.
  If not provided, Claude will determine the appropriate version from commit types.

## Steps

### 1. Find the last release
```bash
git describe --tags --abbrev=0
```
This gives the last release tag. Call it `$LAST_TAG`.

### 2. Get commits since last release
```bash
git log $LAST_TAG..HEAD --pretty=format:"%h %s" --no-merges
```

### 3. Categorize commits
Categorize each commit by its conventional commit type:
- `feat:` → Added
- `fix:` → Fixed
- `perf:` → Changed (performance)
- `refactor:` → Changed (refactoring)
- `docs:` → (omit from changelog unless significant)
- `chore:` → (omit from changelog)
- `BREAKING CHANGE:` in body → Breaking Changes

### 4. Determine version (if not provided as $1)
Based on commit types since last release:
- Any `BREAKING CHANGE` → major bump
- Any `feat:` → minor bump
- Only `fix:` and below → patch bump

Report the determined version and ask for confirmation before continuing.

### 5. Generate the changelog entry

Output in this exact format, ready to paste into CHANGELOG.md:

```
## [$VERSION] - $(date +%Y-%m-%d)

### Breaking Changes
- (list breaking changes, or omit this section if none)

### Added
- (list new features)

### Changed
- (list changes and refactors)

### Fixed
- (list bug fixes)
```

Clean up machine-generated commit messages — expand abbreviations, fix grammar,
combine related commits into single entries. The goal is a changelog entry a
human would write, not a git log dump.

## Completion
Show the generated changelog entry.
Remind the user to:
1. Review and edit before committing
2. Prepend it to CHANGELOG.md
3. Update package.json with the new version number
```

---

### Walkthrough 4: Scaffold New Service

A scaffolding command for a microservices codebase. This demonstrates argument validation, step-by-step file creation, and the completion report pattern.

**`.claude/commands/new-service.md`:**
```markdown
# New Service

Scaffold a new microservice following team conventions.

## Arguments
- `$1` — service name in kebab-case (e.g., `user-notifications`, `payment-processor`)

## Argument Validation
Verify:
1. `$1` is provided — if not, stop: "Usage: /new-service <service-name>"
2. `$1` is kebab-case (lowercase letters, numbers, hyphens only) —
   if not, stop with the correct format
3. A directory named `services/$1` does not already exist —
   if it does, stop: "Service '$1' already exists at services/$1"

## Steps

Create each file in order. Report ✅ after each successful creation.

### 1. Create directory structure
```bash
mkdir -p services/$1/{src,tests,scripts}
mkdir -p services/$1/src/{handlers,services,repositories,schemas,lib}
```

### 2. Create package.json
Create `services/$1/package.json`:
```json
{
  "name": "@myorg/$1",
  "version": "0.1.0",
  "private": true,
  "type": "module",
  "scripts": {
    "start": "node dist/index.js",
    "dev": "tsx watch src/index.ts",
    "build": "tsc",
    "test": "jest",
    "typecheck": "tsc --noEmit"
  }
}
```

### 3. Create TypeScript config
Create `services/$1/tsconfig.json` extending the root tsconfig:
```json
{
  "extends": "../../tsconfig.base.json",
  "compilerOptions": {
    "outDir": "./dist",
    "rootDir": "./src"
  },
  "include": ["src/**/*", "tests/**/*"]
}
```

### 4. Create entry point
Create `services/$1/src/index.ts`:
```typescript
import { createApp } from "./app.js";
import { logger } from "./lib/logger.js";

const port = parseInt(process.env.PORT ?? "3000", 10);

const app = createApp();

app.listen(port, () => {
  logger.info(`$1 service running on port ${port}`);
});
```

### 5. Create health endpoint
Create `services/$1/src/handlers/health.handler.ts` with a standard health check
handler that returns `{ status: "ok", service: "$1", version: process.env.npm_package_version }`.

### 6. Create CLAUDE.md for the service
Create `services/$1/CLAUDE.md`:
```markdown
# Service: $1

## Overview
[TODO: Add a one-paragraph description of what this service does]

## Running Locally
```bash
npm run dev    # development server with hot reload
npm test       # test suite
npm run build  # production build
```

## Architecture
- `src/handlers/` — HTTP route handlers
- `src/services/` — business logic
- `src/repositories/` — database access
- `src/schemas/` — Zod validation schemas

@../../.claude/rules/api-design-standards.md
@../../.claude/rules/testing-requirements.md
```

### 7. Register in workspace
Add `services/$1` to the `workspaces` array in the root `package.json`.

## Completion Report
Report:
- Service name created
- Full directory structure created (tree view)
- Files created (list)
- Next steps:
  1. Add a description to `services/$1/CLAUDE.md`
  2. Run `npm install` from the root to link the workspace
  3. Add the service to the docker-compose.yml for local development
```

---

### Walkthrough 5: Incident Summary

A reporting command. This aggregates information across systems and produces a structured incident summary ready for the post-mortem document or the stakeholder email.

**`.claude/commands/incident-summary.md`:**
```markdown
# Incident Summary

Generate a structured incident summary for a specified time range.

## Arguments
- `$1` — incident start time in ISO 8601 or natural language (e.g., `2026-05-15T14:30:00Z` or `"yesterday at 2pm"`)
- `$2` — incident end time (same format), or `now` if the incident is still ongoing

## Argument Validation
If `$1` is not provided: stop with "Usage: /incident-summary <start-time> <end-time>"

## Steps

### 1. Gather error logs
```bash
# Pull error logs for the time range
grep -r "ERROR\|FATAL" logs/ --include="*.log" | \
  awk -v start="$1" -v end="$2" '$0 >= start && $0 <= end'
```
Summarize: total error count, most frequent error types, affected services.

### 2. Check deployment history
```bash
git log --tags --simplify-by-decoration --pretty="format:%d %ai" | head -20
```
Note any deployments that occurred within 30 minutes before the incident start.

### 3. Check recent configuration changes
```bash
git log --since="$1" --until="$2" --oneline -- "*.env*" "*.config.*" "infra/**"
```
List any configuration changes in the window.

### 4. Calculate impact metrics
From the logs, calculate:
- Duration of incident (from $1 to $2)
- Error rate at peak vs. baseline
- Number of unique users affected (if user IDs appear in logs)
- Services involved

## Output Format

Generate a summary in this format:

---
## Incident Summary

**Duration:** [start] to [end] ([X hours Y minutes])
**Severity:** [P1/P2/P3 — determine from impact scope]
**Status:** [Resolved / Ongoing]

### What Happened
[2-3 sentence plain-language description of the incident based on the logs]

### Impact
- Error rate: [peak rate] vs [baseline rate]
- Users affected: [estimated count or "unknown"]
- Services affected: [list]

### Timeline
- [timestamp] — [event]
- [timestamp] — [event]
(Fill from log events and deployment history)

### Likely Cause
[Best hypothesis from the log evidence and deployment history]

### Immediate Actions Taken
[List any rollbacks, restarts, or mitigations visible in the logs/git history]

### Next Steps
- [ ] Root cause analysis
- [ ] Fix implementation
- [ ] Post-mortem scheduled
---

## Completion
Remind the user to:
1. Fill in the "Immediate Actions Taken" section if not visible in logs
2. Add the post-mortem link when scheduled
3. Share with stakeholders and the on-call channel
```

---

### Walkthrough 6: Dependency Audit

A reporting command that audits the dependency tree for security issues, outdated packages, and license compliance — useful before a major release or on a quarterly cadence.

**`.claude/commands/dependency-audit.md`:**
```markdown
# Dependency Audit

Run a full dependency audit: security vulnerabilities, outdated packages, and license compliance.

## Arguments
- `$1` — audit scope: `security`, `outdated`, `licenses`, or `all`. Default: `all`

## Steps

### 1. Security vulnerabilities
(Run if $1 is `security` or `all`)
```bash
npm audit --json
```
Report:
- Total vulnerabilities by severity (critical, high, moderate, low)
- For CRITICAL and HIGH: package name, vulnerability description, and fix availability
- Recommended action: `npm audit fix` if fixes are available without breaking changes

### 2. Outdated packages
(Run if $1 is `outdated` or `all`)
```bash
npm outdated
```
Categorize results:
- **Major version behind** (e.g., currently on v1.x, latest is v3.x) — flag for manual review
- **Minor/patch behind** — flag as routine updates
- Highlight any package that is more than 2 major versions behind

### 3. License compliance
(Run if $1 is `licenses` or `all`)
```bash
npx license-checker --summary --excludePrivatePackages
```
Flag any package with these licenses as requiring legal review:
- GPL (any version)
- AGPL
- SSPL
- Commons Clause
Report all other licenses as a summary table.

## Output Format

### Security Summary
| Severity | Count | Fixable |
|---|---|---|
| Critical | X | Y |
| High | X | Y |
| Moderate | X | Y |
| Low | X | Y |

### Packages Requiring Attention
(List only critical/high vulnerabilities and major-version-outdated packages)

### License Summary
(Table of license types and package counts)

## Completion Report
Overall verdict:
- 🔴 CRITICAL ACTION REQUIRED — if any critical/high vulnerabilities exist
- 🟡 ATTENTION NEEDED — if any major-version-outdated packages or license concerns exist
- 🟢 ALL CLEAR — if no issues in any category

Remind the user to:
1. File a ticket for any critical/high vulnerability fixes
2. Schedule a dependency update sprint if many packages are major-version outdated
3. Escalate any license concerns to the legal team
```

---

## Part 5 — Advanced Patterns

### Argument Composition

Commands can reference multiple arguments in combination to produce conditional behavior:

```markdown
# Deploy

Deploy to an environment, optionally at a specific version.

## Arguments
- `$1` — environment (`staging`, `production`, `preview`)
- `$2` — version tag to deploy (optional — defaults to current HEAD)

## Steps

If `$2` is provided:
```bash
git checkout $2
```
Report: "Deploying version $2 to $1"

If `$2` is not provided:
Report: "Deploying current HEAD to $1"
Show the commit SHA being deployed: `git rev-parse --short HEAD`

...
```

The conditional is written in natural language. Claude interprets it correctly. You do not need a programming language to branch on argument presence — plain English works.

### Commands That Invoke Scripts

For commands where some steps require deterministic computation — calculating a version number, generating a checksum, parsing a log file — scripts are cleaner than asking Claude to interpret command output:

```markdown
### 1. Calculate new version
```bash
bash .claude/scripts/calculate-version.sh $1
```
The script outputs the new version number. Use it for all subsequent steps.
```

The script does the computation; Claude does the orchestration and reporting. This pattern works especially well for release commands where version math needs to be precise.

### Commands That Reference Rules

Commands can explicitly invoke team rules for the domain they are operating in:

```markdown
# Code Review

Perform a structured code review of the files changed in the current branch.

Apply the review standards from .claude/rules/api-design-standards.md
and .claude/rules/testing-requirements.md when evaluating the changes.

## Steps
...
```

This is particularly useful for quality gate commands — the command procedure defines the workflow, and the rules files define the standards applied within that workflow. Changes to standards automatically flow through to the command without needing to update the command file.

### Cross-Command Workflows

A high-level command can reference lower-level commands by name, creating a composable workflow:

```markdown
# Release

Run the full release workflow: quality gates, version bump, changelog, deploy.

## Arguments
- `$1` — version type: `patch`, `minor`, or `major`

## Steps

### Phase 1: Quality gates
Run /pre-pr-check. If any check fails, stop here. Do not proceed with the release
until all checks pass.

### Phase 2: Changelog
Run /generate-changelog $1. Review the output with the user before proceeding.

### Phase 3: Version bump and tag
```bash
npm version $1
git push origin main --tags
```

### Phase 4: Deploy to production
Run /deploy production. Confirm with the user before executing this step.

## Completion
Report the version released, the tag pushed, and the production deployment status.
```

This pattern works because Claude understands what the referenced commands do and can invoke them in sequence. The `/release` command is readable at a high level; the implementation details live in the individual commands it coordinates.

### Personal Command Libraries

The `~/.claude/commands/` directory is your personal command library — utilities that follow you across every project. Good candidates:

**`~/.claude/commands/standup.md`** — generates a standup summary from recent git commits and open PRs across your active repositories.

**`~/.claude/commands/debug-session.md`** — your personal structured debugging protocol: reproduce the issue, form a hypothesis, isolate variables, verify the fix.

**`~/.claude/commands/explain-diff.md`** — reads a git diff and produces a plain-language explanation suitable for a PR description or team announcement.

**`~/.claude/commands/braindump.md`** — takes unstructured notes and organizes them into a structured document: decisions made, open questions, next steps.

Personal commands are an investment that compounds. A debugging command you write once and refine over six months becomes a genuinely better debugging protocol than you would apply ad hoc.

---

## Part 6 — Quality and Maintenance

### Writing Commands That Actually Work

Commands are instructions Claude will follow precisely. The quality of the command determines the quality of the output. A few principles that separate commands that work reliably from commands that require frequent correction:

**Specify outputs as well as procedures.** A command that tells Claude what to do but not what to report produces unpredictable output. Always specify exactly what the completion report should contain.

**Validate arguments at the top, before anything else.** If an argument is wrong, Claude should fail immediately with a helpful error message — not halfway through the procedure after doing several steps that now need to be undone.

**Make preconditions explicit and hard stops.** "Check that the working tree is clean" is not a precondition. "If `git status --porcelain` returns any output, stop immediately with: 'Cannot deploy with uncommitted changes. Commit or stash your changes first.'" is a precondition.

**Use concrete bash commands for verifiable state.** "Check if tests are passing" is ambiguous. "`npm test` must exit with code 0" is not. When a step can be verified with a command, include the command.

**Keep commands focused.** A command that does ten different things is hard to use, hard to debug when something goes wrong, and hard to maintain when procedures change. If your command is getting long, consider whether it should be multiple commands composed in a higher-level command.

### Command Quality Checklist

Before committing a command:

- [ ] Command name is descriptive and matches the filename
- [ ] Arguments are documented and validated at the top
- [ ] Preconditions are listed and include explicit stop conditions
- [ ] Steps are numbered and ordered correctly
- [ ] Completion report specifies exactly what Claude should output
- [ ] Destructive operations require explicit confirmation
- [ ] The command has been manually tested end-to-end
- [ ] Any scripts referenced by the command exist and are executable
- [ ] The command file is under 150 lines (if longer, consider splitting)

### Versioning and Maintenance

Commands, like rules files, drift if not maintained. A deploy command that references a CI tool you replaced six months ago, a scaffold command that creates the wrong directory structure — these do not just fail to help, they actively mislead.

Treat command files the same way you treat build scripts: they are code, they should be reviewed when changed, and they should be audited when the system they describe changes.

A lightweight maintenance pattern: when a significant infrastructure change lands (new CI/CD system, new scaffolding conventions, new deployment process), add "update relevant command files" to the migration checklist. It takes 20 minutes and prevents six months of subtle workflow failures.

---

## Part 7 — Reference

### Command File Template

```markdown
# Command Name

One-line description of what this command does and when to use it.

## Arguments
- `$1` — description (required/optional)
- `$2` — description (required/optional)
- `$ARGUMENTS` — if the command takes a variable number of arguments

## Argument Validation
Check that all required arguments are present and valid.
Stop with a helpful error message if not.

## Preconditions
List conditions that must be true before starting.
Stop loudly if any are not met.

## Steps

### 1. Step name
Instructions. Commands to run if applicable.
What to do if this step fails.

### 2. Step name
...

## Completion Report
Specify exactly what Claude should report when done.
```

### Command Type Decision Tree

```
What kind of operation is this?
│
├── Affects infrastructure or external services
│   → Environment Operation
│     - Require explicit environment argument ($1)
│     - Strict preconditions
│     - Confirmation before destructive steps
│
├── Aggregates checks into a go/no-go verdict
│   → Quality Gate
│     - Run all checks, report pass/fail per item
│     - Clear overall verdict at the end
│
├── Creates files or initializes structure
│   → Scaffolding
│     - Validate name/args first
│     - Check for conflicts
│     - Report every file created
│
└── Reads and summarizes information
    → Reporting
      - Specify output format precisely
      - Include scope parameters ($1, $2 for time ranges, etc.)
      - State the intended audience
```

### Installation Quick Reference

```bash
# Create a project command (shared via git)
touch .claude/commands/my-command.md

# Create a personal command (all projects)
touch ~/.claude/commands/my-command.md

# Invoke a command
/my-command

# Invoke with arguments
/my-command staging v1.2.3

# List available commands (in session)
# Type / and tab — Claude Code shows available commands

# Commands take effect after session restart if added mid-session
```

### Argument Reference

| Variable | Value |
|---|---|
| `$ARGUMENTS` | Full argument string as provided |
| `$1` | First positional argument |
| `$2` | Second positional argument |
| `$N` | Nth positional argument |

---

## Conclusion

Custom commands are, at their core, a solution to the reproducibility problem in engineering workflows. Every team has procedures that are supposed to happen consistently — before a PR, before a deploy, at release time — and in practice happen inconsistently because they depend on individual engineers remembering the right steps in the right order.

Commands make those procedures reproducible. They live in the repository. They run identically for every engineer. They can be updated centrally when procedures change. And they are one slash command away from any Claude Code session, which is a much lower barrier than opening a Confluence page, finding the right section, and executing eleven steps by hand.

The progression worth following as you build out your command library:

**Start with your pre-PR gate.** This one command, done well, immediately improves code quality and saves review cycles. Write it, test it, refine it over a few weeks.

**Add your most error-prone procedures.** What do engineers on your team get wrong most often? Migrations applied in the wrong order? Deploys to the wrong environment? Changelog entries missed? Those are your next commands.

**Build your scaffolding commands.** Every time a new engineer asks "how do I add a new service/endpoint/component," you have identified a scaffolding command that does not exist yet. Write it, and that question becomes `/new-service`.

**Compose into higher-level workflows.** Once individual commands are reliable, compose them. A `/release` command that invokes `/pre-pr-check`, `/generate-changelog`, and `/deploy production` in sequence is a genuinely powerful workflow abstraction.

The command library you end up with is executable institutional knowledge — the procedures that used to live in runbooks nobody reads, translated into something Claude can actually run. Build it one command at a time, starting with the procedure that causes the most pain today.

---

*Official references: [Claude Code custom commands](https://code.claude.com/docs/en/commands) · [Claude Code memory and project files](https://code.claude.com/docs/en/memory) · [Anthropic prompt engineering overview](https://docs.claude.com/en/docs/build-with-claude/prompt-engineering/overview)*
