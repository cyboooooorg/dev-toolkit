<!-- GSD:project-start source:PROJECT.md -->
## Project

**dev-tools**

A development toolkit that works as an AI skill — triggered by natural language like "add Redis to this project." It interactively asks configuration questions (port, version, credentials) and then writes Docker Compose and Taskfile configurations into a `.devtools/` directory in the target project. Designed to work with both GitHub Copilot CLI and Claude/MCP-compatible AI runtimes.

**Core Value:** An AI can drop production-ready Docker service configs and Taskfiles into any project in one conversation, with zero manual file writing.

### Constraints

- **Compatibility**: Must work as a skill in both GitHub Copilot CLI and Claude/MCP — no runtime-specific code in templates
- **No dependencies**: The skill itself must not require npm, pip, or any package manager to run — it writes static files
- **Docker Compose v2**: Use current syntax, no deprecated `version:` field
- **Taskfile**: Use Taskfile v3 schema (taskfile.dev)
<!-- GSD:project-end -->

<!-- GSD:stack-start source:research/STACK.md -->
## Technology Stack

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
### Development Tools
| Tool | Purpose | Notes |
|------|---------|-------|
| GitHub Copilot CLI | Primary skill host | Reads `.github/skills/<name>/SKILL.md` and `~/.copilot/skills/` |
| Claude Desktop / Claude Code | Secondary skill host | Reads `.claude/skills/<name>/SKILL.md` and `~/.claude/skills/` |
| `task` CLI (taskfile.dev) | Run generated Taskfiles | v3.49.1 — installed in target project's dev environment |
| `docker compose` (v2 plugin) | Run generated Compose files | Bundled with Docker Desktop; standalone `docker compose` (space, not hyphen) |
## Skill Format Specification
### SKILL.md Structure (GitHub Copilot CLI + Claude)
| Scope | GitHub Copilot CLI | Claude |
|-------|-------------------|--------|
| Project | `.github/skills/<name>/SKILL.md` | `.claude/skills/<name>/SKILL.md` |
| Project (generic) | `.agents/skills/<name>/SKILL.md` | `.agents/skills/<name>/SKILL.md` |
| Personal | `~/.copilot/skills/<name>/SKILL.md` | `~/.claude/skills/<name>/SKILL.md` |
### MCP Tool Format (Claude — alternative to SKILL.md)
## Docker Compose v2 Syntax
### Key Rules
### Canonical Service Template (Redis example)
# .devtools/compose.yaml
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
## Taskfile v3 Schema
### Root Schema Properties
### Include Pattern for Per-Service Taskfiles
# .devtools/Taskfile.yml
### Per-Service Taskfile Template
# .devtools/redis.yml
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
## What NOT to Use
| Avoid | Why | Use Instead |
|-------|-----|-------------|
| `version:` in compose.yaml | Officially obsolete as of Compose Spec 2024+. Generates deprecation warnings. No benefit. | Omit entirely |
| `docker-compose` (v1, hyphen) | Python binary, deprecated since 2023, removed from Docker Desktop 2024+. Breaks on modern systems. | `docker compose` (space, v2 plugin) |
| MCP server for this project | Requires running process, language runtime, package install. Violates "no dependencies" constraint. | `SKILL.md` format (static, zero deps) |
| GitHub Copilot Skillset API (old) | Old REST-based skillset extension format (GitHub App + HTTP endpoints). Complex, server required, not compatible with CLI skill format. | `SKILL.md` agent skills format |
| Makefile | Shell-only, poor Windows support, cryptic syntax, no built-in help. Explicitly out of scope per PROJECT.md. | Taskfile v3 |
| `version: 2` Taskfile | Deprecated schema, no longer officially supported. Version 2 behavior differs from v3. | `version: '3'` |
## Stack Patterns by Variant
- Ship `SKILL.md` in `.github/skills/dev-tools/`, `.claude/skills/dev-tools/`, AND `.agents/skills/dev-tools/` (symlinks or identical copies)
- Covers GitHub Copilot CLI, Claude Code, and any future ACP-compatible runtime
- No configuration needed by the end user
- Use Python `mcp` SDK (FastMCP) or TypeScript `@modelcontextprotocol/sdk`
- Define tools with JSON Schema inputSchema
- Register in `~/.config/claude/claude_desktop_config.json`
- Write each service to a separate `compose.<service>.yaml`
- Use `include:` directive in the root `compose.yaml`
- Let users choose to merge or include
- Add `includes: devtools: .devtools/Taskfile.yml` to the project root `Taskfile.yml`
- All devtools tasks are namespaced: `task devtools:redis:up`
## Version Compatibility
| Package | Compatible With | Notes |
|---------|-----------------|-------|
| Taskfile v3 (any v3.x) | Docker Compose v2 | No coupling; independent tools |
| `SKILL.md` format | GitHub Copilot CLI current, Claude Code current | Both scan same paths as of early 2026 |
| Compose Spec (no version field) | Docker Engine 20.10+, Docker Desktop 4.x+ | Older Docker Engine may not support all spec features |
| Taskfile `includes` + `optional: true` | Taskfile v3.17+ | `optional` on includes added in v3.17; specify `version: '3.17'` to enforce |
## Alternatives Considered
| Recommended | Alternative | When to Use Alternative |
|-------------|-------------|-------------------------|
| `SKILL.md` agent skills | MCP server tool | When the skill needs programmatic validation, external API calls, or stateful behavior. Over-engineered for static file generation. |
| `SKILL.md` in all three dirs | Single `.github/skills/` only | If GitHub Copilot CLI is the only supported runtime and Claude support is explicitly not required. |
| `compose.yaml` (single file, `include:`) | One file per service | When each service file needs to be standalone and independent. Adds complexity to compose invocations. |
| Taskfile `includes` (per-service files) | Monolithic `Taskfile.yml` | When all services are always installed together and modularity is not needed. |
| No `version:` in compose | `version: "3.9"` | Never. The `version` field is obsolete and the number is meaningless in Compose v2. |
## Installation
# Docker Compose v2 (comes with Docker Desktop, or standalone)
# Taskfile (https://taskfile.dev/docs/installation)
# macOS
# Linux
# Windows (scoop)
## Sources
- `https://docs.github.com/en/copilot/how-tos/copilot-cli/customize-copilot/create-skills` — SKILL.md format, frontmatter fields, skill locations (HIGH confidence, official GitHub Docs, verified 2026-04-10)
- `https://raw.githubusercontent.com/compose-spec/compose-spec/master/spec.md` — Compose v2 spec, `version:` obsolescence, file naming (HIGH confidence, authoritative spec source)
- `https://raw.githubusercontent.com/compose-spec/compose-spec/master/10-fragments.md` — YAML anchors and merge keys in Compose (HIGH confidence)
- `https://taskfile.dev/docs/reference/schema.md` — Full Taskfile v3 schema reference (HIGH confidence, official docs)
- `https://taskfile.dev/docs/guide.md` — `includes`, env, variables usage (HIGH confidence, official docs)
- `https://taskfile.dev/schema.json` — Machine-readable JSON schema (HIGH confidence, live schema)
- `https://modelcontextprotocol.io/docs/concepts/tools.md` — MCP tool format, JSON-RPC 2.0 (HIGH confidence, official MCP docs, spec version 2025-06-18)
- `https://modelcontextprotocol.io/specification/2025-06-18` — MCP current spec version (HIGH confidence)
<!-- GSD:stack-end -->

