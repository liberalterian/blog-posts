# An Engineer's Introduction to the Model Context Protocol: Architecture, Primitives, and Building Servers

*From the N×M integration problem that motivated MCP, through its three core primitives and transport layer, to a practical walkthrough of building and connecting real MCP servers to Claude Code.*

---

## Introduction

Every AI engineering team eventually hits the same wall. The model is capable. The tools exist. But connecting them requires custom integration code for every combination: GitHub needs its own bridge, the database needs its own bridge, the internal ticketing system needs its own bridge. Three tools, three models, nine integrations. Ten tools, five models, fifty integrations. The combinatorial math gets ugly fast.

This is the **N×M integration problem**, and it is the explicit motivation behind the Model Context Protocol. Anthropic published MCP as an open standard in November 2024. By mid-2025, every major AI platform — OpenAI, Google, Microsoft — had adopted it. In December 2025 it moved to the Linux Foundation under neutral governance. As of 2026 the ecosystem has over 97 million monthly SDK downloads and 13,000+ public servers. It is, in any practical sense, the standard.

MCP solves the N×M problem by inserting a protocol layer between models and tools. Build one MCP server, and any compliant client — Claude, ChatGPT, VS Code Copilot, or your custom agent — can discover and call it without custom glue code. The integration matrix collapses from N×M to N+M.

This guide covers:

1. **The architecture** — clients, servers, hosts, and the session model
2. **The three primitives** — Tools, Resources, and Prompts, and when to use each
3. **The transport layer** — stdio vs. Streamable HTTP, and which to choose
4. **Building servers** — practical walkthroughs in TypeScript and Python
5. **Connecting to Claude Code** — `.mcp.json`, configuration scopes, and CLI commands
6. **Tool annotations and safety** — the behavioral metadata system introduced in the 2025 spec
7. **Advanced patterns** — composition, authentication, and production considerations

By the end you will understand how MCP works at the protocol level, be able to build a functional server from scratch, and know how to wire it into Claude Code at project and personal scope. Let's go.

---

## Part 1 — Architecture

### The Cast of Characters

MCP defines three roles. Every integration involves all three.

**The Host** is the application the user interacts with — Claude Desktop, Claude Code, VS Code with Copilot, or a custom agent you build. The host owns the user session and is responsible for making the final decision about what to do with tool results. It is the orchestration layer.

**The Client** lives inside the host. Each client maintains one persistent, stateful connection to one MCP server. A host can run multiple clients simultaneously — one for GitHub, one for your database, one for your internal API — and the user interacts with all of them through a single interface. The host-to-client relationship is one-to-many.

**The Server** is the external program that exposes capabilities. A server declares what it can do (tools, resources, prompts), executes requests, and returns results. Servers can be local processes on your machine or remote services in the cloud. They are purpose-built and narrowly scoped — one server for GitHub, one for Postgres, one for your analytics platform.

### The Session Model

MCP is a *stateful* protocol. This is the key architectural difference from a REST API, where every request is independent. An MCP session has a lifecycle:

1. **Initialization** — The client and server negotiate the protocol version and exchange capability declarations. A client that does not support prompt templates will not receive them. A server that does not support resource subscriptions will not advertise them. The session is configured to the intersection of both sides' capabilities.

2. **Operation** — Messages flow bidirectionally according to the negotiated capabilities. The client can discover tools (`what tools do you have?`), invoke them, request resources, and use prompt templates. The server executes requests and can proactively send notifications (resource updates, log events).

3. **Termination** — One side initiates shutdown. The transport layer signals the end of the session — no special JSON-RPC shutdown message is defined.

All communication uses **JSON-RPC 2.0** as the message format. Every request has an `id`, a `method`, and optional `params`. Every response has the same `id` and either a `result` or an `error`. This is the wire format MCP runs over, regardless of which transport is used.

### Why Stateful Matters

Stateful sessions are more expensive than stateless requests, but they unlock patterns that REST cannot support cleanly: multi-step tool execution where the server maintains context between calls, resource subscriptions where the server pushes updates when data changes, and capability negotiation so clients and servers can evolve independently without breaking existing integrations.

