# The Engineer's Complete Guide to Instructing Claude: From Chat Projects to Enterprise CLAUDE.md Architectures

*A progression from first principles to production-scale memory systems — covering claude.ai Projects, prompt engineering fundamentals, and the full Claude Code CLAUDE.md ecosystem.*

---

## Introduction

If you have used Claude for more than a few weeks, you have probably hit the same wall everyone hits: you explain your stack, your conventions, your preferred tone, and your specific constraints — then close the browser tab, come back tomorrow, and explain everything again. Claude has the memory of a goldfish with a very impressive vocabulary.

The good news: that wall is entirely engineerable around. The mechanisms range from simple (custom instructions in a chat project) to sophisticated (a hierarchical, version-controlled CLAUDE.md architecture spanning a 200-service enterprise monorepo). This guide walks every level of that spectrum.

We will cover:

1. **Claude.ai Projects** — persistent chat workspaces with custom instructions and knowledge files
2. **Prompt engineering fundamentals** — the craft of writing instructions that actually stick
3. **Claude Code memory systems** — `CLAUDE.md`, auto-memory, and the full file hierarchy
4. **Advanced patterns** — modular imports, `.claude/rules/`, skills, and enterprise-scale architectures

By the end, you should be able to design a context system appropriate for any scale of work, from a personal research assistant to a multi-team engineering platform. Let's get into it.

---

## Part 1 — Chat Projects: Persistent Workspaces Without Code

### What Claude.ai Projects Actually Are

Claude.ai Projects are the no-code equivalent of deploying Claude with a custom system prompt via the API. Each project is a self-contained workspace with its own custom instructions, document library, and conversation history. Every new chat you open inside a project inherits the same configured context automatically — you do not need to paste your prompt at the top of every message.

Think of a well-configured project as a domain-specific colleague who has already read all your documentation, knows your terminology, understands your preferred output format, and never asks you to repeat yourself. That is the goal.

### Anatomy of a Project

A project has three components:

**Custom Instructions** — A free-text field (roughly equivalent to a system prompt) where you define persistent behavioral rules: tone, format, scope restrictions, persona, domain context. This is evaluated before every conversation in the project.

**Knowledge Files** — Documents you upload once and Claude references in any conversation. Good candidates include API documentation, style guides, company policies, technical specifications, glossaries, and examples of work you want Claude to emulate. When you reference something from a knowledge file during conversation, Claude draws on it as if you had pasted the relevant excerpt inline.

**Conversation History** — All chats within the project share the same configured context, though individual conversations remain separate. Related work stays grouped, which eliminates the chronic problem of hunting through a graveyard of unrelated chats.

### Writing Effective Custom Instructions

This is where most people underinvest. A custom instruction is not a chat message. It is a specification. Treat it accordingly.

**The Five Things Every Instruction Set Should Cover:**

1. **Role and domain context** — Who is Claude in this project? What does it know? What field is it operating in?
2. **Audience** — Who will read the output? What is their expertise level?
3. **Output format** — Length, structure, use of markdown, use of code blocks, preferred units, citation style.
4. **Behavioral constraints** — What should Claude always do? What should it never do?
5. **Calibration examples** — Short examples of the tone, style, or format you want are worth more than paragraphs of description.

**A Minimal But Effective Example:**

```
You are a senior backend engineer assistant embedded in a Python/FastAPI team.

Role: Help with architecture decisions, code review, debugging, and documentation.
Audience: Engineers with 3–8 years of experience. Skip basics, but explain non-obvious tradeoffs.

Output format:
- Code blocks for all code, even short snippets
- Use type annotations in all Python examples
- Prefer concrete recommendations over option menus ("use X" not "you could use X or Y")
- Responses under 400 words unless the complexity genuinely requires more

Constraints:
- Never suggest adding dependencies without noting the tradeoff
- Flag security issues explicitly with a ⚠️ marker
- Do not use async/await unless it's relevant to the question
```

Clean, specific, opinionated. Claude responds significantly better to this than to a paragraph of vague aspirational language about being "helpful and professional."

### Knowledge File Strategy

Not everything belongs in custom instructions. Knowledge files are better for:

- **Reference material** that is too long for instructions but that Claude needs to quote or reason about (API schemas, database models, long style guides)
- **Domain glossaries** — specialized terminology that would otherwise require constant correction
- **Persona documents** — a brief document about your background, role, and expertise level so Claude calibrates responses to you specifically
- **Templates and examples** — actual samples of output you want emulated, not just descriptions of it

