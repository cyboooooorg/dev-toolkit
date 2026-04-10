# dev-tools

## What This Is

A development toolkit that works as an AI skill — triggered by natural language like "add Redis to this project." It interactively asks configuration questions (port, version, credentials) and then writes Docker Compose and Taskfile configurations into a `.devtools/` directory in the target project. Designed to work with both GitHub Copilot CLI and Claude/MCP-compatible AI runtimes.

## Core Value

An AI can drop production-ready Docker service configs and Taskfiles into any project in one conversation, with zero manual file writing.

## Requirements

### Validated

- [x] Docker Compose service definitions for Redis, RabbitMQ, PostgreSQL, MySQL, MongoDB — Validated in Phase 01: Templates & Credential Foundation
- [x] Taskfile tasks per service: `up`, `up-ui`, `up-monitoring`, `down`, `logs`, `restart` — Validated in Phase 01: Templates & Credential Foundation
- [x] Template library with `{{PLACEHOLDER}}` token format for credential substitution — Validated in Phase 01: Templates & Credential Foundation
- [x] No hardcoded secrets in any template; all credentials via `.env` — Validated in Phase 01: Templates & Credential Foundation

### Active

- [ ] AI skill definition that triggers on natural language like "add Redis to this project"
- [ ] Interactive configuration flow: ask port, version, credentials before writing files
- [ ] Writes a `.devtools/` directory into the target project with Docker Compose + Taskfile
- [ ] Per-service Taskfiles (e.g. `redis.yml`, `rabbitmq.yml`) included from a root `Taskfile.yml`
- [ ] Compatible with both GitHub Copilot CLI skill format and Claude/MCP skill format
- [ ] If a `.devtools/` directory already exists, merge new service without overwriting existing ones
- [ ] Skill is self-contained and installable into any project without dependencies

### Out of Scope

- Kubernetes / Helm chart generation — different problem, different tool
- Cloud provisioning (AWS, GCP, Azure) — this is local dev only
- Language-specific scaffolding (app code, frameworks) — only infrastructure config
- Service health checks or monitoring dashboards — out of scope for v1

## Context

- The project lives in a GitHub org repo (`Cyboooooorg/dev-tools`)
- Skills need to work in two AI runtime formats: GitHub Copilot CLI (`.github/skills/`) and Claude/MCP
- Taskfile is the task runner of choice (https://taskfile.dev) — not Makefile, not npm scripts
- Docker Compose v2 syntax (services block, no `version:` key)
- Services must use sensible defaults for local dev (e.g., no auth required for Redis by default, but configurable)

## Constraints

- **Compatibility**: Must work as a skill in both GitHub Copilot CLI and Claude/MCP — no runtime-specific code in templates
- **No dependencies**: The skill itself must not require npm, pip, or any package manager to run — it writes static files
- **Docker Compose v2**: Use current syntax, no deprecated `version:` field
- **Taskfile**: Use Taskfile v3 schema (taskfile.dev)

## Key Decisions

| Decision | Rationale | Outcome |
|----------|-----------|---------|
| `.devtools/` as output directory | Isolated from project source, easy to gitignore or include selectively | Confirmed Phase 01 |
| Per-service Taskfiles included from root | Modular — adding a service doesn't require editing existing files | Confirmed Phase 01 |
| `{{PLACEHOLDER}}` token format | Simple, unambiguous substitution; no shell expansion collisions | Confirmed Phase 01 |
| Three-tier Docker networks (per-service / _services / _monitoring) | Isolation + cross-service reach without exposing internals | Confirmed Phase 01 |
| Plain volume key names (no `${VAR}` in keys) | Docker Compose v2 does not support variable expansion in volume key names | Confirmed Phase 01 |
| Interactive config before writing | Avoids wrong defaults baked in; user owns their config from the start | — Pending Phase 02 |
| Support both Copilot CLI and Claude/MCP | Maximize portability across AI tools | — Pending Phase 02 |

## Current State

**Phase 02 complete** — Core `skills/add-service/SKILL.md` written (542 lines, 14 steps). Interactive AI skill guides configuration Q&A, shows a confirmation gate, then writes Docker Compose and Taskfile configs to `.devtools/`. Merge detection, alias multi-instance, and alias idempotency all implemented. Human UAT pending (3 runtime tests). Ready for Phase 03: cross-runtime wiring & distribution.

## Evolution

This document evolves at phase transitions and milestone boundaries.

**After each phase transition** (via `/gsd-transition`):
1. Requirements invalidated? → Move to Out of Scope with reason
2. Requirements validated? → Move to Validated with phase reference
3. New requirements emerged? → Add to Active
4. Decisions to log? → Add to Key Decisions
5. "What This Is" still accurate? → Update if drifted

**After each milestone** (via `/gsd-complete-milestone`):
1. Full review of all sections
2. Core Value check — still the right priority?
3. Audit Out of Scope — reasons still valid?
4. Update Context with current state

---
*Last updated: 2026-04-10 — Phase 02 complete*