For AI agents doing multi-step work — read a file, make a change, verify the result, commit — the stateful session model maps naturally to how agents actually operate. The connection persists across the entire workflow rather than requiring the client to re-authenticate and re-negotiate for every individual action.

---

## Part 2 — The Three Primitives

MCP servers expose capabilities through three primitives. Understanding when to use each one is the core design skill in MCP server development.

### Tools: Things Claude Can Do

Tools are executable functions. They take typed inputs, perform some action or computation, and return results. They are the action surface of an MCP server.

A tool can do anything a function can do: call an external API, run a database query, write a file, trigger a CI pipeline, send a Slack message, look up a stock price. The model decides when to call a tool based on the tool's name and description. It constructs the required inputs, calls the tool, receives the output, and incorporates it into its reasoning.

**Tools are the primitive for side effects.** If your server needs to *do* something in the world — modify state, trigger an action, retrieve live data — that belongs in a tool.

The key design principle: tool descriptions are the routing mechanism. The model reads the description and decides whether the tool is relevant to the current task. This is structurally identical to how skill descriptions work in Claude Code. Precise, task-specific descriptions outperform vague domain descriptions every time.

### Resources: Context Claude Can Read

Resources are data endpoints. They provide content — files, database records, API responses, configuration — that the model can read and incorporate into its context. The distinguishing characteristic: resources return data but do not execute side effects.

Resources are identified by URIs (`file:///path/to/file`, `postgres://mydb/table/users`, `https://api.example.com/config`). A server declares what resources it exposes; the client can list them and retrieve their content. Resources can also be *subscribed to*, so the server can push an update notification when the underlying data changes.

**Resources are the primitive for context injection.** If your server needs to make reference material available to the model — current state of a system, content of a file, records from a database — that belongs in a resource, not a tool.

The practical distinction from Tools: reading a database record is a resource. Updating that record is a tool. Fetching the current configuration is a resource. Deploying a new configuration is a tool.

### Prompts: Reusable Workflows

Prompts are pre-built prompt templates that encode best practices for working with a specific server or domain. They are parameterizable — a server can define a prompt that accepts arguments and returns a fully constructed message ready to send to the model.

Think of prompts as instruction manuals that ship alongside your tools. If your server wraps a complex internal API with non-obvious usage patterns, a prompt template can teach the model how to query it effectively without the user having to know the API's quirks.

Prompts are less commonly used than tools and resources, but they are genuinely useful when a workflow is complex enough that you want a pre-built template for it — code review procedures, incident response workflows, structured reporting formats.

### The Decision Matrix

| | **Tools** | **Resources** | **Prompts** |
|---|---|---|---|
| **Has side effects** | Yes | No | No |
| **Returns data** | Yes | Yes | Template/messages |
| **Model-invoked** | Yes (model decides when) | Yes (model retrieves) | Yes (model applies) |
| **Use for** | Actions, computation, live data | Context, reference, state | Reusable workflows |

When in doubt: if it *does* something, it's a Tool. If it *provides* something, it's a Resource. If it *teaches* something, it's a Prompt.

---

## Part 3 — The Transport Layer

The transport layer is responsible for moving JSON-RPC messages between client and server. MCP currently defines two active transports.

### stdio — Local Processes

Standard input/output transport. The host spawns the server as a child process and communicates over the process pipes. Messages flow in over stdin, responses come back over stdout, and stderr is available for logging.

stdio is the right choice for:
- Local development and testing
- Desktop integrations (Claude Desktop, Claude Code)
- Tools that need direct access to the local filesystem
- Any server that should not be exposed to the network

The operational model is dead simple: no network configuration, no TLS, no auth headers. You define the server binary and its arguments in the client configuration; the client handles spawning and teardown. One process per client connection, no shared state between clients.

### Streamable HTTP — Remote Services

A single HTTP endpoint handles bidirectional messaging. This is the standard for remote MCP servers, multi-user deployments, and cloud-hosted services. The 2025 spec revision formalized Streamable HTTP as the remote transport standard, replacing the earlier SSE (Server-Sent Events) transport which was deprecated in June 2025.

