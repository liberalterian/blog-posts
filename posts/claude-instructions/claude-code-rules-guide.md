# Claude Code Rules: A Technical Primer and Engineering Walkthrough

*From the anatomy of a rule file to topic-scoped architectures, import composition, and production-grade rule systems for real software engineering teams.*

---

## Introduction

If CLAUDE.md is the onboarding document you write for a highly capable new engineer, then `.claude/rules/` is your team's policy handbook — the place where the non-negotiable standards live, organized by domain, versioned like code, and enforced consistently across every session.

Rules are not a new concept. Every mature engineering organization has them: security policies, API design standards, testing requirements, database conventions. The problem has always been distribution and enforcement. They live in a Confluence page nobody reads, a Notion doc that is three major refactors out of date, or worse — exclusively in the heads of the engineers who wrote the original code.

Claude Code's rules system changes that calculus. Rules files are Markdown documents that Claude reads and follows as standing instructions. They are version-controlled alongside the code they govern. They compose modularly through an import system. And they scope precisely — a database rule loads when Claude is working on database code, not when it is editing the frontend.

This guide covers the full progression:

1. **The architecture** — how rules relate to CLAUDE.md, how they load, and when to use them
2. **Anatomy of a rule file** — structure, naming conventions, and what belongs where
3. **Rule taxonomy** — the four categories every engineering rule set falls into
4. **Walkthroughs** — building five real rule files for a production engineering team
5. **Advanced patterns** — import composition, path-scoped rules, conflict resolution, and enterprise architectures

By the end you will be able to design and deploy a rules system that meaningfully enforces your team's standards — not as documentation, but as active, session-persistent instructions Claude actually follows.

---

## Part 1 — How Rules Actually Work

### Rules vs. CLAUDE.md: The Right Mental Model

Before getting into mechanics, the conceptual distinction matters. CLAUDE.md and rules files occupy different roles in the same system:

**CLAUDE.md** is the project overview — architecture, build commands, directory structure, high-level conventions. It is what Claude needs to orient itself and work effectively in any part of the codebase. It is loaded in full at session start, every time.

**Rules files** are policy documents. They govern specific domains — security, testing, API design, database conventions — in more depth than a root CLAUDE.md can accommodate without becoming unwieldy. They are either imported into CLAUDE.md (always-loaded) or placed in subdirectory CLAUDE.md files (lazy-loaded when Claude navigates that part of the codebase).

Think of it this way: CLAUDE.md is the employee handbook introduction. Rules files are the departmental policy appendices. You reference them from the intro; you do not cram them all into the first page.

### The Loading Model

Rules files are plain Markdown. They have no special frontmatter, no activation triggers, no YAML configuration. What makes them rules files is where they live and how they are referenced.

There are two loading patterns:

**Import-loaded (always active):** The root CLAUDE.md imports a rules file with the `@` import syntax:

```markdown
@.claude/rules/security-policy.md
@.claude/rules/testing-requirements.md
```

These files are expanded inline at session start. Claude holds them as persistent instructions for the entire session, regardless of which part of the codebase it is working in.

**Subdirectory-loaded (lazy, scoped):** A rules file placed inside a subdirectory's `CLAUDE.md` — or referenced from one — loads only when Claude navigates into that directory. A database migration rule in `src/db/CLAUDE.md` does not consume context when Claude is editing `src/components/`.

**The practical implication:** universal policies (security, commit conventions, secret handling) belong in import-loaded rules. Domain-specific policies (database conventions, API design patterns, infrastructure rules) belong in subdirectory-scoped rules. The goal is always to load exactly the rules that are relevant to the current task, and no more.

### Why Rules Files Instead of Longer CLAUDE.md?

Three reasons, all practical:

**Maintainability.** A 600-line CLAUDE.md covering architecture, build commands, and every team policy is impossible to maintain. When your security policy changes, you should be able to update one file, not hunt through a monolith. Rules files make individual policies independently editable and reviewable.

**Ownership.** In larger organizations, different teams own different policies. The platform security team owns the security policy. The data team owns the database conventions. Separate files means separate ownership, separate review processes, and separate change history in git.

