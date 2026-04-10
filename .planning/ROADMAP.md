# Roadmap: dev-tools

## Overview

Three phases build this project in the only order that makes sense: templates first (the content that everything else references), then the skill that orchestrates them (the interactive flow and merge logic), then the wiring that makes it discoverable in both AI runtimes plus the documentation that lets others install it. Each phase delivers a coherent, independently verifiable capability before the next begins.

## Phases

**Phase Numbering:**
- Integer phases (1, 2, 3): Planned milestone work
- Decimal phases (2.1, 2.2): Urgent insertions (marked with INSERTED)

Decimal phases appear between their surrounding integers in numeric order.

- [x] **Phase 1: Templates & Credential Foundation** - All 5 service templates (Compose + Taskfile) authored with correct credential patterns (completed 2026-04-10)
- [ ] **Phase 2: Skill Core — Interactive Flow & Merge Logic** - SKILL.md that guides an AI through interactive config, credential capture, and safe idempotent writes
- [ ] **Phase 3: Cross-Runtime Wiring & Distribution** - Skill discoverable by both Copilot CLI and Claude/MCP; README documents installation

## Phase Details

### Phase 1: Templates & Credential Foundation
**Goal**: All service templates exist on disk in a locked file layout, with credentials exclusively in `.env` patterns — establishing the foundation that SKILL.md will reference
**Depends on**: Nothing (first phase)
**Requirements**: TMPL-01, TMPL-02, TMPL-03, TMPL-04, TMPL-05, TMPL-06, TMPL-07, TMPL-08, CRED-01, CRED-02, CRED-03
**Success Criteria** (what must be TRUE):
  1. Template files exist for all 5 services (Redis, RabbitMQ, PostgreSQL, MySQL, MongoDB) as Compose fragments and per-service Taskfiles under `templates/`
  2. No Compose template contains a `version:` key — verified by inspection
  3. All credentials in Compose templates reference env vars (e.g. `${POSTGRES_PASSWORD}`) — no hardcoded passwords anywhere
  4. Each per-service Taskfile has `up`, `down`, `logs`, `restart` tasks; all Docker Compose `-f` flags use `{{.TASKFILE_DIR}}` for path resolution
  5. A root Taskfile template includes all service Taskfiles via `includes:` with `optional: true`
**Plans**: 3 plans

Plans:
- [x] 01-01-PLAN.md — Compose service fragments (6 services: Redis, RabbitMQ, PostgreSQL, MySQL/MariaDB, MongoDB, Monitoring) + metadata.json files
- [x] 01-02-PLAN.md — Per-service Taskfile templates (6 services) + root Taskfile with optional includes
- [x] 01-03-PLAN.md — .env.example credential template + credential compliance sweep

### Phase 2: Skill Core — Interactive Flow & Merge Logic
**Goal**: A working `SKILL.md` that guides an AI through port/version/credential questions and writes files safely — never overwriting existing services, fully idempotent on re-run
**Depends on**: Phase 1
**Requirements**: CONF-01, CONF-02, CONF-03, MERGE-01, MERGE-02, MERGE-03, MERGE-04
**Success Criteria** (what must be TRUE):
  1. An AI invoking the skill asks for port, image version, and credentials before writing any files — nothing is written until config is confirmed
  2. RabbitMQ configuration includes a prompt for the Management UI port (15672) — with a sensible accept/skip default
  3. Running the skill for a service that already exists asks the user to add another instance or cancel — no files are silently overwritten
  4. Adding a second instance with an alias (e.g. `redis-cache`) produces namespaced files that don't disturb the original service's files
  5. Re-running the skill with an identical existing alias produces no changes and informs the user it already exists
**Plans**: TBD

Plans:
- [ ] 02-01: SKILL.md frontmatter and service registry (5 services with defaults)
- [ ] 02-02: Interactive Q&A flow (port, version, credentials, RabbitMQ UI prompt)
- [ ] 02-03: Merge detection, alias handling, and idempotency logic

### Phase 3: Cross-Runtime Wiring & Distribution
**Goal**: The skill is installed in both AI runtime discovery paths and the repository has documentation enabling any developer to adopt it
**Depends on**: Phase 2
**Requirements**: SKILL-01, SKILL-02, SKILL-03, DIST-01, DIST-02
**Success Criteria** (what must be TRUE):
  1. A developer can trigger the skill in GitHub Copilot CLI by saying "add Redis to this project" — the correct skill activates
  2. The canonical `SKILL.md` is present (or symlinked) in both `.github/skills/add-service/` and `.agents/skills/add-service/`
  3. The skill frontmatter explicitly enumerates all 5 supported services so AI runtimes reliably match it for service-addition intents
  4. A developer reading `README.md` understands how to install the skill into any project and which services are supported
**Plans**: TBD

Plans:
- [ ] 03-01: Skill installation to both runtime discovery paths (Copilot CLI + Claude/MCP)
- [ ] 03-02: Project README (installation guide + supported services)

## Progress

**Execution Order:**
Phases execute in numeric order: 1 → 2 → 3

| Phase | Plans Complete | Status | Completed |
|-------|----------------|--------|-----------|
| 1. Templates & Credential Foundation | 3/3 | Complete   | 2026-04-10 |
| 2. Skill Core — Interactive Flow & Merge Logic | 0/3 | Not started | - |
| 3. Cross-Runtime Wiring & Distribution | 0/2 | Not started | - |
