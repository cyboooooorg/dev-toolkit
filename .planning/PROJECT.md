# dev-tools

## Current Milestone: v1.1 Port Management

**Goal:** Give users control over host port mapping and prevent port collisions when installing services.

**Target features:**
- Optional host port binding — skill asks if the user wants a host port mapped (default: no, service internal-only)
- Port collision detection — when host port binding is requested, scan all already-installed services (including UI and monitoring ports) for conflicts
- Port re-prompt loop — if chosen port is taken, ask again until a free port is selected or user opts out

---

## What This Is

A development toolkit that works as an AI skill — triggered by natural language like "add Redis to this project." It interactively asks configuration questions (port, version, credentials) and then writes Docker Compose and Taskfile configurations into a `.devtools/` directory in the target project. Designed to work with both GitHub Copilot CLI and Claude/MCP-compatible AI runtimes.

## Core Value

An AI can drop production-ready Docker service configs and Taskfiles into any project in one conversation, with zero manual file writing.

## Requirements

### Validated ✓ v1.0

- ✓ Docker Compose service definitions for Redis, RabbitMQ, PostgreSQL, MySQL, MongoDB — v1.0
- ✓ Taskfile tasks per service: `up`, `up-ui`, `up-monitoring`, `down`, `logs`, `restart` — v1.0
- ✓ Template library with `{{PLACEHOLDER}}` token format for credential substitution — v1.0
- ✓ No hardcoded secrets in any template; all credentials via `.env` — v1.0
- ✓ AI skill definition that triggers on natural language like "add Redis to this project" — v1.0
- ✓ Interactive configuration flow: ask port, version, credentials before writing files — v1.0
- ✓ Writes a `.devtools/<service>/` directory into the target project with Docker Compose + Taskfile — v1.0
- ✓ Per-service Taskfiles included from a root `Taskfile.yml` via `includes:` — v1.0
- ✓ Compatible with both GitHub Copilot CLI skill format and Claude/MCP skill format — v1.0
- ✓ Merge detection: if `.devtools/` exists, offer alias or cancel — v1.0
- ✓ Multi-instance alias namespacing (service, volumes, env vars, Taskfile) — v1.0
- ✓ Idempotent re-install: alias slug re-run exits cleanly with "already installed" — v1.0

### Validated ✓ v1.1

- ✓ Optional host port binding — ask user before mapping any service port to host (PORT-01, PORT-02) — v1.1
- ✓ Port collision detection — scan installed services for conflicting host ports before writing (PORT-03, PORT-04) — v1.1
- ✓ Port re-prompt loop — if chosen port is taken, ask for a different one with 3-option escape menu (PORT-05) — v1.1

### Active (v2.0 targets)

- [ ] Auto-detect existing stack (package.json, requirements.txt, etc.) and suggest relevant services
- [ ] Service-specific CLI tasks: `redis:cli`, `mongo:shell`, `psql`
- [ ] Version pinning recommendations (warn if `latest` tag used)
- [ ] Kafka / Elasticsearch service templates
- [ ] Health-check wait task (poll until service is ready)
- [ ] Fix BF-01: Monitoring Grafana password not applied (add `env_file: .env` to grafana service)
- [ ] Fix DOC-01: README "What Gets Written" shows stale pre-Phase-4 flat layout
- [ ] Fix WR-01/WR-02: Taskfile COMPOSE_PROFILES not loaded from `.devtools/.env`

### Out of Scope

- Kubernetes / Helm chart generation — different problem, different tool
- Cloud provisioning (AWS, GCP, Azure) — this is local dev only
- Language-specific scaffolding (app code, frameworks) — only infrastructure config
- MCP server implementation — overkill; SKILL.md format is sufficient
- Automatic port conflict detection — violates zero-dependency constraint

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
| `.devtools/` as output directory | Isolated from project source, easy to gitignore or include selectively | ✓ Confirmed Phase 01 |
| Per-service Taskfiles included from root | Modular — adding a service doesn't require editing existing files | ✓ Confirmed Phase 01 |
| `{{PLACEHOLDER}}` token format | Simple, unambiguous substitution; no shell expansion collisions | ✓ Confirmed Phase 01 |
| Three-tier Docker networks (per-service / _services / _monitoring) | Isolation + cross-service reach without exposing internals | ✓ Confirmed Phase 01 |
| Plain volume key names (no `${VAR}` in keys) | Docker Compose v2 does not support variable expansion in volume key names | ✓ Confirmed Phase 01 |
| Interactive config before writing | Avoids wrong defaults baked in; user owns their config from the start | ✓ Confirmed Phase 02 |
| Support both Copilot CLI and Claude/MCP | Maximize portability across AI tools | ✓ Confirmed Phase 03 |
| Subfolder per-service layout (`.devtools/<slug>/`) | Supports multi-instance aliases cleanly; each instance has its own `.env` | ✓ Confirmed Phase 04 |
| Proactive alias prompt on first install | Reduces re-run friction; user sets instance name upfront | ✓ Confirmed Phase 05 |
| 6 base service names as idempotency guard | Distinguishes "base service re-install" (→ Step 1a) from "alias slug re-run" (→ early exit) | ✓ Confirmed Phase 07 |

## Current State

**v1.1 shipped 2026-04-23** — Port management milestone complete. All 5 PORT requirements (PORT-01–05) satisfied. Services now have opt-in host port binding with collision detection and a 3-option escape menu. v1.0 tech debt (BF-01, DOC-01, WR-01/WR-02) carried forward to v2.0.

*Last updated: 2026-04-23 — v1.1 milestone archived*

<details>
<summary>Previous milestone: v1.1 Port Management (archived)</summary>

**v1.1 started 2026-04-21** — Port management milestone in requirements/roadmap phase. v1.0 shipped with all 23 requirements satisfied; tech debt BF-01 (Grafana env_file) and DOC-01 (README layout) carried forward.

*Last updated: 2026-04-21 — v1.1 milestone started*

</details>

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

*Last updated: 2026-04-21 — v1.1 milestone started*
