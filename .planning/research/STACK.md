# Stack Research

**Domain:** AI skill toolkit for dev-tools (Docker Compose + Taskfile generator)
**Researched:** 2026-04-10
**Confidence:** HIGH — all findings verified against official docs and live schemas

---

## Recommended Stack

### Core Technologies

| Technology | Version | Purpose | Why Recommended |
|------------|---------|---------|-----------------|
| `SKILL.md` format | Current (2026) | AI skill definition | Official format for both GitHub Copilot CLI and Claude agent skills. Same spec, same file, zero divergence. |
| Docker Compose Spec | v2 (no `version:` key) | Generated service configs | The `version:` top-level field is officially deprecated/obsolete. Current spec uses `compose.yaml` with no version declaration. |
| Taskfile | v3 (current: v3.49.1) | Generated task runner configs | Official schema version `'3'`, JSON schema published at `taskfile.dev/schema.json`. Supports `includes` for modular per-service files. |
| YAML | 1.2 | All file formats | Compose and Taskfile are both YAML 1.2. Anchors (`&`/`*`) and merge keys (`<<:`) supported for DRY templates. |
| Markdown + YAML frontmatter | — | Skill instruction format | SKILL.md is Markdown with YAML frontmatter — no code, no runtime, no dependencies. |

### Supporting Libraries

None required. The skill is purely static file generation — instructions in Markdown + embedded YAML templates written by the AI. No package manager, no runtime.

### Development Tools

| Tool | Purpose | Notes |
|------|---------|-------|
| GitHub Copilot CLI | Primary skill host | Reads `.github/skills/<name>/SKILL.md` and `~/.copilot/skills/` |
| Claude Desktop / Claude Code | Secondary skill host | Reads `.claude/skills/<name>/SKILL.md` and `~/.claude/skills/` |
| `task` CLI (taskfile.dev) | Run generated Taskfiles | v3.49.1 — installed in target project's dev environment |
| `docker compose` (v2 plugin) | Run generated Compose files | Bundled with Docker Desktop; standalone `docker compose` (space, not hyphen) |

---

## Skill Format Specification

### SKILL.md Structure (GitHub Copilot CLI + Claude)

**This is the critical finding**: GitHub Copilot CLI and Claude use the **identical `SKILL.md` format**. One file works for both runtimes.

```
.github/skills/
  dev-tools/
    SKILL.md         ← The skill definition
    redis.yml        ← Companion template (optional)
    postgres.yml     ← Companion template (optional)
```

**SKILL.md frontmatter fields:**

```yaml
---
name: dev-tools                          # required: lowercase, hyphens for spaces
description: >                           # required: what the skill does + when to use it
  Adds Docker Compose service definitions and Taskfile tasks to a project.
  Use when asked to add Redis, PostgreSQL, RabbitMQ, MongoDB, or MySQL.
license: MIT                             # optional
allowed-tools: write_file, create_directory  # optional: pre-approve tool use
---

Markdown body with instructions for the AI...
```

**Skill discovery locations** (both runtimes scan these):

| Scope | GitHub Copilot CLI | Claude |
|-------|-------------------|--------|
| Project | `.github/skills/<name>/SKILL.md` | `.claude/skills/<name>/SKILL.md` |
| Project (generic) | `.agents/skills/<name>/SKILL.md` | `.agents/skills/<name>/SKILL.md` |
| Personal | `~/.copilot/skills/<name>/SKILL.md` | `~/.claude/skills/<name>/SKILL.md` |

**Portability strategy**: Ship skills in ALL three project locations (`.github/skills/`, `.claude/skills/`, `.agents/skills/`) with identical content, or use symlinks. This ensures zero-config pickup by any runtime.

**Invoking a skill** by name: `Use the /dev-tools skill to add Redis to this project`

### MCP Tool Format (Claude — alternative to SKILL.md)

If a richer programmatic interface is ever needed (e.g., validation, dynamic queries), MCP tools are the next layer. Current MCP spec: `2025-06-18`.

```json
{
  "name": "add_dev_service",
  "title": "Add Dev Service",
  "description": "Adds a Docker Compose service and Taskfile tasks to .devtools/",
  "inputSchema": {
    "type": "object",
    "properties": {
      "service": { "type": "string", "enum": ["redis", "postgres", "rabbitmq", "mysql", "mongodb"] },
      "port": { "type": "integer" },
      "version": { "type": "string" }
    },
    "required": ["service"]
  }
}
```

MCP servers require a running process (Python `mcp` SDK, TypeScript SDK, or Go). **Not recommended for this project** — the SKILL.md approach satisfies all requirements with zero dependencies. MCP is overkill unless interactive validation is needed in a future phase.

---

## Docker Compose v2 Syntax

**Verified source:** `https://raw.githubusercontent.com/compose-spec/compose-spec/master/spec.md` (authoritative)

### Key Rules