Streamable HTTP is the right choice for:
- Shared team infrastructure (one server, many clients)
- Cloud-hosted tools and internal APIs
- Anything that needs to run in a container or managed service
- Services requiring authentication (OAuth 2.1, API keys via headers)

The Spring 2025 spec update added OAuth 2.0 support, which is now the standard authentication mechanism for remote MCP servers. API key authentication via request headers works as a simpler alternative for internal tools.

### Choosing Your Transport

The decision is almost always straightforward: local tool running on the developer's machine → stdio; shared service running in your infrastructure → Streamable HTTP. A local Postgres inspector is stdio. A shared knowledge base API is Streamable HTTP. You can mix them freely — Claude Code will happily maintain simultaneous connections to both a local stdio server and a remote HTTP server in the same session.

---

## Part 4 — Building Servers

### Setup and SDK Selection

Two official SDKs exist. Choose based on your backend language.

**TypeScript SDK** (`@modelcontextprotocol/sdk`) — The Tier 1 implementation. Most feature-complete, largest ecosystem, over 66 million npm downloads and 27,000+ dependent packages. Choose this if your stack is Node/TypeScript.

**Python SDK** (`mcp`) — The recommended choice for Python stacks. Includes FastMCP, a decorator-based high-level API that uses Python type hints and docstrings to automatically generate tool schemas. The fastest path from zero to a working server.

We'll build the same server in both languages — a practical code review tool that analyses a file for common issues and exposes current project configuration as a resource.

### TypeScript: Project Setup

```bash
mkdir mcp-code-analyzer && cd mcp-code-analyzer
npm init -y
npm install @modelcontextprotocol/sdk zod
npm install -D typescript @types/node
npx tsc --init
```

Update `tsconfig.json`:
```json
{
  "compilerOptions": {
    "target": "ES2022",
    "module": "Node16",
    "moduleResolution": "Node16",
    "outDir": "./dist",
    "strict": true
  }
}
```

### TypeScript: The Server

