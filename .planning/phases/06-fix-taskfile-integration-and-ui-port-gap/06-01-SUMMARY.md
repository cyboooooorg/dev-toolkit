---
phase: 06
plan: 06-01
title: "Fix Taskfile Integration Breaks & UI Port Gap"
subsystem: taskfile-templates, skills
tags: [bugfix, taskfile, compose-paths, skill-md, redis-ui]
dependency_graph:
  requires: []
  provides: [TMPL-06, MERGE-03, CONF-01]
  affects: [taskfile-templates/redis, taskfile-templates/rabbitmq, taskfile-templates/postgres, taskfile-templates/mysql, taskfile-templates/mongodb, taskfile-templates/monitoring, skills/add-service/SKILL.md]
tech_stack:
  added: []
  patterns: [subfolder compose paths, alias include append, full-path substitution, ui port env var]
key_files:
  created: []
  modified:
    - taskfile-templates/redis/redis.Taskfile.yml
    - taskfile-templates/rabbitmq/rabbitmq.Taskfile.yml
    - taskfile-templates/postgres/postgres.Taskfile.yml
    - taskfile-templates/mysql/mysql.Taskfile.yml
    - taskfile-templates/mongodb/mongodb.Taskfile.yml
    - taskfile-templates/monitoring/monitoring.Taskfile.yml
    - skills/add-service/SKILL.md
decisions:
  - "All Taskfile templates now reference .devtools/<service>/<service>.compose.yml matching Phase 4 subfolder layout"
  - "Alias installs require explicit includes entry append to .devtools/Taskfile.yml — optional:true only covers 6 base services"
  - "Step 12 path substitution must replace full .devtools/${SERVICE}/${SERVICE}.compose.yml not just filename"
  - "REDIS_UI_PORT must be asked (Step 6) and written to .env (Step 10a) when ui_enabled=true"
metrics:
  duration: "~10 minutes"
  completed: "2026-04-16"
  tasks_completed: 4
  files_modified: 7
---

# Phase 06 Plan 01: Fix Taskfile Integration Breaks & UI Port Gap Summary

**One-liner:** Fixed 4 integration gaps — 6 Taskfile templates updated to Phase 4 subfolder compose paths, SKILL.md alias include-append added, Step 12 full-path substitution fixed, REDIS_UI_PORT env var wired end-to-end.

## Objective

Closed 4 integration gaps (3 critical breaks + 1 functional gap) identified in the v1.0 milestone audit:

1. **TMPL-06 (P0):** All 6 per-service Taskfile templates still referenced flat pre-Phase-4 compose paths, breaking every `task redis:up`-style command.
2. **MERGE-03 (P0):** Alias installs produced unreachable tasks because no `includes:` entry was ever appended to `.devtools/Taskfile.yml`.
3. **MERGE-03 (P1):** Alias Taskfile content kept wrong compose path (only filename replaced, not full path).
4. **CONF-01 (P2):** `--profile ui` on Redis silently failed because `REDIS_UI_PORT` was never asked or written to `.env`.

## Tasks Completed

| Task | Name | Commit | Files Modified |
|------|------|--------|----------------|
| 1 | Update all 6 Taskfile templates to subfolder compose paths | ed93e7b | 6 Taskfile templates |
| 2 | Add alias include-append instruction to SKILL.md Step 11b | 902d113 | skills/add-service/SKILL.md |
| 3 | Fix SKILL.md Step 12 Taskfile path substitution | a1835f9 | skills/add-service/SKILL.md |
| 4 | Add REDIS_UI_PORT to Step 6 and Step 10a | a482409 | skills/add-service/SKILL.md |

## Files Modified

### Taskfile Templates (all 6)

Each file received two changes:
1. `# Installed to:` comment updated from `.devtools/<service>.Taskfile.yml` → `.devtools/<service>/<service>.Taskfile.yml`
2. All `-f .devtools/<service>.compose.yml` flags replaced with `-f .devtools/<service>/<service>.compose.yml`