A practical rule: if you would put it in a README for a new team member, put it in a knowledge file. If you would put it in a team meeting agenda slide, it belongs in instructions.

---

## Part 2 — Prompt Engineering Fundamentals

Before we graduate to file-based memory systems, it is worth grounding the fundamentals. A well-designed CLAUDE.md is only as good as the instructions written inside it.

### Specificity Over Aspiration

"Be concise" is aspirational. "Responses under 300 words unless the problem requires a code example, in which case the prose around the code should still be under 200 words" is specific. Claude follows the second one. It guesses at the first.

The principle applies everywhere: replace adjectives with measurable criteria wherever possible.

| Vague | Specific |
|---|---|
| "Use clear variable names" | "Variable names must be self-documenting. No single-letter names outside of loop indices." |
| "Be professional" | "Write in a neutral technical register. No exclamation points. No em-dash enthusiasm." |
| "Handle errors properly" | "All functions that can fail must return a Result type or raise a typed exception. No bare `except:` clauses." |

### Positive Instructions Outperform Negative Ones

"Do not use bullet points" is worse than "Write in prose paragraphs." The former tells Claude what to avoid; the latter gives it a target. Both matter, but lead with the positive.

This is especially true for style: rather than cataloguing what you dislike, provide a short example of what good looks like. Claude pattern-matches from examples far more reliably than it extrapolates from prohibitions.

### Explicit Structure for Complex Instructions