```typescript
// src/index.ts
import { McpServer } from "@modelcontextprotocol/sdk/server/mcp.js";
import { StdioServerTransport } from "@modelcontextprotocol/sdk/server/stdio.js";
import { z } from "zod";
import { readFileSync, existsSync } from "fs";

const server = new McpServer({
  name: "code-analyzer",
  version: "1.0.0",
});

// ─── TOOL: Analyze a file for common issues ───────────────────────────────────
server.registerTool(
  "analyze_file",
  {
    title: "Analyze File",
    description:
      "Analyze a source file for common code quality issues: missing error handling, " +
      "console.log statements, TODO comments, and functions exceeding 40 lines. " +
      "Use when asked to review or audit a specific file.",
    inputSchema: {
      path: z.string().describe("Absolute or relative path to the file to analyze"),
      strict: z
        .boolean()
        .optional()
        .describe("If true, treat TODOs as errors rather than warnings. Default: false"),
    },
    annotations: {
      readOnlyHint: true,       // does not modify files
      openWorldHint: false,     // operates only on local filesystem
    },
  },
  async ({ path, strict = false }) => {
    if (!existsSync(path)) {
      return {
        content: [{ type: "text", text: `Error: File not found at path: ${path}` }],
        isError: true,
      };
    }

    const source = readFileSync(path, "utf-8");
    const lines = source.split("\n");
    const issues: string[] = [];

    // Check for console.log statements
    lines.forEach((line, i) => {
      if (line.includes("console.log")) {
        issues.push(`Line ${i + 1}: console.log found — use the project logger instead`);
      }
    });

    // Check for TODO/FIXME comments
    lines.forEach((line, i) => {
      if (/\/\/\s*(TODO|FIXME)/i.test(line)) {
        const severity = strict ? "ERROR" : "WARNING";
        issues.push(`Line ${i + 1}: [${severity}] ${line.trim()}`);
      }
    });

    // Check for bare catch blocks
    lines.forEach((line, i) => {
      if (/catch\s*\(\s*\w+\s*\)\s*\{\s*$/.test(line)) {
        const nextLine = lines[i + 1]?.trim();
        if (!nextLine || nextLine === "}") {
          issues.push(`Line ${i + 1}: Empty catch block — error is silently swallowed`);
        }
      }
    });

    const summary =
      issues.length === 0
        ? `✅ No issues found in ${path}`
        : `Found ${issues.length} issue(s) in ${path}:\n\n${issues.join("\n")}`;

    return { content: [{ type: "text", text: summary }] };
  }
);

// ─── TOOL: List functions in a file ───────────────────────────────────────────
server.registerTool(
  "list_functions",
  {
    title: "List Functions",
    description:
      "List all function definitions in a file with their line numbers and approximate " +
      "line counts. Use to identify long functions or get a structural overview.",
    inputSchema: {
      path: z.string().describe("Path to the source file"),
    },
    annotations: {
      readOnlyHint: true,
      openWorldHint: false,
    },
  },
  async ({ path }) => {
    const source = readFileSync(path, "utf-8");
    const lines = source.split("\n");
    const functions: string[] = [];

    lines.forEach((line, i) => {
      // Match function declarations and arrow functions assigned to const
      if (/^(export\s+)?(async\s+)?function\s+\w+/.test(line) ||
          /^(export\s+)?const\s+\w+\s*=\s*(async\s+)?\(/.test(line)) {
        functions.push(`Line ${i + 1}: ${line.trim()}`);
      }
    });

    const result =
      functions.length === 0
        ? "No function definitions found"
        : functions.join("\n");

    return { content: [{ type: "text", text: result }] };
  }
);

// ─── RESOURCE: Project configuration ──────────────────────────────────────────
server.registerResource(
  "project-config",
  "file://project/config",
  {
    title: "Project Configuration",
    description:
      "Current project configuration: linting rules, TypeScript settings, " +
      "test configuration, and code quality thresholds.",
    mimeType: "application/json",
  },
  async () => {
    // In a real server this would read from actual config files.
    // Here we construct a representative summary.
    const config = {
      typescript: {
        strict: true,
        noImplicitAny: true,
        target: "ES2022",
      },
      linting: {
        maxFunctionLines: 40,
        noConsoleLog: true,
        requireErrorHandling: true,
        todosAreErrors: false,
      },
      testing: {
        framework: "jest",
        coverageThreshold: 80,
        requireIntegrationTests: true,
      },
    };

    return {
      contents: [
        {
          uri: "file://project/config",
          text: JSON.stringify(config, null, 2),
          mimeType: "application/json",
        },
      ],
    };
  }
);

// ─── PROMPT: Code review template ─────────────────────────────────────────────
server.registerPrompt(
  "review_file",
  {
    title: "File Review",
    description: "Generate a structured code review for a specific file",
    argsSchema: {
      path: z.string().describe("Path to the file to review"),
      focus: z
        .string()
        .optional()
        .describe("Specific concern to focus on (e.g., 'security', 'performance')"),
    },
  },
  ({ path, focus }) => ({
    messages: [
      {
        role: "user",
        content: {
          type: "text",
          text: [
            `Please review the file at ${path}.`,
            focus ? `Focus particularly on: ${focus}.` : "",
            "Use the analyze_file tool first to check for automated issues.",
            "Then provide: a one-paragraph summary, a list of required changes (if any),",
            "and a list of optional suggestions. Be concrete — reference specific lines.",
          ]
            .filter(Boolean)
            .join(" "),
        },
      },
    ],
  })
);

// ─── Connect via stdio ─────────────────────────────────────────────────────────
const transport = new StdioServerTransport();
await server.connect(transport);
console.error("code-analyzer MCP server running on stdio");
```

Add to `package.json`:
```json
{
  "scripts": {
    "build": "tsc",
    "start": "node dist/index.js"
  },
  "type": "module"
}
```

Build: `npm run build`

### Python: The Same Server with FastMCP

The Python SDK's FastMCP pattern is the fastest path to a working server. Type hints become input schemas automatically; docstrings become descriptions.