1. **No `version:` key** — the `version:` top-level field is obsolete. Using it generates a deprecation warning. Omit entirely.
2. **File name**: `compose.yaml` is preferred. `compose.yml`, `docker-compose.yaml`, `docker-compose.yml` are supported for backwards compatibility.
3. **Merge/include**: Use `include:` directive to split across multiple files.
4. **YAML anchors**: Supported for DRY service definitions.

### Canonical Service Template (Redis example)

```yaml
# .devtools/compose.yaml
services:
  redis:
    image: redis:7-alpine
    container_name: ${COMPOSE_PROJECT_NAME:-app}-redis
    ports:
      - "${REDIS_PORT:-6379}:6379"
    volumes:
      - redis-data:/data
    restart: unless-stopped
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 10s
      timeout: 5s
      retries: 3

volumes:
  redis-data:
```

### Key Service Properties (for generated configs)

| Property | Use | Notes |
|----------|-----|-------|
| `image` | Container image | Always pin major version (e.g., `redis:7-alpine`) |
| `container_name` | Named container | Use `${COMPOSE_PROJECT_NAME:-default}-<service>` pattern |
| `ports` | Port mapping | Format: `"host:container"` as string |
| `volumes` | Persistent data | Named volumes declared at top level |
| `environment` | Env vars | Use `KEY: value` map format, not `- KEY=value` list |
| `restart` | Restart policy | `unless-stopped` for local dev services |
| `healthcheck` | Health probes | Optional but recommended |
| `networks` | Network isolation | Omit for simple single-file configs |

### YAML Anchor Pattern for Multiple Services

```yaml
x-common-service: &common-service
  restart: unless-stopped
  networks:
    - devtools

services:
  redis:
    <<: *common-service
    image: redis:7-alpine
    ports:
      - "6379:6379"

  postgres:
    <<: *common-service
    image: postgres:16-alpine
    environment:
      POSTGRES_PASSWORD: devpassword

networks:
  devtools:
```

**Extension fields** (`x-*` prefix) are the correct way to define reusable anchors without creating actual service/volume entries.

---

## Taskfile v3 Schema

**Verified source:** `https://taskfile.dev/schema.json` + `https://taskfile.dev/docs/reference/schema.md`

**Current version**: v3.49.1 (April 2026)

### Root Schema Properties

```yaml
version: '3'          # required; string or number; '3' covers all v3.x
output: interleaved   # interleaved | group | prefixed
silent: false
dotenv: ['.env']      # load .env files
env:                  # global env vars
  KEY: value
vars:                 # global variables
  SERVICE: redis
includes:             # import other Taskfiles
  redis: .devtools/redis.yml
tasks:
  default:
    cmds:
      - task --list
```

### Include Pattern for Per-Service Taskfiles

```yaml
# .devtools/Taskfile.yml
version: '3'

includes:
  redis:
    taskfile: ./redis.yml
    optional: true
  postgres:
    taskfile: ./postgres.yml
    optional: true
  rabbitmq:
    taskfile: ./rabbitmq.yml
    optional: true
```

With `optional: true`, missing service files don't cause errors — safe for incremental additions.

Tasks from `redis.yml` are invoked as `task redis:up`, `task redis:down`, etc.

### Per-Service Taskfile Template

```yaml
# .devtools/redis.yml
version: '3'

tasks:
  up:
    desc: Start Redis
    cmds:
      - docker compose -f {{.TASKFILE_DIR}}/compose.yaml up -d redis

  down:
    desc: Stop Redis
    cmds:
      - docker compose -f {{.TASKFILE_DIR}}/compose.yaml stop redis

  logs:
    desc: Tail Redis logs
    cmds:
      - docker compose -f {{.TASKFILE_DIR}}/compose.yaml logs -f redis

  restart:
    desc: Restart Redis
    cmds:
      - task: down
      - task: up

  cli:
    desc: Open Redis CLI
    cmds:
      - docker compose -f {{.TASKFILE_DIR}}/compose.yaml exec redis redis-cli
```

**`{{.TASKFILE_DIR}}`** — special variable that resolves to the directory of the current Taskfile. Essential for making task paths work regardless of where `task` is invoked from.

### Key Task Properties

| Property | Type | Purpose |
|----------|------|---------|
| `desc` | string | Short description shown in `task --list` |
| `cmds` | []Command | Commands to run |
| `deps` | []Task | Run before this task |
| `dir` | string | Working directory |
| `env` | map | Task-scoped env vars |
| `vars` | map | Task-scoped variables |
| `preconditions` | [] | Fail fast checks before running |
| `aliases` | []string | Alternative names (`task r:up` → `task redis:up`) |
| `silent` | bool | Suppress command echo |
| `interactive` | bool | Required for commands needing stdin |

---

## What NOT to Use