**Composability.** Rules files compose. A root CLAUDE.md can import shared organizational policies from a common location while also referencing project-specific rules. Subdirectory CLAUDE.md files can import rules from parent directories. The import system is a genuine composition mechanism, not just a copy-paste shortcut.

---

## Part 2 — Anatomy of a Rule File

### Structure

A rules file is a Markdown document. That is the complete technical specification — there is no frontmatter, no required schema, no special syntax beyond what Markdown already provides.

What that means in practice: you write rules the same way you would write any internal documentation, with the important difference that Claude treats this document as standing instructions rather than reference material.

A well-structured rules file has:

```markdown
# [Domain] Policy

## Purpose (optional but useful)
One sentence: what this policy governs and why it exists.

## Rules

### [Category 1]
- Specific, enforceable rule
- Another specific rule
- Rule with a "do this instead" counterpart

### [Category 2]
...

## Rationale (optional)
Brief explanation of non-obvious rules. Helps Claude apply them
correctly in edge cases where the letter of the rule is ambiguous.

## Examples (optional but high-leverage)
Concrete examples of compliant and non-compliant patterns.
```

The optional sections are worth including when rules might be ambiguous in application. A rule that says "no raw SQL" is clear. A rule that says "prefer composition over inheritance" benefits from examples showing what that means in your specific codebase.

### Naming Conventions

Rules files live in `.claude/rules/`. Use kebab-case filenames that describe the domain the file governs:

```
.claude/rules/
├── security-policy.md
├── testing-requirements.md
├── api-design-standards.md
├── database-conventions.md
├── error-handling.md
├── code-style.md
└── ci-cd-policy.md
```

Each file covers exactly one domain. If you find yourself writing a rules file that covers both API design and database conventions, split it. The single-responsibility principle applies to policy files for the same reason it applies to source files: mixed concerns are harder to maintain, harder to review, and harder to reason about when something goes wrong.

### What Belongs in a Rules File (vs. CLAUDE.md vs. Skills)