```bash
pip install "mcp[cli]"
# or with uv (recommended)
uv add "mcp[cli]"
```

```python
# server.py
import json
import re
from pathlib import Path
from mcp.server.fastmcp import FastMCP

mcp = FastMCP("code-analyzer")


# ─── TOOL: Analyze a file ──────────────────────────────────────────────────────
@mcp.tool(annotations={"readOnlyHint": True, "openWorldHint": False})
def analyze_file(path: str, strict: bool = False) -> str:
    """
    Analyze a source file for common code quality issues: missing error handling,
    console.log statements, TODO comments, and functions exceeding 40 lines.
    Use when asked to review or audit a specific file.
    """
    file = Path(path)
    if not file.exists():
        return f"Error: File not found at path: {path}"

    lines = file.read_text().splitlines()
    issues: list[str] = []

    for i, line in enumerate(lines, start=1):
        if "console.log" in line:
            issues.append(f"Line {i}: console.log found — use the project logger instead")

        if re.search(r"#\s*(TODO|FIXME)", line, re.IGNORECASE):
            severity = "ERROR" if strict else "WARNING"
            issues.append(f"Line {i}: [{severity}] {line.strip()}")

    if not issues:
        return f"✅ No issues found in {path}"

    return f"Found {len(issues)} issue(s) in {path}:\n\n" + "\n".join(issues)


# ─── TOOL: List functions ──────────────────────────────────────────────────────
@mcp.tool(annotations={"readOnlyHint": True, "openWorldHint": False})
def list_functions(path: str) -> str:
    """
    List all function definitions in a file with line numbers.
    Use to get a structural overview or identify long functions.
    """
    lines = Path(path).read_text().splitlines()
    functions = []

    for i, line in enumerate(lines, start=1):
        if re.match(r"^(async\s+)?def\s+\w+", line.strip()):
            functions.append(f"Line {i}: {line.strip()}")

    return "\n".join(functions) if functions else "No function definitions found"


# ─── RESOURCE: Project configuration ──────────────────────────────────────────
@mcp.resource(
    uri="file://project/config",
    name="Project Configuration",
    description="Current project configuration: linting rules, type checking settings, and test thresholds.",
    mime_type="application/json",
)
def project_config() -> str:
    config = {
        "python": {"type_checking": "strict", "min_version": "3.10"},
        "linting": {
            "max_function_lines": 40,
            "require_docstrings": True,
            "todos_are_errors": False,
        },
        "testing": {
            "framework": "pytest",
            "coverage_threshold": 80,
            "require_integration_tests": True,
        },
    }
    return json.dumps(config, indent=2)


# ─── PROMPT: Code review template ─────────────────────────────────────────────
@mcp.prompt(name="review_file", description="Generate a structured code review for a file")
def review_file(path: str, focus: str = "") -> str:
    parts = [f"Please review the file at {path}."]
    if focus:
        parts.append(f"Focus particularly on: {focus}.")
    parts.extend([
        "Use the analyze_file tool first to check for automated issues.",
        "Then provide: a one-paragraph summary, required changes (if any), and optional suggestions.",
        "Be concrete — reference specific lines.",
    ])
    return " ".join(parts)


if __name__ == "__main__":
    mcp.run()  # defaults to stdio transport
```

Run locally: `python server.py`

Both servers expose identical capabilities: two tools, one resource, one prompt. The TypeScript version is more explicit about schema types; the Python version derives schema from type hints automatically. In production you would swap the mock config reader and regex-based analysis for real implementations, but the MCP plumbing is production-ready as written.

---

## Part 5 — Connecting to Claude Code

### The Configuration System

Claude Code stores MCP configuration at three scope levels, each in a different file:

**User scope** (`~/.claude.json`) — Personal servers available across all projects on your machine. Your private database inspector, your personal utility scripts. Not shared with teammates.

**Project scope** (`.mcp.json` at the repo root) — Team-shared configuration, committed to version control. Every engineer who clones the repo gets these servers automatically. The right place for shared infrastructure: GitHub, your internal APIs, the project database.

