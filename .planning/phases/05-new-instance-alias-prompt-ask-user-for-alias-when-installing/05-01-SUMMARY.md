---
phase: 05-new-instance-alias-prompt-ask-user-for-alias-when-installing
plan: "01"
subsystem: skills/add-service
tags: [skill, alias, ux, validation, normalization]
dependency_graph:
  requires: []
  provides:
    - "Updated alias prompt UX in Step 1 (proactive optional alias) and Step 1a Branch B (full validation loop)"
  affects:
    - skills/add-service/SKILL.md
tech_stack:
  added: []
  patterns:
    - "Prose-step validation loop with cancel escape, silent normalization, empty re-prompt, service-name rejection, unlimited conflict re-prompt"
key_files:
  created: []
  modified:
    - skills/add-service/SKILL.md
decisions:
  - "Branch B now re-prompts on alias conflict (D-09) instead of stopping — unlimited loop with cancel escape"
  - "Normalization is silent (D-04, D-05) — no announcement; result visible in done summary slug"
  - "Empty-slug: re-prompt once then exit (D-06) — empty counter resets on non-empty input"
  - "Alias equals service name → reject and re-prompt indefinitely (D-07)"
  - "Proactive first-install alias prompt placed before Step 2 (D-13) — slug locked before Q&A"
  - "No conflict check on first install (D-13) — slot confirmed free by test -f above"
metrics:
  duration: "~15 minutes"
  completed: "2026-04-16T13:44:00Z"
  tasks_completed: 2
  files_modified: 1
---

# Phase 05 Plan 01: Alias Prompt UX (D-01–D-13) Summary

## One-Liner

Full alias validation loop (silent normalization, empty/same-name/conflict re-prompts) added to Branch B of Step 1a, plus optional proactive alias prompt inserted in Step 1's free-slot path before Step 2.

## What Was Built

Updated `skills/add-service/SKILL.md` at two insertion points:

### Task 1 — Branch B of Step 1a: Full Alias Validation Loop (D-01–D-09)

Replaced the old bare `"Add another instance with an alias, or cancel? [alias/cancel]"` prompt with a structured validation loop:

1. **Cancel escape:** `cancel` → output `"Nothing written. Exiting."` and stop.
2. **Silent normalization (D-04, D-05):** lowercase, replace non-`[a-z0-9-]` with hyphens, collapse consecutive hyphens, trim leading/trailing hyphens. No announcement.
3. **Empty-slug check (D-06):** Re-prompt once with `"Alias cannot be empty. Enter alias (e.g. 'cache' → ${SERVICE}-cache): _"`. Second consecutive empty → exit.
4. **Service-name rejection (D-07):** If alias equals SERVICE → re-prompt with `"Alias must differ from the service name. Enter alias (e.g. 'cache' → ${SERVICE}-cache): _"`.
5. **Slug derivation:** ALIAS, SERVICE_SLUG, SERVICE_SNAKE, ENV_PREFIX.
6. **Conflict re-prompt (D-08, D-09):** Unlimited loop — `"${SERVICE_SLUG} is already installed. Enter a different alias (or 'cancel'): _"`.
7. **Free slot:** set `MODE=alias`, continue to **Step 2**.

### Task 2 — Step 1 Free-Slot Path: Proactive Optional Alias Prompt (D-10–D-13)

Replaced the bare `- If the file **does not exist** → continue to **Step 2**.` with an optional alias prompt before proceeding to Step 2:

- **Prompt:** `"Install with a custom alias? (optional — press Enter to install as '${SERVICE}'): _"`
- **Press Enter (D-11):** Set `MODE=standard`, continue to **Step 2**.
- **Provide alias (D-12):** Apply same normalization/validation as Branch B (empty-slug + service-name checks). On valid alias: set MODE=alias, continue to **Step 2**.
- **No conflict check (D-13):** Slot was just confirmed free by the `test -f` above.

## Commits

| Task | Commit | Description |
|------|--------|-------------|
| 1 | `916a7c0` | feat(05-01): rewrite Step 1a Branch B with full alias validation loop (D-01–D-09) |
| 2 | `9c4e625` | feat(05-01): insert proactive optional alias prompt in Step 1 free-slot path (D-10–D-13) |

## Deviations from Plan

### Auto-fixed Issues

**1. [Rule 1 - Acceptance Criterion Inconsistency] Uppercase "Continue" → lowercase "continue"**
- **Found during:** Task 2 verification
- **Issue:** Plan's verbatim replacement text used "Continue to **Step 2**" (capital C) but acceptance criterion grep checked for lowercase "continue to **Step 2**"
- **Fix:** Changed to lowercase "continue" to satisfy acceptance criteria and maintain consistency with Branch B text
- **Files modified:** skills/add-service/SKILL.md

**2. [Note] Task 2 criterion 7 false positive**
- The acceptance criterion `grep -c "does not exist.*continue to \*\*Step 2\*\*"` → 0 returns 1 because the new Branch B line (`- If file **does not exist**: set 'MODE=alias' and continue to **Step 2**.`) also matches this pattern. The original bare Step 1 line is fully replaced — intent is met. The Branch B line is the new text introduced by Task 1, not the old bare line.

## Threat Surface Scan

No new network endpoints, auth paths, file access patterns, or schema changes introduced. This plan modifies only static Markdown prose instructions (`SKILL.md`). Threat model accepted all threats as `accept` — no mitigations required.

## Known Stubs

None — all alias prompt UX decisions (D-01 through D-13) are fully implemented in prose.

## Self-Check: PASSED

Files exist:
- `skills/add-service/SKILL.md` ✅ (783 lines)
- `.planning/phases/05-new-instance-alias-prompt-ask-user-for-alias-when-installing/05-01-SUMMARY.md` ✅

Commits exist:
- `916a7c0` ✅
- `9c4e625` ✅

All 13 step headings (Step 1 through Step 13) intact ✅
Old bare prompt removed ✅
All D-01–D-13 implemented ✅