| Avoid | Why | Use Instead |
|-------|-----|-------------|
| `version:` in compose.yaml | Officially obsolete as of Compose Spec 2024+. Generates deprecation warnings. No benefit. | Omit entirely |
| `docker-compose` (v1, hyphen) | Python binary, deprecated since 2023, removed from Docker Desktop 2024+. Breaks on modern systems. | `docker compose` (space, v2 plugin) |
| MCP server for this project | Requires running process, language runtime, package install. Violates "no dependencies" constraint. | `SKILL.md` format (static, zero deps) |
| GitHub Copilot Skillset API (old) | Old REST-based skillset extension format (GitHub App + HTTP endpoints). Complex, server required, not compatible with CLI skill format. | `SKILL.md` agent skills format |
| Makefile | Shell-only, poor Windows support, cryptic syntax, no built-in help. Explicitly out of scope per PROJECT.md. | Taskfile v3 |
| `version: 2` Taskfile | Deprecated schema, no longer officially supported. Version 2 behavior differs from v3. | `version: '3'` |

---

## Stack Patterns by Variant

**If deploying the skill project-scoped (recommended for this repo):**
- Ship `SKILL.md` in `.github/skills/dev-tools/`, `.claude/skills/dev-tools/`, AND `.agents/skills/dev-tools/` (symlinks or identical copies)
- Covers GitHub Copilot CLI, Claude Code, and any future ACP-compatible runtime
- No configuration needed by the end user

**If the project needs Claude MCP (future v2):**
- Use Python `mcp` SDK (FastMCP) or TypeScript `@modelcontextprotocol/sdk`
- Define tools with JSON Schema inputSchema
- Register in `~/.config/claude/claude_desktop_config.json`

**If the generated `compose.yaml` needs to be mergeable into an existing one:**
- Write each service to a separate `compose.<service>.yaml`
- Use `include:` directive in the root `compose.yaml`
- Let users choose to merge or include

**If the generated Taskfile must integrate with an existing project Taskfile:**
- Add `includes: devtools: .devtools/Taskfile.yml` to the project root `Taskfile.yml`
- All devtools tasks are namespaced: `task devtools:redis:up`

---

## Version Compatibility

| Package | Compatible With | Notes |
|---------|-----------------|-------|
| Taskfile v3 (any v3.x) | Docker Compose v2 | No coupling; independent tools |
| `SKILL.md` format | GitHub Copilot CLI current, Claude Code current | Both scan same paths as of early 2026 |
| Compose Spec (no version field) | Docker Engine 20.10+, Docker Desktop 4.x+ | Older Docker Engine may not support all spec features |
| Taskfile `includes` + `optional: true` | Taskfile v3.17+ | `optional` on includes added in v3.17; specify `version: '3.17'` to enforce |

---

## Alternatives Considered

| Recommended | Alternative | When to Use Alternative |
|-------------|-------------|-------------------------|
| `SKILL.md` agent skills | MCP server tool | When the skill needs programmatic validation, external API calls, or stateful behavior. Over-engineered for static file generation. |
| `SKILL.md` in all three dirs | Single `.github/skills/` only | If GitHub Copilot CLI is the only supported runtime and Claude support is explicitly not required. |
| `compose.yaml` (single file, `include:`) | One file per service | When each service file needs to be standalone and independent. Adds complexity to compose invocations. |
| Taskfile `includes` (per-service files) | Monolithic `Taskfile.yml` | When all services are always installed together and modularity is not needed. |
| No `version:` in compose | `version: "3.9"` | Never. The `version` field is obsolete and the number is meaningless in Compose v2. |

---

## Installation

No package installation required by the skill runtime. The skill writes static files.

Target project dependencies (what users need installed):

```bash
# Docker Compose v2 (comes with Docker Desktop, or standalone)
docker compose version  # should show v2.x

# Taskfile (https://taskfile.dev/docs/installation)
# macOS
brew install go-task

# Linux
sh -c "$(curl --location https://taskfile.dev/install.sh)" -- -d -b /usr/local/bin

# Windows (scoop)
scoop install task
```

---

## Sources

- `https://docs.github.com/en/copilot/how-tos/copilot-cli/customize-copilot/create-skills` — SKILL.md format, frontmatter fields, skill locations (HIGH confidence, official GitHub Docs, verified 2026-04-10)
- `https://raw.githubusercontent.com/compose-spec/compose-spec/master/spec.md` — Compose v2 spec, `version:` obsolescence, file naming (HIGH confidence, authoritative spec source)
- `https://raw.githubusercontent.com/compose-spec/compose-spec/master/10-fragments.md` — YAML anchors and merge keys in Compose (HIGH confidence)
- `https://taskfile.dev/docs/reference/schema.md` — Full Taskfile v3 schema reference (HIGH confidence, official docs)
- `https://taskfile.dev/docs/guide.md` — `includes`, env, variables usage (HIGH confidence, official docs)
- `https://taskfile.dev/schema.json` — Machine-readable JSON schema (HIGH confidence, live schema)
- `https://modelcontextprotocol.io/docs/concepts/tools.md` — MCP tool format, JSON-RPC 2.0 (HIGH confidence, official MCP docs, spec version 2025-06-18)
- `https://modelcontextprotocol.io/specification/2025-06-18` — MCP current spec version (HIGH confidence)

---

*Stack research for: AI skill toolkit — Docker Compose + Taskfile generator*
*Researched: 2026-04-10*