**Local scope** (entry in `~/.claude.json` under the project path) — Project-specific but private. Useful for local overrides: pointing at a local dev instance instead of staging, or temporarily adding a server you are testing.

Priority order: local overrides project, project overrides user.

### The `.mcp.json` Format

```json
{
  "mcpServers": {
    "code-analyzer": {
      "type": "stdio",
      "command": "node",
      "args": ["dist/index.js"],
      "cwd": "/path/to/mcp-code-analyzer"
    },
    "github": {
      "type": "http",
      "url": "https://mcp.github.com",
      "headers": {
        "Authorization": "Bearer ${GITHUB_TOKEN}"
      }
    },
    "postgres": {
      "type": "stdio",
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-postgres"],
      "env": {
        "DATABASE_URL": "${DATABASE_URL}"
      }
    }
  }
}
```

The `${VAR}` syntax expands environment variables at runtime. Secrets never live in the JSON file — they live in your environment and are referenced by name. This is non-negotiable for anything committed to version control.

### CLI Commands

```bash
# Add a stdio server
claude mcp add code-analyzer --transport stdio -- node dist/index.js

# Add a remote HTTP server
claude mcp add github --transport http https://mcp.github.com

# Add a server at project scope (team-shared)
claude mcp add postgres --scope project --transport stdio \
  -- npx -y @modelcontextprotocol/server-postgres

# Import all servers from Claude Desktop
claude mcp add-from-claude-desktop

# List configured servers
claude mcp list

# Remove a server
claude mcp remove code-analyzer

# Check server status in a session
/mcp
```

The `/mcp` command within a session shows which servers are connected, their tool lists, and any connection errors. This is the first place to look when a server is not behaving as expected.

### Debugging Connection Issues

Three usual culprits when a server does not appear or fails silently:

**JSON syntax error in `.mcp.json`** — run `cat .mcp.json | python3 -m json.tool` to validate. A trailing comma or missing bracket will silently prevent the file from parsing.

**Command does not resolve** — Claude Code may run with a different `PATH` than your terminal. Use absolute paths for binaries in production configurations. `which node` gives you the absolute path.

**Server crashes on startup** — run the server command manually in a terminal to see stderr. `node dist/index.js` from the project directory will show startup errors that Claude Code suppresses.

A minimal working `.mcp.json` for most Claude Code setups:
```json
{
  "mcpServers": {
    "github": {
      "type": "http",
      "url": "https://mcp.github.com",
      "headers": { "Authorization": "Bearer ${GITHUB_TOKEN}" }
    },
    "context7": {
      "type": "stdio",
      "command": "npx",
      "args": ["-y", "@upstash/context7-mcp"]
    }
  }
}
```

GitHub gives Claude awareness of your repository state — issues, PRs, branch status. Context7 provides up-to-date library documentation. These two cover the most common gaps in default Claude Code. Add additional servers as specific workflows demand them.

---

## Part 6 — Tool Annotations

Tool annotations were added in the 2025-03-26 specification revision. They are behavioral metadata — hints about a tool's side effects that help clients make smarter decisions about confirmation prompts, retry logic, and risk assessment.

The key word is *hints*. Annotations are not enforced at the SDK level. They are declarations that you as the server author make about your tool's behavior. Clients that trust your server can use them to improve UX. Do not confuse them with access control — they are not a security boundary.

### The Five Annotation Fields

**`title`** — A human-readable display name for UIs. No behavioral implications.

**`readOnlyHint`** — Set to `true` if the tool does not modify any state. Read-only tools can execute without a confirmation prompt. Only set this if the tool *genuinely cannot modify state* — a "search" tool that also writes to an analytics log is not read-only.

**`destructiveHint`** — Set to `true` if the tool may perform destructive updates: deleting records, overwriting files, dropping tables. Only meaningful when `readOnlyHint` is `false`. Clients like VS Code show confirmation dialogs for tools marked destructive. Default is `true` — tools are assumed destructive unless stated otherwise.

**`idempotentHint`** — Set to `true` if calling the tool multiple times with the same arguments produces the same result. Useful for retry logic: idempotent tools can be safely retried on failure; non-idempotent tools cannot. `DELETE /records/123` is idempotent (the record is gone either way); `POST /records` is not (each call creates a new record).

