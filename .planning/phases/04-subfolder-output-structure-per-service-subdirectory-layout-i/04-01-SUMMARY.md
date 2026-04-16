---
phase: 04-subfolder-output-structure-per-service-subdirectory-layout-i
plan: "01"
subsystem: skills/add-service
tags: [skill, subfolder-layout, detection, gitignore, compose, taskfile]
dependency_graph:
  requires: []
  provides: [subfolder-detection-paths, subfolder-write-paths, gitignore-glob]
  affects: [skills/add-service/SKILL.md]
tech_stack:
  added: []
  patterns: [per-service subfolder layout, mkdir-before-write, glob gitignore]
key_files:
  created: []
  modified:
    - skills/add-service/SKILL.md
decisions:
  - "D-11: Detection paths updated to .devtools/${SERVICE}/${SERVICE}.compose.yml (subfolder check)"
  - "D-12: .gitignore updated to include both .env and **/.env for subfolder coverage"
  - "D-13: No migration — old flat files continue to work; new installs always use subfolder layout"
  - "D-14: Frontmatter description and objective updated to mention .devtools/<service>/ subfolder layout"
  - "D-01/D-02: mkdir -p .devtools/${SERVICE_SLUG} added before compose write in Step 9a"
  - "D-03/D-04: Write destinations in Steps 9a and 9b use subfolder paths"
  - "D-07: env_file: .env resolves to subfolder .env automatically via Docker Compose include: — no template changes needed"
metrics:
  duration: "~3 minutes"
  completed: "2026-04-16T10:42:28Z"
  tasks_completed: 2
  files_modified: 1
---

# Phase 04 Plan 01: Update SKILL.md Steps 1/1a/2/8/9 for Subfolder Layout Summary

**One-liner:** Updated SKILL.md detection paths (Steps 1/1a), .gitignore glob (Step 2), confirmation summary (Step 8), and write paths with mkdir (Steps 9a/9b) to use the new `.devtools/<service>/` per-instance subfolder layout.

## What Was Built

Updated `skills/add-service/SKILL.md` with 9 precise find-and-replace edits across two tasks, establishing the subfolder output layout as the canonical detection and write strategy for the add-service skill.

### Changes Applied

**Task 1 — Frontmatter, objective, Step 1/1a detection, Step 2 .gitignore:**

| Edit | Location | Change |
|------|----------|--------|
| 1 | Frontmatter `description` | Added `.devtools/<service>/` subfolder mention (D-14) |
| 2 | `<objective>` block | Updated to reference per-service subfolder, per-service `.env`, minimal root `.env` (D-14) |
| 3 | Step 1 detection | `test -f .devtools/${SERVICE}/${SERVICE}.compose.yml` (D-11) |
| 4 | Step 1a installed message | Updated to show `.devtools/${SERVICE}/` path |
| 5 | Step 1a alias check | `test -f .devtools/${SERVICE_SLUG}/${SERVICE_SLUG}.compose.yml` (D-11) |
| 6 | Step 2 `.gitignore` content | Added `**/.env` glob + D-13 no-migration note (D-12/D-13) |

**Task 2 — Step 8 summary, Step 9a mkdir + write paths, Step 9b write path:**

| Edit | Location | Change |
|------|----------|--------|
| 7 | Step 8 files-to-write list | Shows `.devtools/redis/redis.compose.yml` style paths + per-service `.env` entry (D-01–D-06) |
| 8 | Step 9a write destination | Added `mkdir -p .devtools/${SERVICE_SLUG}` + subfolder write path + D-07 env_file note |
| 9 | Step 9b write destination | Updated to `.devtools/${SERVICE_SLUG}/<filename>.Taskfile.yml` (D-03/D-04) |

## Verification Results

```
test -f .devtools/${SERVICE}/${SERVICE}.compose.yml  → 1 match (Step 1) ✓
test -f .devtools/${SERVICE}.compose.yml             → 0 matches (old flat path gone) ✓
test -f .devtools/${SERVICE_SLUG}/${SERVICE_SLUG}.compose.yml → 1 match (Step 1a) ✓
test -f .devtools/${SERVICE_SLUG}.compose.yml        → 0 matches (old alias flat path gone) ✓
mkdir -p .devtools/${SERVICE_SLUG}                   → 1 match (Step 9a) ✓
devtools/redis/redis.compose.yml                     → 1 match (Step 8) ✓
devtools/redis/redis.Taskfile.yml                    → 1 match (Step 8) ✓
devtools/redis/.env                                  → 1 match (Step 8) ✓
**/.env                                              → 2 matches (objective + Step 2) ✓
D-07                                                 → 1 match (Step 9a env_file note) ✓
.devtools/<filename>.compose.yml (old flat write)    → 0 matches ✓
.devtools/<filename>.Taskfile.yml (old flat write)   → 0 matches ✓
```

## Commits

| Task | Commit | Description |
|------|--------|-------------|
| 1 | ae8f2fc | feat(04-01): update frontmatter, objective, Step 1/1a detection, Step 2 .gitignore for subfolder layout |
| 2 | 1a4d653 | feat(04-01): update Step 8 summary and Step 9a/9b write paths with mkdir for subfolder layout |

## Deviations from Plan

None — plan executed exactly as written. All 9 find-and-replace edits applied as specified.

## Known Stubs

None.

## Threat Flags

None — no new network endpoints, auth paths, or security surface introduced. This is a documentation-only (SKILL.md) change.

## Self-Check: PASSED

- `skills/add-service/SKILL.md` — modified ✓
- commit `ae8f2fc` exists ✓
- commit `1a4d653` exists ✓
- All acceptance criteria verified via grep ✓
