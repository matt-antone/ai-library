---
name: ingest
description: Ingest Codebase into Memory
---

# Ingest Codebase into Memory

You are a codebase analyst. Your job is to read source code and produce **agent-optimized memories** that reduce context window usage in future sessions.

## Trigger

This command runs when the user says "ingest", "learn this codebase", "memorize this project", "onboard to this repo", or similar.

## Input

The user may specify:
- A directory or module path (e.g. `src/billing/`, `.`, `lib/auth`)
- A project root (defaults to current working directory)
- A namespace workspace_id (defaults to the absolute path of the current working directory)

If no path is given, ingest the entire project starting from the working directory root.

## Prerequisites

1. Confirm `memory.*` MCP tools are available. If not, stop and tell the user.
2. Search memory for existing ingestion of this workspace to avoid duplicates:
   ```json
   {
     "query": "codebase ingestion navigation patterns",
     "namespace": { "scope": "workspace", "workspace_id": "<absolute-path>" }
   }
   ```
3. If prior ingestion exists, ask the user whether to refresh/update or skip already-covered areas.

## Ingestion process

### Phase 1: Survey

1. Use Glob to discover the project structure — source dirs, config files, test dirs, docs.
2. Read `package.json`, `Cargo.toml`, `pyproject.toml`, `go.mod`, or equivalent to understand the stack.
3. Read any existing `README.md`, `CLAUDE.md`, `AGENTS.md`, or similar project docs.
4. Identify the top-level modules/directories that represent distinct concerns.

### Phase 2: Analyze each module

For each significant module or directory, read the source files and reason about:

- **What it does** (purpose, responsibilities)
- **How it connects** to other modules (imports, shared types, data flow)
- **What patterns it follows** (conventions, naming, error handling style)
- **What would surprise a newcomer** (gotchas, non-obvious constraints, workarounds)
- **What config/env it depends on**

### Phase 3: Write memories

Produce memories in these categories. Each memory should be self-contained, concise, and optimized for agent retrieval — not human prose.

#### 3a. Project overview (one memory)

Write a single high-level `kind: "fact"` memory summarizing:
- Tech stack and language
- Module/directory structure (brief map)
- Build/test/run commands
- Deployment model if apparent

Tags: `["ingestion", "overview", "architecture"]`
Importance: `0.9`

#### 3b. Navigation maps (one per major module)

For each module, write a `kind: "fact"` memory answering "where is X?":
- Key files and what they contain
- Entry points
- Shared types or interfaces
- Test file locations

Keep it dense. Example format:
```
Auth module (src/auth/):
- middleware.ts: Express middleware, checks JWT, attaches req.user
- token-manager.ts: issue/refresh/revoke JWT tokens, uses jsonwebtoken
- types.ts: AuthUser, TokenPayload, AuthConfig interfaces
- tests: tests/auth/*.test.ts
- Depends on: src/config (JWT_SECRET), src/db (users table)
```

Tags: `["ingestion", "navigation", "<module-name>"]`
Importance: `0.8`

#### 3c. Pattern library (one memory if patterns are consistent, per-module if they vary)

Capture the "how we do things here" conventions:
- How new routes/endpoints/handlers are added
- Error handling pattern
- Database access pattern
- Test structure and conventions
- Import/export conventions

Tags: `["ingestion", "patterns"]`
Importance: `0.8`

#### 3d. Dependency/relationship map (one memory)

Capture cross-module relationships:
- Which modules depend on which
- Data flow direction
- Shared state or singletons
- External service integrations

Tags: `["ingestion", "dependencies"]`
Importance: `0.75`

#### 3e. Gotchas and constraints (one memory, only if non-obvious issues exist)

Things that would cause an agent to fail or waste tokens:
- Non-obvious constraints (e.g. "ORM doesn't support nested transactions")
- Required env vars that aren't documented
- Order-of-operations requirements
- Known broken or deprecated areas

Tags: `["ingestion", "gotchas"]`
Importance: `0.85`

#### 3f. Config decoder ring (one memory, only if config is non-trivial)

What env vars and config files control, and their relationships:
- Which vars are required vs optional
- What values change behavior
- Test vs prod differences

Tags: `["ingestion", "config"]`
Importance: `0.7`

### Phase 4: Link and summarize

1. Link all module navigation maps back to the project overview using `edge_type: "belongs_to"`.
2. Link the pattern library to the overview using `edge_type: "related_to"`.
3. Link gotchas to the relevant module navigation maps using `edge_type: "related_to"`.
4. Write a final `kind: "summary"` memory noting what was ingested, how many memories were created, and the namespace used.

## Namespace

All memories use:
```json
{
  "scope": "workspace",
  "workspace_id": "<absolute-path-of-project-root>",
  "topic": "ingestion"
}
```

## Quality rules

- **Dense over verbose**: Pack information tightly. Agents don't need narrative.
- **Self-contained**: Each memory must make sense without reading the others.
- **Stable facts only**: Don't memorize things that change with every commit (line numbers, current variable values, WIP code).
- **Skip the obvious**: Don't write memories for things an agent can trivially discover with a single glob or grep (e.g. "there is a file called index.ts").
- **Relationships matter most**: The highest-value memories capture what connects to what and why — things that require reading multiple files to understand.

## Output

When done, report:
- Number of memories written
- Modules/areas covered
- The namespace used
- Any areas skipped and why (e.g. vendor dirs, generated code)
- Suggested follow-up (e.g. "run again targeting `src/legacy/` if you want coverage of the deprecated modules")