**`openWorldHint`** — Set to `true` if the tool interacts with external systems: the internet, third-party APIs, email systems. This is a signal to the client that results may contain untrusted data — relevant for prompt injection risk. Default is `true`. A tool that operates only on local files or a closed internal database should set this to `false`.

### Annotations in Practice

```typescript
// TypeScript
server.registerTool(
  "delete_record",
  {
    title: "Delete Database Record",
    description: "Permanently delete a record from the database by ID.",
    inputSchema: { id: z.string() },
    annotations: {
      readOnlyHint: false,
      destructiveHint: true,    // irreversible
      idempotentHint: true,     // deleting an already-deleted record is a no-op
      openWorldHint: false,     // operates on internal DB only
    },
  },
  async ({ id }) => { /* ... */ }
);
```

```python
# Python
@mcp.tool(annotations={
    "readOnlyHint": False,
    "destructiveHint": True,
    "idempotentHint": True,
    "openWorldHint": False
})
def delete_record(id: str) -> str:
    """Permanently delete a record by ID."""
    ...
```

The investment is low — four boolean fields — and the benefit compounds as clients get better at using annotation data. Current VS Code Copilot behavior: tools marked `readOnlyHint: true` execute without a confirmation prompt; all other tools require user approval. That alone is a meaningful UX improvement for read-heavy MCP servers.

---

## Part 7 — Advanced Patterns

### Composing Multiple Servers

MCP's N+M architecture means you compose capabilities by connecting multiple focused servers, not by building one monolithic server that does everything. A practical engineering environment might use:

- `github` — PRs, issues, branch status, code search
- `postgres` — read access to the development database
- `code-analyzer` — your custom static analysis tools
- `internal-api` — your team's internal platform API
- `slack` — posting to channels, reading thread history

Claude Code maintains simultaneous connections to all of them. During a session, Claude can cross-reference a GitHub issue, query the database to understand current state, run your custom analyzer on the relevant files, and post a summary to Slack — all in a single workflow, pulling from different servers transparently.

The design principle: keep servers narrow and composable. A server that does one thing well is easier to maintain, easier to secure, and easier to reason about than a server that does everything.

### Authentication for Remote Servers

OAuth 2.1 is the standard for remote server authentication as of the 2025 spec update. For internal team servers where OAuth is overkill, API key authentication via request headers is the practical alternative:

```json
{
  "mcpServers": {
    "internal-api": {
      "type": "http",
      "url": "https://mcp.internal.company.com",
      "headers": {
        "Authorization": "Bearer ${INTERNAL_API_KEY}",
        "X-Team-ID": "${TEAM_ID}"
      }
    }
  }
}
```

The environment variable pattern (`${VAR}`) is how secrets stay out of version-controlled configuration. Engineers set their own credentials in their shell environment; the `.mcp.json` references variable names only.

### The `.mcpb` Bundle Format

For distributing packaged MCP servers — particularly for teams or marketplace distribution — the MCP spec introduced the `.mcpb` (MCP Bundle) format in late 2025. A bundle packages the server code, its dependencies, and a manifest into a single portable file. Distribution becomes `install bundle.mcpb` rather than `npm install && configure`. This is relevant primarily for server authors who want to distribute servers to other teams or publicly.

### Resource Subscriptions

For servers backed by live data sources, resource subscriptions allow the server to push notifications when data changes. The client subscribes to a resource URI and receives update events without polling:

```typescript
// Server notifies client when a resource changes
server.sendResourceUpdated({ uri: "postgres://mydb/config" });
```

This enables patterns like: Claude's context stays synchronized with live database state during a long session, without the client having to periodically re-fetch.

### Performance and Scale Considerations

A few operational notes for production deployments:

**Keep tool lists short.** Claude Code documentation explicitly notes that a short tool list keeps the agent focused. Five or six well-chosen servers covering real workflow needs outperform twenty servers with overlapping capabilities. Prune servers that are not regularly used.

