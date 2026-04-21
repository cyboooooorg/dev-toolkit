---
phase: 07-fix-merge-04-idempotency-ux
plan: "01"
subsystem: skill-instructions
tags: [skill, idempotency, alias-slug, add-service, MERGE-04]

# Dependency graph
requires:
  - phase: 05-alias-prompt
    provides: Alias slug install flow and Step 1a rename-before-add logic
  - phase: 04-subfolder-output
    provides: .devtools/<service>/<service>.compose.yml subfolder layout
provides:
  - "Updated Step 1 conditional in skills/add-service/SKILL.md with alias-slug early-exit"
  - "Idempotent re-run of alias slugs: exits cleanly with 'already installed' message"
affects:
  - add-service skill consumers
  - any plan that extends Step 1 detection logic

# Tech tracking
tech-stack:
  added: []
  patterns:
    - "Two-branch exists conditional: base service names route to Step 1a; alias slugs exit early"

key-files:
  created: []
  modified:
    - skills/add-service/SKILL.md

key-decisions:
  - "The 6 base service names (redis, rabbitmq, postgres, mysql, mongodb, monitoring) are the discriminator between re-add intent and idempotent re-run"
  - "Alias slug idempotency check is placed at the test -f branch point in Step 1, before Step 1a is entered"

patterns-established:
  - "Pattern: early-exit on idempotent conditions with a clear message before entering interactive flows"

requirements-completed:
  - MERGE-04

# Metrics
duration: 5min
completed: 2026-04-21
---

# Phase 7 Plan 01: Fix MERGE-04 — Alias-Slug Idempotency Check Summary

**SKILL.md Step 1 now exits cleanly with "already installed" when re-run with an alias slug, while base service names still enter the rename-before-add flow (Step 1a)**

## Performance

- **Duration:** ~5 min
- **Started:** 2026-04-21T09:16:00Z
- **Completed:** 2026-04-21T09:21:27Z
- **Tasks:** 1
- **Files modified:** 1

## Accomplishments
- Replaced single-line `- If the file **exists** → go to **Step 1a**.` with a two-branch conditional
- Alias slugs (already-installed services not in the 6 base names list) now emit a clean "already installed — nothing to do." message and stop
- Base service names (redis, rabbitmq, postgres, mysql, mongodb, monitoring) still proceed to Step 1a as before
- MERGE-04 closed: idempotent re-run of an alias slug no longer enters the interactive rename flow

## Task Commits

Each task was committed atomically:

1. **Task 1: Fix SKILL.md Step 1 — alias-already-installed idempotency check** - `236c0c9` (fix)

**Plan metadata:** _(docs commit follows)_

## Files Created/Modified
- `skills/add-service/SKILL.md` - Step 1 two-branch conditional replacing old single-line exists redirect

## Decisions Made
- Used 6 explicit base service names as the discriminator: if the slug is not in the list and the compose file exists, it IS the installed service and should exit cleanly.
- Placed the check at the `test -f` branch point (existing decision site D-13) to avoid altering any downstream step logic.

## Deviations from Plan

None — plan executed exactly as written.

## Issues Encountered

None.

## User Setup Required

None — no external service configuration required.

## Next Phase Readiness
- MERGE-04 is resolved; Phase 7 is complete.
- The skill is now fully idempotent for alias slug re-runs.
- No blockers for future phases.

## Known Stubs

None — the change is purely instructional text with no stubs or placeholders.

## Threat Flags

None — SKILL.md is an instruction document for AI runtimes; no executable code paths, credentials, or security surfaces affected.

---
*Phase: 07-fix-merge-04-idempotency-ux*
*Completed: 2026-04-21*