| File | Old path occurrences fixed | New path count |
|------|---------------------------|----------------|
| taskfile-templates/redis/redis.Taskfile.yml | 6 | 6 |
| taskfile-templates/rabbitmq/rabbitmq.Taskfile.yml | 6 | 6 |
| taskfile-templates/postgres/postgres.Taskfile.yml | 6 | 6 |
| taskfile-templates/mysql/mysql.Taskfile.yml | 6 | 6 |
| taskfile-templates/mongodb/mongodb.Taskfile.yml | 6 | 6 |
| taskfile-templates/monitoring/monitoring.Taskfile.yml | 5 | 5 |

### SKILL.md (skills/add-service/SKILL.md)

**Step 11b:** Added "Exception — alias installs" block between "No append is needed for the Taskfile. (D-10)" and "**Important:** Never overwrite" paragraphs. Block instructs executor to append 2-space-indented `includes:` entry for `${SERVICE_SLUG}` with `optional: true` to `.devtools/Taskfile.yml` when `MODE=alias`.

**Step 12 Taskfile substitutions:** Changed partial filename rule `` `${SERVICE}.compose.yml` → `${SERVICE_SLUG}.compose.yml` `` to full-path rule `` `.devtools/${SERVICE}/${SERVICE}.compose.yml` → `.devtools/${SERVICE_SLUG}/${SERVICE_SLUG}.compose.yml` ``.

**Step 6:** Updated redis heading from `#### redis (RedisInsight — no extra credentials needed)` to `#### redis (RedisInsight)` and added `Ask: "RedisInsight UI port? [default: 8001]" → ANSWERS[ui_port]`.

**Step 10a:** Updated redis env-vars row to include `REDIS_UI_PORT (if ANSWERS[ui_enabled]=true)`.

## Verification Results

```
1. No flat compose paths remain in any Taskfile template:
   PASS: no flat paths

2. All 6 templates have subfolder paths:
   redis: 6 subfolder path(s)
   rabbitmq: 6 subfolder path(s)
   postgres: 6 subfolder path(s)
   mysql: 6 subfolder path(s)
   mongodb: 6 subfolder path(s)
   monitoring: 5 subfolder path(s)

3. SKILL.md alias exception block and full-path substitution rule:
   D-10 alias extension: 1
   Full-path substitution rule: 2 (appears in Step 12 + alias exception block)

4. SKILL.md REDIS_UI_PORT coverage:
   ANSWERS[ui_port]: 2 (Step 6 redis + Step 6 rabbitmq — correct)
   REDIS_UI_PORT in Step 10a: 1
```

## Requirements Addressed

| Requirement | Description | Status |
|-------------|-------------|--------|
| TMPL-06 | Per-service Taskfile templates reference correct subfolder compose paths | ✅ Satisfied |
| MERGE-03 | Alias installs append includes entry + use full-path substitution | ✅ Satisfied |
| CONF-01 | REDIS_UI_PORT asked in Step 6 and written in Step 10a | ✅ Satisfied |

## Deviations from Plan

None — plan executed exactly as written.

## Known Stubs

None — all changes are structural fixes with no placeholder content.

## Threat Flags

None — changes are template path corrections and SKILL.md instruction text. No new network endpoints, auth paths, file access patterns, or schema changes introduced.

## Self-Check

## Self-Check: PASSED

| Item | Status |
|------|--------|
| taskfile-templates/redis/redis.Taskfile.yml | FOUND |
| taskfile-templates/rabbitmq/rabbitmq.Taskfile.yml | FOUND |
| taskfile-templates/postgres/postgres.Taskfile.yml | FOUND |
| taskfile-templates/mysql/mysql.Taskfile.yml | FOUND |
| taskfile-templates/mongodb/mongodb.Taskfile.yml | FOUND |
| taskfile-templates/monitoring/monitoring.Taskfile.yml | FOUND |
| skills/add-service/SKILL.md | FOUND |
| .planning/phases/06-fix-taskfile-integration-and-ui-port-gap/06-01-SUMMARY.md | FOUND |
| Commit ed93e7b (Task 1) | FOUND |
| Commit 902d113 (Task 2) | FOUND |
| Commit a1835f9 (Task 3) | FOUND |
| Commit a482409 (Task 4) | FOUND |