**Prefer absolute paths in production configs.** The `PATH` environment in which Claude Code spawns stdio servers may differ from your interactive shell. Use `which node`, `which python3`, `which npx` to get absolute paths and hardcode them in production configurations.

**Fail loudly on startup.** A server that silently fails to connect to its backend on startup creates confusing tool failures later. Validate connections during initialization and fail with a clear error message if required resources are unavailable.

**Use separate servers for separate concerns.** A server that handles both database reads and Slack notifications is harder to scope permissions for, harder to maintain, and harder to reason about in incident scenarios. One server, one concern.

---

## Part 8 — Reference

### Primitive Selection Guide

```
What does your server need to do?
│
├── Perform an action or computation
│   (write, delete, call API, trigger pipeline, run a query) → Tool
│
├── Provide read-only data for context
│   (file contents, DB records, config, documentation)    → Resource
│
└── Encode a reusable workflow or instruction set
    (complex query template, review procedure, reporting format) → Prompt
```

### Transport Selection Guide

```
Where does the server run?
│
├── Local machine (developer tool, filesystem access, desktop integration) → stdio
│
└── Remote / shared infrastructure (cloud, team service, multi-user)      → Streamable HTTP
    │
    └── Needs authentication?
        ├── Simple internal tool → API key in headers + ${ENV_VAR} pattern
        └── Public or cross-org service → OAuth 2.1
```

### Configuration Quick Reference

```bash
# Add stdio server (project scope, team-shared)
claude mcp add my-server --scope project --transport stdio -- node /path/to/dist/index.js

# Add HTTP server with auth
claude mcp add my-api --transport http https://mcp.example.com \
  --header "Authorization: Bearer ${MY_API_KEY}"

# Check what's connected in current session
/mcp

# List all configured servers
claude mcp list

# Debug: run server manually to see startup errors
node /path/to/dist/index.js
python3 /path/to/server.py
```

### Tool Annotations Quick Reference

| Annotation | Type | Default | Set to `true` when... |
|---|---|---|---|
| `readOnlyHint` | boolean | false | Tool reads only, never writes |
| `destructiveHint` | boolean | true | Tool may delete or overwrite data |
| `idempotentHint` | boolean | false | Repeated calls with same args are safe |
| `openWorldHint` | boolean | true | Tool reaches external APIs or internet |

---

## Conclusion

MCP is infrastructure. Like a database driver or an HTTP library, it is not the interesting part of what you are building — it is the plumbing that makes the interesting parts possible. The protocol itself is not complex: three primitives, two transports, a JSON-RPC wire format, and a stateful session model. The value is in what becomes possible once the plumbing is in place.

The progression worth keeping in mind:

**Start with consuming existing servers.** Add the GitHub MCP server to Claude Code, connect your project database, install Context7 for library documentation. Feel the difference in what Claude can do with those tool surfaces before writing a single line of server code.

**Build your first server for a real gap.** What does Claude repeatedly not know about your environment? What do you find yourself pasting into every session? Start there. A 50-line FastMCP server that answers that one question correctly is more valuable than a feature-complete server that answers hypothetical questions.

**Add primitives deliberately.** Most servers need more Tools than Resources, and very few need Prompts at all. Add what the actual workflow demands, not what the spec allows. A server with three tightly-scoped, well-annotated tools beats a server with fifteen loosely-defined ones.

**Compose, don't consolidate.** The instinct to build one server that handles everything is wrong. Narrow servers are easier to reason about, easier to secure, and easier to replace when a better option appears. The MCP ecosystem already has servers for most common tools — your job is usually integration and composition, not rebuilding what already exists.

The model is capable. The tools exist. Now the protocol to connect them cleanly is stable, widely adopted, and well-supported. The plumbing work, for once, is worth doing.

---

*Official references: [Model Context Protocol Specification](https://modelcontextprotocol.io/specification) · [MCP TypeScript SDK](https://github.com/modelcontextprotocol/typescript-sdk) · [MCP Python SDK](https://github.com/modelcontextprotocol/python-sdk) · [Claude Code MCP Configuration](https://docs.claude.ai/code/mcp)*
