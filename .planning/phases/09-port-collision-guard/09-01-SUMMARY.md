---
phase: 09-port-collision-guard
plan: "01"
subsystem: skills/add-service
tags: [port-collision, guard, skill, ux]
dependency_graph:
  requires: [08-01]
  provides: [port-collision-guard-step5]
  affects: [skills/add-service/SKILL.md]
tech_stack:
  added: []
  patterns: [conflict-detection-loop, bound-ports-scan, 3-option-escape-menu]
key_files:
  created: []
  modified:
    - skills/add-service/SKILL.md
decisions:
  - "Port Scan sub-step runs once before Q2 (not on every re-prompt) to avoid repeated file I/O (D-12)"
  - "Conflict message format: 'Port N is already used by <service> — try M?' (D-06)"
  - "3-option escape menu: Try different port / Use internal only / Cancel (D-08)"
  - "NEXT_FREE is a hint only — user may enter any port on re-prompt (D-10)"
  - "Option 2 sets ANSWERS[host_port]=false, identical downstream to Phase 8 opt-out (D-08)"
metrics:
  duration_minutes: 15
  completed_date: "2026-04-23"
  tasks_completed: 2
  tasks_total: 2
  files_modified: 1
---

# Phase 9 Plan 01: Port Collision Guard Summary

**One-liner:** Port scan sub-step + conflict-detection loop with 3-option escape menu added to Step 5 of the add-service skill, preventing silent host port collisions between installed services.

## What Was Built

### Task 1 — Port Scan Sub-step (commit `63a1acc`)

Inserted a **Port Scan sub-step** into `skills/add-service/SKILL.md` Step 5, positioned between the Q1 Yes-branch and Q2. The sub-step:

- Is gated under `ANSWERS[host_port]=true` — no scan for internal-only installs
- Silently skips if `.devtools/` does not exist (clean project, no services yet) — sets `BOUND_PORTS={}` (D-04)
- **Primary scan**: `find .devtools -name "*.compose.yml"` + grep for `127.0.0.1:` port entries; resolves `${ENV_VAR}` references via sibling `.env` files (D-01, D-03)
- **Supplemental scan**: `find .devtools -maxdepth 2 -name ".env"` + grep for `_PORT=[0-9]+` (D-02)
- Covers ALL container types (main service, UI companion, monitoring exporter) and alias installs via `.devtools/<service>-<alias>/` subdirectories (D-03, D-13)
- Populates `BOUND_PORTS` (port-number → service-slug map) once before Q2 is shown (D-12)

### Task 2 — Conflict-Detection Loop + 3-Option Escape Menu (commit `bd00f42`)

Replaced Q2's bare `Store answer as ANSWERS[port].` bullet with a full **conflict check loop**:

- **No conflict**: set `ANSWERS[port]=<value>` and proceed to Q3
- **Conflict detected**:
  - Identifies `CONFLICT_SERVICE` = `BOUND_PORTS[entered_port]` (first found if multiple) (D-07)
  - Computes `NEXT_FREE` via linear scan from `entered_port + 1` upward (D-06)
  - Displays: `Port <N> is already used by <CONFLICT_SERVICE> — try <NEXT_FREE>?`
  - Shows 3-option escape menu (D-08):
    1. **Try a different port** — re-prompt (any value; `NEXT_FREE` is hint only); no retry limit (D-09, D-10)
    2. **Use internal only (no host binding)** — sets `ANSWERS[host_port]=false`, continues to Q3 (same as Phase 8 opt-out path) (D-08)
    3. **Cancel service install** — outputs `"Cancelled. Nothing written."` and stops (D-08)

## Files Modified

| File | Change |
|------|--------|
| `skills/add-service/SKILL.md` | Added Port Scan sub-step + conflict-detection loop to Step 5 Universal Q&A (+92 lines) |

## Decisions Made

1. **Scan-before-prompt**: BOUND_PORTS populated once (not per-retry) to keep I/O minimal (D-12)
2. **Silent skip on clean project**: No message when `.devtools/` absent — conflict is impossible (D-04)
3. **Compose primary, .env supplemental**: Compose files are authoritative; .env catches edge cases (D-01, D-02)
4. **All container types scanned**: Main service + UI companion + monitoring exporter — no filtering (D-03)
5. **NEXT_FREE is hint only**: User enters freely on re-prompt; suggestion is not pre-filled (D-10)
6. **No retry limit**: Loop continues indefinitely while user selects option 1 (D-09)

## Deviations from Plan

None — plan executed exactly as written.

## Requirements Satisfied

- **PORT-03**: Port scan scope (compose files + .env files, all container types) ✅
- **PORT-04**: Conflict detection with service name + next-free-port suggestion ✅
- **PORT-05**: Re-prompt loop with 3-option escape menu, no retry limit ✅

## Threat Surface Scan

No new network endpoints, auth paths, file access patterns, or schema changes introduced. The Port Scan sub-step reads `.devtools/` files written by prior skill runs — no execution of file content, no untrusted input processed. All threats reviewed and accepted per plan's `<threat_model>`.

## Self-Check

- [x] `skills/add-service/SKILL.md` modified (confirmed: `grep BOUND_PORTS` returns 10+ matches)
- [x] Commit `63a1acc` exists (Task 1)
- [x] Commit `bd00f42` exists (Task 2)
- [x] Q3 "Image version/tag" still present at original position (line 421)
- [x] Old bare "Store answer as `ANSWERS[port]`." line is gone (0 matches)