| Content | Location |
|---|---|
| Project overview, architecture, build commands | Root `CLAUDE.md` |
| Universal team policies (security, secrets, commits) | `.claude/rules/`, imported into root CLAUDE.md |
| Domain-specific policies (database, API, infra) | `.claude/rules/`, referenced from subdirectory CLAUDE.md |
| Complex multi-step procedures | `.claude/skills/` |
| Facts Claude will learn within one session | Auto-memory (don't write them down) |

The distinction between rules and skills is the most common source of confusion. Rules are *standing instructions* — things Claude should always do or never do. Skills are *executable procedures* — step-by-step workflows for a specific class of task. "Never use raw SQL strings" is a rule. "Here is the ten-step procedure for creating a safe parameterized migration" is a skill.

---

## Part 3 — Rule Taxonomy

Every engineering rule falls into one of four categories. Knowing which category you are writing helps you write it correctly.

### Category 1: Prohibitions

Things Claude must never do. The clearest category to write and the most important to get right.

**Characteristics:** Short, declarative, often accompanied by a "do this instead" redirect. Prohibitions are load-bearing rules — violating them produces real consequences (security vulnerabilities, data loss, broken builds).

**Examples:**
- Never commit secrets, credentials, or API keys to the repository
- Never use `console.log` — use the project logger at `src/utils/logger.ts`
- Never modify files in `src/generated/` — they are auto-generated and will be overwritten
- Never write raw SQL strings — use the query builder at `src/db/query.ts`

**Writing guidance:** Always pair a prohibition with an alternative. "Never do X" without "do Y instead" forces Claude to guess, and it may guess wrong. "Never write raw SQL — use the parameterized query builder at `src/db/query.ts`" gives Claude a clear path forward.

### Category 2: Requirements

Things Claude must always do. The affirmative complement to prohibitions.

**Characteristics:** Often more nuanced than prohibitions because they frequently have conditional structure: "always do X when Y." Requirements encode the team's quality bar.

**Examples:**
- All new functions must have JSDoc comments
- All endpoints must validate input with Zod before processing
- All database queries must include a timeout parameter
- All error responses must use the standard error envelope from `src/lib/errors.ts`

**Writing guidance:** Make requirements measurable. "Write good tests" is aspirational. "Every new public function requires at minimum one happy-path test and one error-path test" is enforceable.

### Category 3: Preferences

Guidance for situations where multiple approaches are valid but the team has a preferred pattern. Lower stakes than prohibitions or requirements, but important for consistency.

**Characteristics:** Often expressed as "prefer X over Y" or "default to X unless Z." Preferences encode accumulated team wisdom without being rigid mandates.

**Examples:**
- Prefer `async/await` over Promise chains for readability
- Default to named exports; use default exports only for React page components
- Prefer early returns over deeply nested conditionals
- When choosing between verbosity and cleverness, choose verbosity

**Writing guidance:** Explain the rationale for non-obvious preferences. "Prefer named exports" is a preference. "Prefer named exports because they produce better IDE refactoring support and clearer import statements — default exports only for React pages to match Next.js conventions" is a preference Claude can apply correctly in edge cases.

### Category 4: Constraints

Boundaries that exist for external reasons — compliance, performance, architecture, or organizational policy. Often the hardest to explain but the most important to enforce.

**Characteristics:** Frequently reference external context (a regulation, an architectural decision, a technical limitation). May feel arbitrary without that context, which is why the rationale section matters more here than anywhere else.

**Examples:**
- Do not add new dependencies without a PR comment explaining the tradeoff — we are trying to reduce bundle size
- Do not modify the `payments/` module directly — changes require a security review ticket first
- All user-facing strings must go through the i18n system, even for internal tools — the auditors check this
- Do not access the `user_data` table directly — always go through the `UserRepository` abstraction

**Writing guidance:** Always include the reason. A constraint without a rationale looks like bureaucracy. A constraint with a rationale gets applied intelligently: Claude will understand why the constraint exists and will flag edge cases where the intent of the constraint might conflict with the letter of it.

---

## Part 4 — Walkthroughs: Five Production Rule Files

### Walkthrough 1: Security Policy

The most important rules file in any codebase. This one loads universally — imported directly in the root CLAUDE.md so it is never absent, regardless of what part of the codebase Claude is working in.

**`.claude/rules/security-policy.md`:**
```markdown
# Security Policy

## Purpose
Prevent common security vulnerabilities from being introduced during development.
These rules apply to all code in this repository, all the time.

## Secrets and Credentials

- NEVER hardcode secrets, API keys, passwords, or credentials in source code
- NEVER hardcode them in comments, even as examples
- NEVER log sensitive values — mask them: `log("token:", token.slice(0, 4) + "****")`
- All secrets go in environment variables, referenced via `process.env.SECRET_NAME`
- Local secrets go in `.env.local`, which is gitignored. Never `.env` (not gitignored)

If you need to add a new secret, add it to:
1. `.env.local` for local development
2. The team secrets manager (1Password) for shared access
3. The CI/CD environment variables for production

## Input Validation

- NEVER trust user input without validation
- ALL request bodies must be validated with Zod before processing
- ALL query parameters must be type-checked before use
- ALL file uploads must be validated for type, size, and content before processing
- Validation schemas live in `src/schemas/` — create one per resource

## SQL and Database

- NEVER construct SQL strings by concatenation or template literals
- ALL database queries must use the parameterized query builder at `src/db/query.ts`
- ALL raw Prisma queries must use the `$queryRaw` tagged template syntax, not string concat
- Query results must never be returned directly to clients — always map through a DTO

## Authentication and Authorization

- NEVER expose an endpoint without auth unless it is explicitly marked `@public`
- Authorization checks must happen at the service layer, not just the route layer
- User IDs from the session must be used to scope queries — never trust user-supplied IDs
  for ownership checks

## Dependencies

- NEVER add a dependency with known CVEs — run `npm audit` before adding
- NEVER use `eval()`, `new Function()`, or dynamic `require()`
- Be suspicious of packages with very few downloads or recent ownership changes

## Rationale
These rules exist because security vulnerabilities are expensive to fix after the fact
and catastrophic if exploited. When in doubt, err on the side of more validation and
fewer assumptions about input.
```

**Root `CLAUDE.md` import:**
```markdown
@.claude/rules/security-policy.md
```

### Walkthrough 2: Testing Requirements

Testing rules are most useful when they encode the team's specific quality bar — not just "write tests" but exactly what coverage is expected and what good tests look like in this codebase.

**`.claude/rules/testing-requirements.md`:**
```markdown
# Testing Requirements

## Purpose
Ensure new code ships with sufficient test coverage and that tests are meaningful
rather than tautological.

## Coverage Requirements

- Every new public function requires at minimum:
  - One happy-path test covering the expected use case
  - One error-path test covering the primary failure mode
- New API endpoints require integration tests (not just unit tests)
- New database queries require tests against the test database, not mocks
- Coverage threshold is 80% — do not decrease it

## What Makes a Good Test

A good test:
- Has a descriptive name: `it("returns 404 when user does not exist")` not `it("works")`
- Tests behavior, not implementation: assert on outputs and side effects, not internal calls
- Can actually fail — a test that always passes is worse than no test
- Is independent — does not depend on test execution order or shared mutable state

A bad test:
- Mocks everything and only verifies that mocks were called
- Tests a private implementation detail that could change without breaking behavior
- Has a name like `it("test 1")` or `it("should work")`

## Test Organization

- Unit tests: `tests/unit/<module-name>.test.ts` — test pure functions in isolation
- Integration tests: `tests/integration/<feature>.test.ts` — test full request/response cycles
- Migration tests: `tests/migrations/<migration-name>.test.ts` — verify schema changes
- No test files inside `src/` — all tests live in `tests/`

## Test Data

- Use the factory functions in `tests/factories/` to create test data — do not hardcode IDs
- Integration tests use the test database — never the development or production database
- Clean up test data in `afterEach` or `afterAll` — do not leave orphaned records

## Running Tests

```bash
npm test                    # full suite
npm test -- --watch         # watch mode
npm test -- <filename>      # single file
npm run test:integration    # integration tests only
npm run test:coverage       # with coverage report
```

## What Not to Test

- Auto-generated code in `src/generated/`
- Third-party library internals
- Configuration files
- Type definitions with no runtime behavior
```

### Walkthrough 3: API Design Standards

API rules are particularly valuable for teams that are growing — onboarding a new engineer and having them immediately produce API code that matches your conventions is a concrete productivity win.

**`.claude/rules/api-design-standards.md`:**
```markdown
# API Design Standards

## Purpose
Ensure all API endpoints follow consistent patterns for predictable client behavior
and maintainable server code.

## Response Format

ALL responses use the standard envelope from `src/lib/response.ts`:

Success:
```json
{
  "success": true,
  "data": { ... },
  "timestamp": "2026-01-15T10:30:00Z"
}
```

Error:
```json
{
  "success": false,
  "error": {
    "code": "USER_NOT_FOUND",
    "message": "No user found with the provided ID",
    "details": { }
  },
  "timestamp": "2026-01-15T10:30:00Z"
}
```

NEVER return raw data objects or arrays at the top level.
NEVER expose internal error messages, stack traces, or database errors to clients.

## HTTP Status Codes

Use these consistently:
- `200` — successful GET, PATCH, PUT
- `201` — successful POST that created a resource
- `204` — successful DELETE (no body)
- `400` — validation error (malformed request)
- `401` — unauthenticated (no valid session)
- `403` — unauthorized (authenticated but not permitted)
- `404` — resource not found
- `409` — conflict (duplicate, version mismatch)
- `422` — unprocessable entity (semantically invalid but syntactically correct)
- `500` — unexpected server error (should be rare; fix the cause)

Do not use 200 for errors. Do not use 500 for client mistakes.

## Endpoint Naming

- Use lowercase kebab-case: `/api/v1/user-sessions`, not `/api/v1/userSessions`
- Use plural nouns for resource collections: `/users`, not `/user`
- Use nested routes for owned resources: `/users/:id/posts`, not `/posts?userId=...`
- Actions on resources use POST with a verb suffix: `/users/:id/deactivate`
- API version prefix on all routes: `/api/v1/...`

## Request Validation

- ALL request bodies must have a corresponding Zod schema in `src/schemas/`
- Schemas must be strict: `z.object({...}).strict()` — reject unknown fields
- Validation errors must return 400 with field-level detail:
  ```json
  { "error": { "code": "VALIDATION_ERROR", "fields": { "email": "Invalid email format" } } }
  ```

## Pagination

For any endpoint returning a list:
- Default page size: 20
- Maximum page size: 100
- Use cursor-based pagination for large collections: `{ "cursor": "...", "hasMore": true }`
- Use offset pagination only for small, bounded collections

## Deprecation

When deprecating an endpoint:
1. Add a `Deprecation` header to responses: `Deprecation: true`
2. Add a `Sunset` header with the removal date
3. Add a comment in the route file with the ticket number tracking removal
4. Keep the endpoint functional until the sunset date
```

### Walkthrough 4: Database Conventions

Database rules are a good example of subdirectory-scoped loading. These rules are relevant when working in `src/db/` or `src/repositories/` — they don't need to be in context when Claude is editing a React component.

**`src/db/CLAUDE.md`** (loads when Claude works in this directory):
```markdown
# Database Layer

@../../.claude/rules/database-conventions.md
```

**`.claude/rules/database-conventions.md`:**
```markdown
# Database Conventions

## Purpose
Ensure consistent, safe, and performant database access across the codebase.

## Query Patterns

- ALL database access goes through the Repository pattern in `src/repositories/`
- NEVER query the database directly from a handler or service — always through a repository
- Repository methods must be named descriptively: `findUserByEmail`, not `get` or `query`
- NEVER select `*` — always list the columns you need explicitly

## Prisma Usage

- NEVER use `prisma.$queryRaw` with string concatenation — use the tagged template form:
  ```typescript
  // CORRECT
  prisma.$queryRaw`SELECT * FROM users WHERE id = ${userId}`
  // WRONG — SQL injection risk
  prisma.$queryRaw(`SELECT * FROM users WHERE id = ${userId}`)
  ```
- Always include `select` to limit returned fields — never return full models to the service layer
- Use `prisma.$transaction()` for operations that must be atomic
- Set timeouts on long-running queries: `{ timeout: 5000 }`

## Migrations

- Migration names: `<verb>_<noun>` in snake_case: `add_user_roles`, `create_audit_log`
- NEVER drop a column in the same migration that removes the code that uses it —
  deploy code removal first, then drop the column in a subsequent migration
- NOT NULL constraints on existing tables require a backfill migration first
- All migrations must be reversible unless there is an explicit comment explaining why not

## Indexes

- Add an index for any column used in a `WHERE` clause in a frequent query
- Composite indexes: most selective column first
- Use `CREATE INDEX CONCURRENTLY` for indexes on tables with existing data —
  the blocking form will lock the table

## Naming

- Tables: plural snake_case — `user_sessions`, `audit_logs`
- Columns: snake_case — `created_at`, `external_id`
- Foreign keys: `<referenced_table_singular>_id` — `user_id`, `organization_id`
- Boolean columns: present-tense adjective — `is_active`, `has_verified_email`
- Timestamp columns: past-tense verb — `created_at`, `deleted_at`, `last_seen_at`

## Performance

- NEVER load a collection without a limit — default to 100 rows maximum in repository methods
- N+1 queries are a blocking code review issue — use `include` or batch queries
- Log slow queries (> 500ms) — the query logger middleware handles this automatically
- Avoid SELECT in a loop — batch with `findMany` and `where: { id: { in: ids } }`
```

### Walkthrough 5: Error Handling

Error handling rules are surprisingly high-leverage. Inconsistent error handling is one of the most common causes of hard-to-debug production incidents, and it is exactly the kind of thing that varies by engineer without explicit standards.

**`.claude/rules/error-handling.md`:**
```markdown
# Error Handling

## Purpose
Ensure errors are handled consistently, logged appropriately, and never silently swallowed.

## The Golden Rule

NEVER write an empty catch block. An error that is caught and discarded is harder to
debug than an error that crashes loudly. If you genuinely want to ignore an error,
add a comment explaining why:

```typescript
// WRONG
try {
  await sendAnalyticsEvent(event);
} catch (err) {}

// CORRECT — explicit about intent
try {
  await sendAnalyticsEvent(event);
} catch (err) {
  // Analytics failures are non-critical and should not affect the user flow.
  // The event will be retried by the analytics queue on next flush.
  logger.warn("Analytics event failed", { event, err });
}
```

## Typed Exceptions

Use the typed exception classes from `src/lib/errors.ts` — do not throw raw `Error` objects:

- `ValidationError(message, fields)` — invalid input, will produce a 400 response
- `AuthorizationError(message)` — permission denied, will produce a 403 response
- `NotFoundError(resource, id)` — resource missing, will produce a 404 response
- `ConflictError(message)` — duplicate or version conflict, will produce a 409 response
- `ExternalServiceError(service, message, cause)` — third-party failure, retryable

```typescript
// WRONG
throw new Error("User not found");

// CORRECT
throw new NotFoundError("User", userId);
```

## Where to Handle Errors

- **Handlers:** Catch only typed exceptions you can translate into meaningful responses.
  Let unexpected errors propagate to the global error handler.
- **Services:** Throw typed exceptions for domain errors. Wrap external service calls in
  `ExternalServiceError` so the caller knows the failure is retryable.
- **Repositories:** Translate database errors into typed domain exceptions. Callers should
  never see `PrismaClientKnownRequestError` — translate it first.

## Logging

- Log errors at the point where they become unexpected, not where they are thrown
- Use structured logging: `logger.error("message", { context, err })` — not string concat
- Include enough context to reproduce the issue: user ID, request ID, relevant inputs
- Do not log sensitive data: passwords, tokens, full credit card numbers, SSNs

## Async Error Handling

- NEVER use `Promise.then().catch()` — use `async/await` with `try/catch`
- NEVER use unhandled promise rejections — all async calls must be awaited or explicitly handled
- For fire-and-forget async work, wrap in a helper that logs failures:
  ```typescript
  fireAndForget(sendWelcomeEmail(user));  // in src/utils/async.ts
  ```
```

---

## Part 5 — Advanced Patterns

### Import Composition: Building a Rule Hierarchy

The `@` import syntax in CLAUDE.md enables a clean composition pattern where the root file imports only what needs to be universally active, and subdirectory files import what is domain-specific.

```
project/
├── CLAUDE.md                          ← imports universal rules
├── .claude/
│   └── rules/
│       ├── security-policy.md         ← always imported
│       ├── testing-requirements.md    ← always imported
│       ├── api-design-standards.md    ← imported by src/api/CLAUDE.md
│       ├── database-conventions.md    ← imported by src/db/CLAUDE.md
│       └── error-handling.md          ← always imported
├── src/
│   ├── api/
│   │   └── CLAUDE.md                  ← imports api-design-standards.md
│   └── db/
│       └── CLAUDE.md                  ← imports database-conventions.md
```

**Root `CLAUDE.md`:**
```markdown
# Project: my-api

## Overview
...

## Policies
@.claude/rules/security-policy.md
@.claude/rules/testing-requirements.md
@.claude/rules/error-handling.md
```

**`src/api/CLAUDE.md`:**
```markdown
# API Layer

This directory contains Express route handlers and request validation.

@../../.claude/rules/api-design-standards.md
```

**`src/db/CLAUDE.md`:**
```markdown
# Database Layer

This directory contains Prisma queries and the Repository pattern implementation.

@../../.claude/rules/database-conventions.md
```

The result: Claude always holds the security, testing, and error handling policies. It only holds the API design rules when working in `src/api/`. It only holds the database conventions when working in `src/db/`. Context is scoped, relevant, and never redundant.

### Monorepo Rules Architecture

In a monorepo, the rules hierarchy extends naturally to accommodate both shared organizational policy and package-specific conventions:

```
monorepo/
├── CLAUDE.md                          ← imports org-wide rules
├── .claude/
│   └── rules/
│       ├── security-policy.md         ← org-wide, always loaded
│       ├── commit-conventions.md      ← org-wide, always loaded
│       └── shared-package-policy.md   ← org-wide, always loaded
├── packages/
│   ├── api/
│   │   ├── CLAUDE.md                  ← imports api-specific rules
│   │   └── .claude/
│   │       └── rules/
│   │           ├── api-design.md
│   │           └── database.md
│   └── web/
│       ├── CLAUDE.md                  ← imports web-specific rules
│       └── .claude/
│           └── rules/
│               ├── component-standards.md
│               └── accessibility.md
```

The key discipline: the root rules cover only what is genuinely universal. A security policy that applies to every package belongs at the root. A React component naming convention that applies only to the web package belongs in `packages/web/.claude/rules/`. Resist the instinct to promote package-specific rules to the root level — it inflates context for every session across every package.

### Explicit Precedence Rules

When multiple rule files could conflict — a root-level convention and a package-level convention that disagree on the same point — explicit precedence statements prevent Claude from having to guess which one wins.

Add a precedence note to package-level rules that potentially overlap with root rules:

```markdown
# Web Component Standards

> These standards apply to code in `packages/web/` and take precedence over
> any conflicting conventions in the root CLAUDE.md for this package.

## Component Naming
...
```

And in the root CLAUDE.md, acknowledge that packages may override:

```markdown
## Policies
@.claude/rules/security-policy.md  ← these cannot be overridden by package rules
@.claude/rules/commit-conventions.md

Individual packages may define additional conventions for their own code.
Package-level conventions supplement but do not replace these root policies
unless the package file explicitly states otherwise.
```

### Auditing for Conflicts and Staleness

Rules files are documentation, and documentation drifts. A rule that says "use the logger at `src/utils/logger.ts`" becomes wrong the moment that path changes. A security rule about a dependency becomes wrong when the dependency is removed.

Audit strategies that actually work:

**Quarterly rule reviews.** Add a recurring calendar event. At minimum, verify that all path references in rules still resolve and that all tool or library references are still accurate. Takes 30 minutes on a healthy project.

**Change-triggered reviews.** When a major architectural change lands (new ORM, new API framework, new test runner), the rule files are part of the migration checklist — not an afterthought.

**Test critical rules explicitly.** For rules where violations have serious consequences (security, compliance), manually verify in a session that Claude follows them. Ask Claude directly: "What would you do if I asked you to use a raw SQL string?" The answer tells you whether the rule is landing.

**Own the rules file like code.** Changes to `.claude/rules/security-policy.md` should go through code review, require a signoff from the relevant owner, and have a CHANGELOG entry for significant changes. Treat it like a `.eslintrc` — not like a Notion document.

### The CLAUDE.local.md Pattern for Rule Overrides

For local development scenarios where a rule needs to be temporarily relaxed — pointing at a local dev database instead of the staging one, using a debug logging level, relaxing rate limits for local testing — `CLAUDE.local.md` provides a gitignored override mechanism:

```markdown
# Local Development Overrides

> This file is gitignored. These settings apply to this machine only.
> Do not commit this file.

## Database
Use the local development database at `postgresql://localhost:5432/myapp_dev`
instead of the staging database referenced in the database conventions.

## Logging
Verbose logging is enabled locally. `LOG_LEVEL=debug` is set in `.env.local`.
```

Rules in `CLAUDE.local.md` are read with lower priority than the project rules. Use this pattern for local configuration, not for disabling security policies — the local file should extend the project rules, not contradict them.

---

## Part 6 — Quality and Maintenance

### Writing Rules That Actually Stick

The same principles that make CLAUDE.md effective apply to rules files, but with higher stakes. A rule that Claude misapplies or ignores is worse than no rule — it creates false confidence.

**Make every rule testable.** Before writing a rule, ask: how would I verify that Claude is following this? If you cannot answer that question, the rule is probably too vague. "Write secure code" is not testable. "Use parameterized queries for all database access" is testable.

**Lead with the most important rules.** Claude reads sequentially. The security policy rules that prevent data breaches belong at the top, not buried in a long list. Front-loading critical rules ensures they are processed before Claude takes any action.

**Pair every prohibition with an alternative.** "Never do X — do Y instead" is twice as useful as "never do X." The alternative tells Claude where to go; without it, Claude has to infer the right path, and the inference may be wrong.

**Prefer specificity to comprehensiveness.** A rule file with 10 specific, enforceable rules is more effective than a rule file with 50 vague guidelines. Every rule you add that Claude cannot act on concretely dilutes the rules it can.

### Rules File Quality Checklist

Before committing a rules file:

- [ ] Every rule is specific and measurable, not aspirational
- [ ] Every prohibition has a "do this instead" alternative
- [ ] All path references resolve to files that currently exist
- [ ] All tool and library references match what is actually in the project
- [ ] No contradictions with rules in parent CLAUDE.md or sibling rule files
- [ ] Critical rules are at the top, not buried
- [ ] The file covers exactly one domain (not a grab-bag)
- [ ] Rationale is included for non-obvious rules
- [ ] The file has a clear owner (team or individual) for future maintenance

---

## Part 7 — Reference

### Import Syntax Quick Reference

```markdown
# In CLAUDE.md or any subdirectory CLAUDE.md

# Import a rules file (relative to the current file)
@.claude/rules/security-policy.md

# Import from a parent directory
@../../.claude/rules/shared-policy.md

# Imports are expanded inline at session start
# Recursive imports are supported (up to 5 hops)
# Both relative and absolute paths are supported
```

### Rule Category Decision Tree

```
What kind of rule are you writing?
│
├── Something Claude must NEVER do
│   → Prohibition. Pair with "do Y instead."
│
├── Something Claude must ALWAYS do
│   → Requirement. Make it measurable.
│
├── A preference between valid alternatives
│   → Preference. Include rationale for non-obvious ones.
│
└── A boundary that exists for external reasons
    → Constraint. Always explain why it exists.
```

### Loading Pattern Decision Tree

```
Should this rule always be active?
│
├── YES (security, secrets, commits, error handling)
│   → Import in root CLAUDE.md:
│     @.claude/rules/your-rule.md
│
└── NO (only relevant in a specific part of the codebase)
    │
    ├── Which directory is relevant?
    │   → Add to that directory's CLAUDE.md:
    │     @../../.claude/rules/your-rule.md
    │
    └── Is it relevant across multiple packages in a monorepo?
        → Import in each relevant package's CLAUDE.md
```

### File Organization Quick Reference

```bash
# Create a new rules file
touch .claude/rules/my-domain-policy.md

# Import it from root CLAUDE.md (always active)
echo "@.claude/rules/my-domain-policy.md" >> CLAUDE.md

# Import it from a subdirectory (scoped to that directory)
echo "@../../.claude/rules/my-domain-policy.md" >> src/my-module/CLAUDE.md

# Verify it loads (check session context)
# In a Claude Code session:
/memory
```

---

## Conclusion

Rules files are a forcing function for a discipline that most engineering teams aspire to but rarely achieve: writing down the standards, keeping them current, and applying them consistently. The usual failure mode is that standards exist in someone's head, get documented once in a Confluence page, and then drift out of sync with reality until they are effectively ignored.

The rules system changes the incentive structure. A rules file that is wrong gets noticed immediately — Claude follows the wrong rule and produces incorrect output. That feedback loop is tighter than waiting for a PR review to catch a convention violation, and far tighter than a periodic documentation audit.

The progression worth following as you build out your rules system:

**Start with prohibitions.** Write down the three things that, if Claude did them, would cause you the most pain: committing secrets, using raw SQL strings, returning raw database errors to clients. Write them as simple, specific rules. That alone is valuable.

**Add requirements for your quality bar.** What does "done" actually mean on your team? Tests, types, documentation, review sign-offs? Write those down as requirements. Make them measurable.

**Layer in preferences and constraints as the team grows.** Preferences and constraints become more important as teams scale and individual engineers need to make consistent decisions without synchronous coordination. Rules files are how that coordination scales asynchronously.

**Treat them as infrastructure.** The rules system is as important as your linting configuration, your CI pipeline, or your type system. It deserves the same rigor: version control, code review, clear ownership, and regular audits.

The goal is a codebase where any engineer — human or AI — can work effectively because the standards are explicit, current, and enforced. Rules files are how you get there.

---

*Official references: [Claude Code memory documentation](https://code.claude.com/docs/en/memory) · [CLAUDE.md best practices](https://docs.anthropic.com/en/docs/claude-code/best-practices) · [Anthropic prompt engineering overview](https://docs.claude.com/en/docs/build-with-claude/prompt-engineering/overview)*
