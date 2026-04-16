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
- [x] **Phase 3: Cross-Runtime Wiring & Distribution** - Skill discoverable by both Copilot CLI and Claude/MCP; README documents installation (completed 2026-04-14)

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
**Plans**: 3 plans

Plans:
- [x] 02-01-PLAN.md — SKILL.md frontmatter, service registry, and process Steps 1–2 (merge detection + first-run setup)
- [x] 02-02-PLAN.md — Interactive Q&A flow: service selection, per-service questions, UI companion, optional prompts, confirmation gate (Steps 3–8)
- [x] 02-03-PLAN.md — File writing, .env management, root compose/Taskfile management, alias substitution, done summary (Steps 9–13)

### Phase 3: Cross-Runtime Wiring & Distribution
**Goal**: The skill is installed in both AI runtime discovery paths and the repository has documentation enabling any developer to adopt it
**Depends on**: Phase 2
**Requirements**: SKILL-01, SKILL-02, SKILL-03, DIST-01, DIST-02
**Success Criteria** (what must be TRUE):
  1. A developer can trigger the skill in GitHub Copilot CLI by saying "add Redis to this project" — the correct skill activates
  2. The canonical `SKILL.md` is present (or symlinked) in both `.github/skills/add-service/` and `.agents/skills/add-service/`
  3. The skill frontmatter explicitly enumerates all 5 supported services so AI runtimes reliably match it for service-addition intents
  4. A developer reading `README.md` understands how to install the skill into any project and which services are supported
**Plans**: 2 plans

Plans:
- [x] 03-01-PLAN.md — Verify SKILL.md frontmatter is valid for skills.sh discovery (SKILL-01, SKILL-02, SKILL-03, DIST-01)
- [x] 03-02-PLAN.md — Write root README.md with install command, services table, and .devtools/ output description (DIST-02)

## Progress

**Execution Order:**
Phases execute in numeric order: 1 → 2 → 3

| Phase | Plans Complete | Status | Completed |
|-------|----------------|--------|-----------|
| 1. Templates & Credential Foundation | 3/3 | Complete   | 2026-04-10 |
| 2. Skill Core — Interactive Flow & Merge Logic | 3/3 | Complete | 2026-04-13 |
| 3. Cross-Runtime Wiring & Distribution | 2/2 | Complete   | 2026-04-14 |
| 4. Subfolder Output Structure | 3/3 | Complete | 2026-04-15 |
| 5. New Instance Alias Prompt | 1/1 | Complete | 2026-04-16 |
| 6. Fix Taskfile Integration & UI Port Gap | 0/1 | Not started | - |
| 7. Fix MERGE-04 Idempotency UX | 0/1 | Not started | - |

### Phase 4: Subfolder Output Structure — Per-service subdirectory layout in .devtools/

**Goal:** Update `SKILL.md` so each installed service writes to `.devtools/<slug>/` (subfolder per instance) with per-service `.env`, updated root compose/Taskfile include paths, and a rename-before-add flow for multi-instance UX.
**Requirements**: TMPL-08, CRED-01, CRED-02, CRED-03, MERGE-01, MERGE-02, MERGE-03, MERGE-04, SKILL-03
**Depends on:** Phase 3
**Plans:** 3 plans

Plans:
- [x] 04-01-PLAN.md — Frontmatter/objective update, Step 1/1a detection paths, Step 2 .gitignore, Step 8 summary, Step 9 subfolder write paths with mkdir (D-11, D-12, D-13, D-14, D-01–D-04)
- [x] 04-02-PLAN.md — Step 10 .env split (per-service subfolder vs minimal root), Step 11 include path updates, root Taskfile template paths (D-05–D-10)
- [x] 04-03-PLAN.md — Step 1a rename-before-add flow, Step 13 done summary, success_criteria (D-15–D-18)

### Phase 5: New Instance Alias Prompt — Ask user for alias when installing into an existing service slot

**Goal:** Update `SKILL.md` with a full alias prompt UX: slug-preview prompt, silent normalization, empty/service-name validation, unlimited conflict re-prompt in Branch B; plus an optional proactive alias prompt on first install before the Q&A flow.
**Requirements**: MERGE-02, MERGE-03, MERGE-04
**Depends on:** Phase 4
**Plans:** 1 plan

Plans:
- [x] 05-01-PLAN.md — Rewrite Branch B alias validation loop (D-01–D-09) + insert proactive first-install alias prompt in Step 1 (D-10–D-13)

### Phase 6: Fix Taskfile Integration & UI Port Gap

**Goal:** Fix 3 critical integration breaks and REDIS_UI_PORT gap found in v1.0 milestone audit: update all 6 per-service Taskfile templates to subfolder compose paths, add alias `includes:` append instruction in SKILL.md Step 11b, fix Step 12 full path substitution, and add REDIS_UI_PORT question + `.env` write.
**Requirements**: TMPL-06, MERGE-03, CONF-01
**Gap Closure:** Closes Break 1 (P0), Break 2 (P1), Break 3 (P0), Gap 1 (P2) from v1.0 milestone audit
**Depends on:** Phase 5
**Plans:** 1 plan

Plans:
- [ ] 06-01-PLAN.md — Fix per-service Taskfile templates (Break 1) + SKILL.md Step 11b alias include append (Break 3) + Step 12 path substitution (Break 2) + REDIS_UI_PORT Q&A and env write (Gap 1)

### Phase 7: Fix MERGE-04 Idempotency UX

**Goal:** When re-running the skill for an already-installed alias, emit "already installed — nothing to do" and exit cleanly instead of entering the alias conflict re-prompt loop.
**Requirements**: MERGE-04
**Gap Closure:** Closes Gap 2 (P3) from v1.0 milestone audit
**Depends on:** Phase 6
**Plans:** 1 plan

Plans:
- [ ] 07-01-PLAN.md — Detect identical alias re-install in Step 1a Branch B conflict check; exit early with "already installed" message instead of prompting for a different alias
