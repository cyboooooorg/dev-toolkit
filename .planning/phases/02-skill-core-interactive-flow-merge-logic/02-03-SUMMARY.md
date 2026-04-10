---
phase: 2
plan: 3
subsystem: skill-core
tags: [skill, SKILL.md, file-writing, merge-logic, alias-substitution, env-management]
dependency_graph:
  requires: [skills/add-service/SKILL.md with Steps 1–8 (from 02-01, 02-02)]
  provides: [skills/add-service/SKILL.md complete — Steps 9–13, closed <process>, <success_criteria>]
  affects: []
tech_stack:
  added: []
  patterns:
    - Write-order safety (per-service file before compose.yml include)
    - Alias substitution map (SERVICE_SNAKE for YAML keys, SERVICE_SLUG for filenames/containers, ENV_PREFIX for env vars)
    - .env conflict marker (## REPLACED inline comment + new value appended)
    - Root Taskfile copy-once-never-overwrite pattern (optional: true makes per-service appends unnecessary)
key_files:
  created: []
  modified:
    - skills/add-service/SKILL.md
decisions:
  - "Write order enforced: per-service compose file written in Step 9 before compose.yml include updated in Step 11 (Pitfall 6 guard)"
  - "Root Taskfile.yml copied from template only on first install — never overwritten (Pitfall 1 guard)"
  - "Redis no-password: --requirepass and healthcheck -a flags removed when ANSWERS[password] is empty string (not passed as empty)"
  - "YAML service keys use underscore (redis_cache:); container names and filenames use hyphen (redis-cache) — explicit note in Step 12"
  - "Monitoring files written before compose.yml include updated — same write-order rule as Step 9"
metrics:
  duration: ~20m
  completed: 2026-04-10
  tasks_completed: 2
  files_changed: 1
---

# Phase 2 Plan 3: File Writing + Merge Logic Summary

## One-liner

Steps 9–13 appended to SKILL.md completing all file writes: per-service compose/Taskfile (verbatim standard, in-memory alias substitution), .env append with conflict detection (## REPLACED), root compose.yml create-or-append, Taskfile.yml copy-once-never-overwrite, alias substitution map, and done summary with security reminder — plus closed `</process>` and `<success_criteria>` section.

## What Was Built

Completed `skills/add-service/SKILL.md` with the file-writing and merge-logic steps:

**Task 1 — Steps 9–11 (file writes, .env management, root file management):**

- **Step 9a:** Reads compose template from skill repo; copies verbatim for standard install; removes `--requirepass` and healthcheck `-a` flags for Redis when password is empty; performs in-memory alias substitution before write for alias installs; writes per-service file to `.devtools/` BEFORE updating `compose.yml` (write-order Pitfall 6 guard)
- **Step 9b:** Reads per-service Taskfile template; performs alias substitutions for alias installs; writes to `.devtools/<filename>.Taskfile.yml`
- **Step 10a:** Appends service env vars to `.devtools/.env` with section header; conflict detection: marks old line with ` ## REPLACED` inline comment, appends new value below; handles `COMPOSE_PROJECT_NAME` prepend if absent
- **Step 10b:** Appends dummy/placeholder values to `.devtools/.env.example` — never real credentials
- **Step 10c:** Appends monitoring env vars (with conflict detection) when `ANSWERS[monitoring]=true`
- **Step 10d:** Writes monitoring compose and Taskfile files verbatim; updates `compose.yml` include AFTER writing monitoring file (write-order rule)
- **Step 11a:** Creates `.devtools/compose.yml` with `include:` block on first install; appends new `include:` line on subsequent installs without rewriting the file
- **Step 11b:** Copies `taskfile-templates/root/Taskfile.yml` to `.devtools/Taskfile.yml` only when file does not exist; never overwrites (Pitfall 1 guard); explains `optional: true` makes per-service appends unnecessary

**Task 2 — Steps 12–13 + `</process>` + `<success_criteria>`:**

- **Step 12:** Complete alias substitution map: YAML service key → `SERVICE_SNAKE` (underscore), container name suffix → `SERVICE_SLUG` (hyphen), volume/network keys/refs → `SERVICE_SNAKE`, env var names → `ENV_PREFIX_*`; explicit note that YAML keys must use underscore to avoid Docker Compose network auto-naming issues; Taskfile substitutions for compose filename and service name references
- **Step 13:** Done summary with complete files-written list (skipping files that were skipped), next-step `task` commands for standard and aliased installs, `## REPLACED` conflict warning block (only shown when conflicts occurred), and permanent security reminder: "Do not commit .devtools/.env"
- Closed `</process>` block (intentionally left open since 02-01)
- Added `<success_criteria>` section with 10 verification checks

## Threat Mitigations Applied

| Threat | Mitigation Applied |
|--------|-------------------|
| Per-service compose.yml written then include added — partial failure leaves dangling reference (Pitfall 6) | Step 9 writes per-service file; Step 11 updates compose.yml include — separate steps with explicit write-order note |
| `.devtools/Taskfile.yml` overwritten on second install (Pitfall 1) | Step 11b: `test -f .devtools/Taskfile.yml` guard; "Never overwrite" note explicit |
| `.devtools/.env` appended with duplicate key without detection | Step 10a: conflict detection with `## REPLACED` marker (D-13) |
| Credentials written to `.devtools/.env` committed to git | Step 13: "Do not commit .devtools/.env" reminder present in every done summary |
| Alias substitution missed for specific field | Step 12: explicit checklist of every substitution target (env vars, service key, container name, volume key+ref, network key+ref, filename, Taskfile commands) |

## Commits

| Task | Commit | Files |
|------|--------|-------|
| Task 1: Append Steps 9–11 | 7fd200c | skills/add-service/SKILL.md (appended +178 lines) |
| Task 2: Append Steps 12–13, close process, add success_criteria | a44a9c4 | skills/add-service/SKILL.md (appended +100 lines) |

## Deviations from Plan

None — plan executed exactly as written.

## Known Stubs

None — `skills/add-service/SKILL.md` is complete and self-contained. All 13 steps (plus Step 1a) are present. The `<process>` block is closed. The `<success_criteria>` section is present. No stubs or placeholder text.

## Threat Flags

None — no new network endpoints, auth paths, file access patterns, or schema changes introduced. This plan only extends a SKILL.md markdown file.

## Self-Check: PASSED

- [x] `skills/add-service/SKILL.md` exists (542 lines)
- [x] 14 step headings present (Steps 1, 1a, 2–13)
- [x] Step 9: compose template read verbatim (standard), alias in-memory substitution, Redis no-password flag removal
- [x] Step 9: "Always write the per-service file (Step 9) BEFORE updating compose.yml (Step 11)" — write-order guard present
- [x] Step 10: .env append with section header, ## REPLACED conflict detection, COMPOSE_PROJECT_NAME prepend
- [x] Step 10: .env.example with dummy values only
- [x] Step 10d: monitoring files written before compose.yml include updated (Pitfall 6 guard)
- [x] Step 11a: compose.yml create-or-append
- [x] Step 11b: "Never overwrite an existing .devtools/Taskfile.yml" — Pitfall 1 guard
- [x] Step 12: complete alias substitution map (SERVICE_SNAKE, SERVICE_SLUG, ENV_PREFIX)
- [x] Step 12: "YAML service keys must use underscore" note present
- [x] Step 13: files-written list, next-step task commands, REPLACED conflict warning, do-not-commit reminder
- [x] `</process>` closed
- [x] `<success_criteria>` present and closed
- [x] Both tasks committed individually (7fd200c, a44a9c4)
- [x] All plan verification greps pass
