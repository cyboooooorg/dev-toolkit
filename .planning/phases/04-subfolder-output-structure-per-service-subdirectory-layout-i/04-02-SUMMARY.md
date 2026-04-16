---
phase: 04-subfolder-output-structure-per-service-subdirectory-layout-i
plan: "02"
subsystem: skills/add-service
tags: [skill, subfolder-layout, env-split, compose-include, taskfile-include]
dependency_graph:
  requires: [04-01-PLAN.md]
  provides: [per-service-env-split, root-env-minimal, subfolder-compose-includes, subfolder-taskfile-includes]
  affects: [skills/add-service/SKILL.md, taskfile-templates/root/Taskfile.yml]
tech_stack:
  added: []
  patterns: [per-service .env in subfolder, minimal root .env, subfolder-relative compose include paths, subfolder-relative Taskfile include paths]
key_files:
  created: []
  modified:
    - skills/add-service/SKILL.md
    - taskfile-templates/root/Taskfile.yml
decisions:
  - "D-05: Service credentials written to .devtools/${SERVICE_SLUG}/.env (per-service subfolder)"
  - "D-06: Root .devtools/.env holds ONLY COMPOSE_PROJECT_NAME and COMPOSE_PROFILES (first install only)"
  - "D-08: Root files (.devtools/compose.yml, Taskfile.yml, .gitignore, .env) stay at .devtools/ root"
  - "D-09: compose.yml include entries use path: <slug>/<filename>.compose.yml format"
  - "D-10: Taskfile includes use subfolder-relative paths (redis/redis.Taskfile.yml) — updated in template"
  - "Monitoring subfolder: .devtools/monitoring/ created with mkdir -p in Step 10d"
metrics:
  duration: "~3 minutes"
  completed: "2026-04-16T10:46:50Z"
  tasks_completed: 2
  files_modified: 2
---

# Phase 04 Plan 02: Update SKILL.md Step 10/11 and Root Taskfile Template for Subfolder Layout Summary

**One-liner:** Split .env management into per-service subfolder writes (Step 10a) and minimal root writes (Step 10a-root), updated monitoring files to `.devtools/monitoring/` subfolder, and updated compose/Taskfile include paths to use `<slug>/<filename>` format throughout.

## What Was Built

Updated `skills/add-service/SKILL.md` with restructured Step 10 (.env management) and Step 11 (root file management), plus updated `taskfile-templates/root/Taskfile.yml` to reflect the subfolder include path convention.

### Changes Applied

**Task 1 — Step 10 .env management split (SKILL.md):**

| Edit | Location | Change |
|------|----------|--------|
| 1 | Step 10a header | Renamed to "Write per-service .devtools/${SERVICE_SLUG}/.env (D-05)" |
| 2 | Step 10a body | Service credentials go to `.devtools/${SERVICE_SLUG}/.env`; conflict detection references subfolder path |
| 3 | Step 10a-root (new) | New sub-step for root `.devtools/.env`: COMPOSE_PROJECT_NAME + COMPOSE_PROFILES only (D-06) |
| 4 | Step 10b | .env.example now documents project-level structure with subfolder references; no per-service keys appended |
| 5 | Step 10c | Monitoring vars go to `.devtools/monitoring/.env` (not root `.devtools/.env`) |
| 6 | Step 10d | Monitoring files go to `.devtools/monitoring/` subfolder with `mkdir -p .devtools/monitoring`; include path updated to `monitoring/monitoring.compose.yml` |

**Task 2 — Step 11 include paths + root Taskfile template:**

| Edit | Location | Change |
|------|----------|--------|
| 7 | Step 11a first-install block | `path: <slug>/<filename>.compose.yml` format (D-09) |
| 8 | Step 11a subsequent-install block | `path: <slug>/<filename>.compose.yml` format (D-09) |
| 9 | Step 11b commentary | Updated to mention subfolder-relative paths + D-08 root file locations note |
| 10 | taskfile-templates/root/Taskfile.yml | All 6 includes: `./service.Taskfile.yml` → `service/service.Taskfile.yml` (D-10) |

## Verification Results

```
.devtools/${SERVICE_SLUG}/.env references   → 5 matches (Step 10a header, write target, conflict detection, why note, 10b reference) ✓
10a-root sub-step                           → 1 match ✓
COMPOSE_PROJECT_NAME documented             → 4+ matches (still in root .env section) ✓
monitoring/monitoring.compose.yml           → 5 matches (Step 10d write + include update) ✓
mkdir -p .devtools/monitoring               → 2 matches (10c reference + 10d actual command) ✓
.devtools/monitoring.compose.yml (old flat) → 0 matches ✓
Append to .devtools/.env (old pattern)      → 0 matches ✓
path: <slug>/<filename>.compose.yml         → 2 matches (first + subsequent install) ✓
- ./<...>.compose.yml (old flat include)    → 0 matches in Step 11a context ✓
redis/redis.Taskfile.yml in Step 11b        → mentioned in commentary ✓
D-08 / Root file locations                  → 2 matches ✓
redis/redis.Taskfile.yml in template        → 1 match ✓
rabbitmq/rabbitmq.Taskfile.yml in template  → 1 match ✓
postgres/postgres.Taskfile.yml in template  → 1 match ✓
mysql/mysql.Taskfile.yml in template        → 1 match ✓
mongodb/mongodb.Taskfile.yml in template    → 1 match ✓
monitoring/monitoring.Taskfile.yml          → 1 match ✓
./redis.Taskfile.yml (old flat path)        → 0 matches ✓
```

## Commits

| Task | Commit | Description |
|------|--------|-------------|
| 1 | 8bfe402 | feat(04-02): rewrite Step 10 .env management — split per-service subfolder vs root |
| 2 | b4d0ea0 | feat(04-02): update Step 11 compose include paths and root Taskfile template to subfolder layout |

## Deviations from Plan

**Minor: 2 matches for `mkdir -p .devtools/monitoring` instead of 1**
- **Found during:** Task 1 verification
- **Issue:** Plan acceptance criteria specified "exactly 1 match" but plan replacement text itself produces 2 occurrences — one reference in Step 10c (`will be run in Step 10d`) and one actual command block in Step 10d.
- **Resolution:** Both occurrences are intentional per plan replacement text. The spirit of the criterion (mkdir is present and in the right step) is satisfied. No fix needed.

## Known Stubs

None.

## Threat Flags

None — no new network endpoints, auth paths, or security surface introduced. All changes are documentation-only (SKILL.md) and template-only (Taskfile.yml). T-04-02-01 (information disclosure) mitigated: `.env.example` now documents structure without leaking credentials; per-service `.env` files covered by `**/.env` gitignore from Plan 01.

## Self-Check: PASSED

- `skills/add-service/SKILL.md` — modified ✓
- `taskfile-templates/root/Taskfile.yml` — modified ✓
- commit `8bfe402` exists ✓
- commit `b4d0ea0` exists ✓
- All acceptance criteria verified via grep ✓
