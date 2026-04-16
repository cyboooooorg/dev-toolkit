---
phase: 04-subfolder-output-structure-per-service-subdirectory-layout-i
plan: "03"
subsystem: skills/add-service
tags: [skill, merge-detection, rename-flow, ux, step-1a, done-summary]
dependency_graph:
  requires: [04-01-PLAN.md, 04-02-PLAN.md]
  provides: [rename-before-add UX, updated done summary, updated success_criteria]
  affects: [skills/add-service/SKILL.md]
tech_stack:
  added: []
  patterns:
    - "rename-before-add: offer rename of existing service slot before alias-only fallback"
    - "Branch A / Branch B structure for conflict resolution in Step 1a"
    - "D-18 atomic rename: all mv/substitution/reference ops complete before any new writes"
key_files:
  modified:
    - skills/add-service/SKILL.md
decisions:
  - "D-15: rename-before-add offered first when service already exists at .devtools/${SERVICE}/"
  - "D-16: 8-step rename flow — folder mv, file mv, in-file substitution, compose.yml update, Taskfile.yml update, .env key rename with ## REPLACED markers"
  - "D-17: confirmation message 'Existing ${SERVICE} renamed to ${RENAME_SLUG}. Now installing new ${SERVICE} instance...'"
  - "D-18: all rename operations are atomic — completed before any new files are written"
metrics:
  duration: "~10m"
  completed_date: "2026-04-16"
  tasks_completed: 2
  files_modified: 1
---

# Phase 04 Plan 03: Rename-Before-Add Flow + Done Summary Update

**One-liner:** Step 1a rewritten with rename-before-add (Branch A) and alias-or-cancel fallback (Branch B), Step 13 done summary and `<success_criteria>` updated to reflect subfolder output paths.

## What Was Built

### Task 1: Step 1a rename-before-add flow (D-15 to D-18)

Rewrote the entire `## Step 1a: Merge Detection` block. The old single-path "alias-or-cancel" prompt was replaced with a two-branch structure:

**Branch A — rename the existing instance first:**
When the user provides a rename suffix, the skill performs 8 atomic operations before any new writes:
1. Set rename variables (`RENAME_SLUG`, `RENAME_SNAKE`, `RENAME_PREFIX`)
2. `mv .devtools/${SERVICE} .devtools/${RENAME_SLUG}` (folder rename)
3. Rename compose/Taskfile files inside the folder
4. Apply in-file string substitutions (YAML keys, container names, volume/network keys, env var names, filename references)
5. Update `.devtools/compose.yml` include path
6. Update `.devtools/Taskfile.yml` include block
7. Update `.env` keys with `## REPLACED` markers and new-name duplicates
8. Announce completion: "Existing ${SERVICE} renamed to ${RENAME_SLUG}. Now installing new ${SERVICE} instance..." (D-17)
9. Proceed to Step 3 with the slot now free (D-18 atomicity guarantee)

**Branch B — skip rename, use alias-or-cancel:**
The original alias-or-cancel fallback is preserved intact for users who choose not to rename.

### Task 2: Step 13 done summary and success_criteria update

Three precise edits to bring the file list and criteria in line with the subfolder layout:

- **Edit 1:** Step 13 file list now shows `.devtools/redis/redis.compose.yml` style paths, plus a conditional rename output block (folder renamed from, renamed compose/Taskfile/env files)
- **Edit 2:** `<success_criteria>` updated — subfolder paths, root `.devtools/.env` is minimal (COMPOSE_PROJECT_NAME + COMPOSE_PROFILES only), rename-first behavior noted in re-run description, post-rename state described
- **Edit 3:** Warning messages reference per-service `.devtools/${SERVICE_SLUG}/.env`; commit reminder updated to cover all `.devtools/*/.env` files

## Verification Results

All 18 decisions across Phase 4 are covered in SKILL.md:

| Decision | Result |
|----------|--------|
| D-11: subfolder detection | ✓ 1 match |
| D-12: .gitignore `**/.env` | ✓ 4 matches |
| D-01/D-02: mkdir with SERVICE_SLUG | ✓ 1 match |
| D-05: per-service .env in subfolder | ✓ 8 matches |
| D-06: root .env minimal | ✓ 4 matches |
| D-09: compose include subfolder path | ✓ 4 matches |
| D-15/D-16: rename flow (RENAME_SLUG) | ✓ 16 matches |
| D-17: rename confirmation | ✓ 1 match |
| D-14: frontmatter description | ✓ 7 matches |

## Commits

| Task | Commit | Description |
|------|--------|-------------|
| Task 1 | ac43da1 | feat(04-03): rewrite Step 1a with rename-before-add flow (D-15 to D-18) |
| Task 2 | 99fd2e0 | feat(04-03): update Step 13 done summary and success_criteria for subfolder layout |

## Deviations from Plan

None — plan executed exactly as written.

## Known Stubs

None — no placeholder content; SKILL.md is the authoritative skill definition.

## Threat Flags

None — no new network endpoints, auth paths, file access patterns, or schema changes introduced.

## Self-Check: PASSED

- [x] `skills/add-service/SKILL.md` exists and contains all rename flow content
- [x] Commit ac43da1 exists: `feat(04-03): rewrite Step 1a with rename-before-add flow (D-15 to D-18)`
- [x] Commit 99fd2e0 exists: `feat(04-03): update Step 13 done summary and success_criteria for subfolder layout`
- [x] `grep -c 'RENAME_SLUG' skills/add-service/SKILL.md` → 16 (≥4 required)
- [x] `grep -c 'Branch A\|Branch B' skills/add-service/SKILL.md` → 2
- [x] `grep -c '\.devtools/redis/redis\.compose\.yml' skills/add-service/SKILL.md` → 2 (≥2 required)
- [x] Old `<success_criteria>` path `.devtools/<service>.compose.yml` → 0 matches ✓
