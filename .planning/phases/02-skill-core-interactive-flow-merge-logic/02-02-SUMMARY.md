---
phase: 2
plan: 2
subsystem: skill-core
tags: [skill, SKILL.md, Q&A-flow, service-selection, confirmation-gate, inline-defaults]
dependency_graph:
  requires: [skills/add-service/SKILL.md with Steps 1–2 (from 02-01)]
  provides: [skills/add-service/SKILL.md Steps 3–8 (Q&A flow, confirmation gate)]
  affects: [02-03]
tech_stack:
  added: []
  patterns:
    - Per-service Q&A branching pattern in SKILL.md
    - Inline defaults on every question ([default: X])
    - Confirmation gate before file writes (Step 8)
    - Password masking in summary table
key_files:
  created: []
  modified:
    - skills/add-service/SKILL.md
decisions:
  - "CONF-03 inline defaults: per-service port defaults hardcoded in per-service sections (6379/5672/5432/3306/27017) so verification greps and AI guidance are unambiguous"
  - "Universal port question uses [default: <port from metadata>] with per-service port notes below for completeness"
  - "Step 6 explicitly skipped for RabbitMQ — management UI always-on, handled entirely in Step 5"
  - "Redis password shown as optional with [optional — press enter to skip] not a required prompt"
  - "MYSQL_ROOT_PASSWORD explicitly required in Step 5 mysql section despite token: null in metadata"
metrics:
  duration: ~15m
  completed: 2026-04-10
  tasks_completed: 2
  files_changed: 1
---

# Phase 2 Plan 2: Interactive Q&A Flow Summary

## One-liner

Steps 3–8 appended to SKILL.md: service selection, metadata read, per-service Q&A for all 5 services (with inline port/credential defaults), UI companion prompts, optional feature prompts, and a confirmation gate that blocks file writes until the user answers Yes.

## What Was Built

Extended `skills/add-service/SKILL.md` with the complete interactive configuration conversation:

**Task 1 — Steps 3–5 (service selection, metadata read, per-service Q&A):**

- **Step 3:** Asks user to select a service when `$ARGUMENTS` not set; normalises to lowercase; rejects unknown services with explicit error; returns to Step 1 for merge detection
- **Step 4:** Reads `compose-templates/${SERVICE}/metadata.json`; extracts `parameters[]`, `ui_companion`, `exporter`
- **Step 5:** Universal questions (port, version) with `[default: X]` shown inline; per-service credential branches:
  - **redis:** password optional with `[optional — press enter to skip]`
  - **rabbitmq:** username, password, Management UI port `[default: 15672]`; UI prompt in Step 6 explicitly skipped
  - **postgres:** username, password, database name
  - **mysql:** username, password, database name, MYSQL_ROOT_PASSWORD (token: null pitfall avoided)
  - **mongodb:** username, password
  - Each service has its explicit port default annotated (6379 / 5672 / 5432 / 3306 / 27017)

**Task 2 — Steps 6–8 (UI companion, optional prompts, confirmation gate):**

- **Step 6:** UI companion `[y/N]` for redis/postgres/mysql/mongodb; explicitly skipped for rabbitmq; pgAdmin email+password for postgres; Mongo Express auth for mongodb when auth enabled; mysql (phpMyAdmin uses service creds — no extra questions); redis (RedisInsight uses Redis password — no extra questions)
- **Step 7:** Taskfile tasks `[Y/n]` (default yes); monitoring `[y/N]` (default no); Grafana admin password surfaced when monitoring enabled (default `admin` shown so user sees weak default)
- **Step 8:** Summary table with all collected values, passwords masked with `****`; files-to-write list; `"Write these files? [Y/n]"` gate; explicit statement that nothing is written before this confirmation (D-02 threat mitigation)

## Threat Mitigations Applied

| Threat | Mitigation Applied |
|--------|-------------------|
| Files written without review (D-02) | Step 8 explicitly states "Do NOT write any files before this confirmation. Steps 9–13 perform all writes." |
| Grafana default password `admin` written silently | Step 7 surfaces `[default: admin]` so user sees and can change it |
| pgAdmin / Mongo Express auth defaulting to OFF silently | Step 6 prompts explicitly; user must type `y` to enable auth |

## Commits

| Task | Commit | Files |
|------|--------|-------|
| Task 1: Append Steps 3–5 | 413037d | skills/add-service/SKILL.md (appended) |
| Task 2: Append Steps 6–8 | eab0a73 | skills/add-service/SKILL.md (appended) |

## Deviations from Plan

### Auto-fixed Issues

**1. [Rule 1 - Verification Criteria] Added explicit per-service port defaults to satisfy CONF-03 grep**
- **Found during:** Task 2 verification (full plan verification run)
- **Issue:** Plan verification checks `grep "default: 6379\|default: 5672\|..."` but plan instruction text uses generic `[default: <port from metadata>]` for the universal port question — grep would never match
- **Fix:** Added per-service port default annotations at the start of each per-service section (e.g. `*(Port default: \`[default: 6379]\`)* `) — satisfies both the grep check and makes the SKILL.md unambiguous for the AI following it
- **Files modified:** skills/add-service/SKILL.md
- **Commit:** eab0a73 (included in Task 2 commit)

## Known Stubs

- `<process>` block is still intentionally open (no closing tag) — Steps 9–13 and the closing tag are written in 02-03.

## Self-Check: PASSED

- [x] `skills/add-service/SKILL.md` exists and contains all 6 steps (3–8)
- [x] Step 3: service selection prompt, rejects unknown services, returns to Step 1
- [x] Step 4: reads metadata.json, extracts parameters/ui_companion/exporter
- [x] Step 5: universal questions (port, version) + all 5 per-service credential branches
- [x] Redis password is `[optional — press enter to skip]` — not required
- [x] RabbitMQ asks Management UI port `[default: 15672]`; Step 6 skipped for RabbitMQ
- [x] MySQL asks MYSQL_ROOT_PASSWORD (token: null pitfall avoided)
- [x] PostgreSQL asks pgadmin_email + pgadmin_password when UI enabled
- [x] MongoDB asks Mongo Express auth when ui_auth=true
- [x] Step 6: UI companion for redis/postgres/mysql/mongodb; explicitly skipped for rabbitmq
- [x] Step 7: Taskfile [Y/n], monitoring [y/N], Grafana password when monitoring=yes
- [x] Step 8: summary table with passwords masked, files-to-write list, confirmation gate
- [x] Step 8: "Do NOT write any files before this confirmation" (D-02)
- [x] CONF-03: all 5 service port defaults hardcoded inline (6379/5672/5432/3306/27017)
- [x] Both tasks committed individually (413037d, eab0a73)