For anything non-trivial, use explicit structure — headers, numbered rules, clear section labels. Claude processes structured instructions more reliably than prose-embedded ones. It is easier to catch contradictions when writing them, too. (And yes, you should audit your instructions for contradictions. Claude may pick one arbitrarily when two rules conflict. Don't give it the chance.)

### The Role of Examples (Few-Shot Prompting)

Few-shot examples are among the highest-leverage tools in prompt engineering. Showing Claude two or three examples of the exact format, tone, or structure you want is dramatically more effective than describing it in words.

In a Projects context, you can put these examples in knowledge files and reference them in your instructions: *"Format all status reports like the examples in status-report-template.md."*

In Claude Code CLAUDE.md files, you can do the same: *"See `docs/code-review-checklist.md` for the review format we use on this team."*

### Handling Ambiguity Explicitly

Tell Claude what to do when it is uncertain. "If the requirement is ambiguous, ask one clarifying question before proceeding" is far better than leaving Claude to either guess or produce a long hedged response covering every possibility. Ambiguity handling is often the difference between a useful AI assistant and an irritating one. Specify it.

---

## Part 3 — Claude Code Memory Systems

Now we enter engineering territory. Claude Code — the terminal-based agentic coding tool — has a layered memory architecture that is materially more sophisticated than a single instruction field. Understanding it is prerequisite to using Claude Code effectively at any serious scale.

### The Core Problem: Sessions Are Stateless

Every Claude Code session begins with a fresh context window. Without memory files, Claude has no knowledge of your stack, your conventions, your preferences, or the mistakes it made last week. Every session is day one.

Memory files are the solution. They are loaded into the context window at session start, giving Claude persistent knowledge across sessions without requiring you to re-explain anything.

### The Three-Layer Memory Stack

Claude Code uses three complementary memory systems, all loaded at session start and stacked in priority order:

**Layer 1: Global User Memory — `~/.claude/CLAUDE.md`**
Your personal preferences that apply across all projects on your machine. This is the place for universal editor preferences, preferred code style defaults, personal workflow quirks, and anything that is about you rather than about a specific project. Changes here affect every Claude Code session.

**Layer 2: Project CLAUDE.md — `./CLAUDE.md` (and nested variants)**
The primary engineering artifact. Team-shared, version-controlled instructions specific to a project. Architecture, build commands, conventions, testing requirements, directory structure, team norms. This is what most of this article focuses on.

**Layer 3: Auto-Memory — `~/.claude/projects/<project>/memory/`**
Notes Claude writes to itself based on what it learns during sessions — file naming patterns it observed, dependencies it discovered, conventions it inferred from your code. This layer is personal (not shared with the team) and self-maintaining. You can inspect it with the `/memory` command.

The priority rule: more specific instructions override broader ones. A project-level CLAUDE.md overrides global user memory. A subdirectory CLAUDE.md overrides the root project file.

There is also `CLAUDE.local.md` — automatically added to `.gitignore`, intended for private local settings like sandbox URLs, local environment notes, or personal testing preferences that the team does not need.

### The CLAUDE.md File: First Principles

A CLAUDE.md file is a Markdown document that Claude reads at the start of every session. Treat it as the onboarding document you would write for a highly capable new engineer who has never seen your codebase — because that is exactly what it is.

The official guidance from Anthropic's documentation describes it as the ideal place for documenting: common bash commands, core files and utility functions, code style guidelines, testing instructions, repository etiquette (branch naming, merge vs rebase preferences), developer environment setup, unexpected behaviors specific to the project, and anything else you want Claude to hold in every session.

A practical minimal structure:

```markdown
# Project: <name>

## Overview
One paragraph: what this project is, its primary language/framework, and deployment target.

## Environment Setup
```bash
# Install dependencies
npm install

# Run tests
npm test

# Start dev server
npm run dev
```

## Architecture
- `src/api/` — Express route handlers
- `src/services/` — Business logic layer
- `src/models/` — Prisma ORM models
- `tests/` — Jest test suite

## Conventions
- TypeScript everywhere. No `any` types.
- All functions require JSDoc comments.
- Errors are thrown as typed exceptions, never returned.

## Branch & Commit Rules
- Branches: `feature/<description>` or `fix/<description>`
- Commits: Conventional Commits format
- No direct commits to `main`

## What Not to Do
- Do not modify `src/lib/legacy/` — it will be deleted in Q3, don't add to it.
- Do not use `console.log` for debugging; use the logger at `src/utils/logger.ts`.
```

Concise. Factual. Actionable. This is the pattern.

### The Loading Hierarchy: What Claude Reads and When

This is the most commonly misunderstood part of Claude Code's memory system, and getting it right is essential for advanced use.

**Always loaded at session start:**
- Root `CLAUDE.md`
- All parent directory `CLAUDE.md` files (ancestor chain up to filesystem root)
- `~/.claude/CLAUDE.md` (global user memory)
- Auto-memory for the current project

**Loaded on access (lazy loading):**
- Nested `CLAUDE.md` files in subdirectories — loaded when Claude navigates to or reads files from that directory

**Loaded on demand:**
- Skills (`.claude/skills/`) — the name and description load initially; the full skill file loads only when Claude determines it is relevant

**Never auto-loaded:**
- Files in `.claudeignore`
- Files denied in `settings.json`
- `node_modules/`, `.git/`, build output directories

The lazy-loading behavior of nested `CLAUDE.md` files is significant: Claude does not bloat its context with database migration instructions while working on the frontend. Instructions are scoped to where they are relevant, and loaded precisely when they become relevant.

### Import Syntax: Modular Instruction Files

CLAUDE.md files support an import syntax that enables modular composition:

```markdown
@docs/architecture.md
@docs/api-standards.md
@.claude/rules/security-policy.md
```

Imported files are expanded inline at session start. Both relative and absolute paths are supported. Relative paths resolve from the file containing the import, not the working directory. Imports can be recursive, up to five hops deep.

This enables a critical pattern for teams: a root CLAUDE.md that imports specialized documents, rather than a single monolithic file that becomes a maintenance liability.

```
project/
├── CLAUDE.md                    ← imports the below
├── docs/
│   ├── architecture.md
│   ├── api-standards.md
│   └── database-schema.md
└── .claude/
    └── rules/
        ├── security-policy.md
        └── testing-requirements.md
```

### `.claude/rules/` — Topic-Scoped Rule Files

For large projects, a flat CLAUDE.md becomes unwieldy. The `.claude/rules/` directory allows you to decompose instructions by topic. Each file in this directory covers one domain: security, testing, API design, database conventions, CI/CD requirements.

The root CLAUDE.md imports what is globally relevant; path-scoped rules load when Claude enters relevant subdirectories. The result is a system where:

- Global rules stay global
- Package-specific rules stay local
- Context size stays manageable
- Rules are auditable by domain

### Auto-Memory: The Self-Writing Layer

Auto-memory deserves more attention than it typically receives. Claude maintains its own notes at `~/.claude/projects/<project>/memory/` — things it learned about your project that are not in CLAUDE.md: file naming patterns it observed, dependencies it discovered, conventions it inferred from your code corrections.

Run `/memory` to inspect exactly what Claude has written to itself. This has an important practical implication: do not waste CLAUDE.md space on things Claude will discover on its own within the first session. If you correct Claude once and it records the pattern to auto-memory, you do not need that line in your CLAUDE.md.

Auto-memory is personal — not shared with the team. CLAUDE.md is shared. Keep the distinction clear: team knowledge goes in version-controlled CLAUDE.md; personal session-learned knowledge accumulates in auto-memory.

**Debugging tip:** If Claude ignores a rule, run `/memory` first. The rule may be in a nested file that has not been loaded yet because Claude has not read files from that directory.

---

## Part 4 — Advanced Patterns and Enterprise Architectures

### Monorepo CLAUDE.md Architecture

Monorepos introduce a specific challenge: a single root CLAUDE.md gets loaded for every task, which means your frontend developer receives irrelevant database migration advice and your backend service receives design token instructions. Context gets noisy, token budgets inflate, and Claude makes decisions based on rules that do not apply.

The solution is a hierarchical CLAUDE.md architecture:

```
my-monorepo/
├── CLAUDE.md                    ← universal rules only
├── packages/
│   ├── api/
│   │   └── CLAUDE.md            ← Node.js + PostgreSQL conventions
│   ├── web/
│   │   └── CLAUDE.md            ← Next.js + Tailwind conventions
│   ├── mobile/
│   │   └── CLAUDE.md            ← React Native + Expo conventions
│   └── shared/
│       └── CLAUDE.md            ← ⚠️ HIGH IMPACT: changes affect all packages
└── infra/
    └── CLAUDE.md                ← Terraform + AWS conventions
```

**Root CLAUDE.md — keep it lean.** A healthy root file is 200–300 lines covering only what applies across every package: TypeScript-everywhere rules, conventional commits, no-secrets policy, universal testing requirements. Beyond 500 lines, start questioning whether content belongs in a subdirectory. Recommended guidance is that root CLAUDE.md loading should cost no more than roughly 10–15% of the session token budget.

**Package-level CLAUDE.md — specific and scoped.** Each package file covers only its domain. The `packages/shared` CLAUDE.md is particularly important to annotate explicitly — changes to shared packages propagate everywhere, and Claude should know to treat them with appropriate caution.

When working in a monorepo, explicitly tell Claude which package you are in. *"I am working in `packages/api`. Only modify files under `packages/api/`. Do not touch `packages/shared/` types directly."* Without this, Claude may modify the wrong package or duplicate shared types.

### The Enterprise-Scale Architecture

For large engineering organizations — hundreds of engineers, dozens of services, regulated industries with compliance requirements — a five-layer architecture is appropriate:

**Layer 1: Organization Policy** (`~/.claude/org-policy.md` or loaded via Claude Code organization settings)
Security policies, compliance requirements, data handling rules. Applies to all engineers, all projects. Managed by the platform team.

**Layer 2: Global Engineer Preferences** (`~/.claude/CLAUDE.md`)
Individual engineer preferences: editor settings, personal style defaults, preferred debugging approach. Local to each machine.

**Layer 3: Repository Root** (`./CLAUDE.md`)
Project-wide conventions, architecture overview, universal build commands, team norms. Version-controlled, reviewed on changes like any critical documentation.

**Layer 4: Package / Service Level** (`packages/<name>/CLAUDE.md`)
Service-specific rules, local API conventions, framework-specific patterns. Maintained by the team that owns the service.

**Layer 5: Directory Rules** (`.claude/rules/<topic>.md` or nested `CLAUDE.md`)
Highly specific rules for particular subdirectories. Database migration policies, security-sensitive code handling, generated-file constraints.

At enterprise scale, the CLAUDE.md hierarchy is engineering infrastructure. It should be reviewed, version-controlled, tested (manually verify that Claude follows critical rules), and owned — with a team responsible for its maintenance. Treat it like a `.eslintrc` or a `Dockerfile`, not a personal notes file.

### Skills: Reusable Procedural Knowledge

Beyond memory, Claude Code supports a skills system (`.claude/skills/`) for reusable procedural knowledge — multi-step procedures, domain-specific workflows, complex refactoring patterns. Unlike CLAUDE.md instructions (which are always-on facts), skills load on demand when Claude determines they are relevant.

This distinction matters: CLAUDE.md is for *what Claude should always know*; skills are for *how Claude should do specific complex tasks when asked*.

A skill file might cover: the exact process for creating a new API endpoint following your team's standards (schema validation, handler, service layer, database migration, integration test, documentation), or the steps for a compliant database migration in a regulated environment.

Skill files sit at `.claude/skills/<name>/SKILL.md`. The name and a brief description load at session start (so Claude knows what is available); the full file loads only when the skill is invoked.

### Custom Slash Commands

Claude Code also supports custom slash commands at `.claude/commands/<name>.md`. These are user-defined commands that execute multi-step workflows. Common uses: `/review` (run the full code review checklist), `/ship` (run tests, lint, check coverage, generate changelog entry), `/debug` (structured debugging protocol for common failure modes).

Commands are invoked explicitly (`/review src/api/user.ts`) rather than loaded into context continuously. Use them for workflows you want reproducible but that would be too verbose to include as standing instructions.

### Resolving Conflicts and Auditing

At scale, instruction conflicts become the primary maintenance problem. Two rules that contradict each other produce unpredictable behavior — Claude picks one, and you may not discover this until it matters.

Mitigation strategies:

- **Periodic audits** — review all CLAUDE.md files and `.claude/rules/` quarterly, or when major architectural changes occur
- **Explicit precedence** — state priority rules explicitly: *"These API standards override any conflicting conventions in the root CLAUDE.md"*
- **`claudeMdExcludes`** — in monorepos, use this setting to exclude CLAUDE.md files from other teams that are not relevant to your work
- **Testing critical rules** — manually verify that Claude follows security and compliance rules by testing them explicitly in a session before relying on them

---

## Part 5 — Reference Cheatsheet

### When to Put Something Where

| Content Type | Location |
|---|---|
| Universal personal preferences | `~/.claude/CLAUDE.md` |
| Project architecture + build commands | Root `./CLAUDE.md` |
| Team-shared conventions | Root `./CLAUDE.md` (version-controlled) |
| Private local settings (sandbox URLs, etc.) | `./CLAUDE.local.md` (gitignored) |
| Package-specific rules in a monorepo | `packages/<name>/CLAUDE.md` |
| Domain-specific rule sets (security, testing) | `.claude/rules/<topic>.md` |
| Reusable complex procedures | `.claude/skills/<name>/SKILL.md` |
| Reproducible multi-step workflows | `.claude/commands/<name>.md` |
| Personal per-project session learnings | Auto-memory (Claude writes automatically) |
| Long reference documents | Imported via `@path/to/file.md` |

### CLAUDE.md Quality Checklist

Before committing a CLAUDE.md:

- [ ] Every rule is specific, not aspirational
- [ ] No contradictions between rules in this file and parent files
- [ ] Build commands are tested and current
- [ ] Architecture section reflects actual current directory structure
- [ ] "Do not" rules have a "do this instead" counterpart
- [ ] No content that Claude will learn on its own within one session (check `/memory`)
- [ ] File length is appropriate to scope (root: ≤300 lines; package-level: ≤150 lines)

---

## Conclusion

The gap between engineers who use Claude effectively and those who use it like a slightly faster Stack Overflow grows wider every month. Memory architecture is a large part of that gap — not because it is technically difficult, but because most people never realize it exists.

The progression is worth summarizing cleanly:

**For personal workflows** — configure a claude.ai Project with specific custom instructions and targeted knowledge files. This alone eliminates most repetition and gets you consistent, calibrated output.

**For individual engineering work** — a root `CLAUDE.md` with a clear architecture section, build commands, and explicit conventions. Add a personal `~/.claude/CLAUDE.md` for preferences. Run `/memory` periodically to see what Claude has learned and avoid duplicating it.

**For team engineering** — version-controlled `CLAUDE.md` hierarchy with package-level scoping, `.claude/rules/` for domain separation, import syntax for modularity. Treat it as infrastructure, not documentation.

**For enterprise engineering** — five-layer architecture with organizational policy, hierarchical project files, skills for complex workflows, custom commands for reproducible procedures, and a team that owns and audits the whole system.

None of this requires exotic tooling. It requires clear thinking about what Claude needs to know, when it needs to know it, and at what scope.

Build the memory system your work deserves. Future you — the one who returns to a project after two weeks away and finds that Claude immediately knows exactly what it is, how it works, and what the team expects — will be grateful.

---

*For further reading: [Claude Code memory documentation](https://code.claude.com/docs/en/memory) — [Anthropic prompt engineering overview](https://docs.claude.com/en/docs/build-with-claude/prompt-engineering/overview)*