<!-- GSD:conventions-start source:CONVENTIONS.md -->
## Conventions

Conventions not yet established. Will populate as patterns emerge during development.
<!-- GSD:conventions-end -->

<!-- GSD:architecture-start source:ARCHITECTURE.md -->
## Architecture

Architecture not yet mapped. Follow existing patterns found in the codebase.
<!-- GSD:architecture-end -->

<!-- GSD:skills-start source:skills/ -->
## Project Skills

No project skills found. Add skills to any of: `.github/skills/`, `.agents/skills/`, `.cursor/skills/`, or `.github/skills/` with a `SKILL.md` index file.
<!-- GSD:skills-end -->

<!-- GSD:workflow-start source:GSD defaults -->
## GSD Workflow Enforcement

Before using Edit, Write, or other file-changing tools, start work through a GSD command so planning artifacts and execution context stay in sync.

Use these entry points:
- `/gsd-quick` for small fixes, doc updates, and ad-hoc tasks
- `/gsd-debug` for investigation and bug fixing
- `/gsd-execute-phase` for planned phase work

Do not make direct repo edits outside a GSD workflow unless the user explicitly asks to bypass it.
<!-- GSD:workflow-end -->



<!-- GSD:profile-start -->
## Developer Profile

> Profile not yet configured. Run `/gsd-profile-user` to generate your developer profile.
> This section is managed by `generate-claude-profile` -- do not edit manually.
<!-- GSD:profile-end -->
