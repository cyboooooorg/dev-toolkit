---
phase: 2
plan: 1
subsystem: skill-core
tags: [skill, SKILL.md, service-registry, merge-detection, first-run-setup]
dependency_graph:
  requires: []
  provides: [skills/add-service/SKILL.md skeleton with Steps 1–2]
  affects: [02-02, 02-03]
tech_stack:
  added: []
  patterns: [SKILL.md GSD inline-process style, service registry table, merge detection flow]
key_files:
  created:
    - skills/add-service/SKILL.md
  modified: []
decisions:
  - "SKILL.md <process> block left open intentionally — 02-02 and 02-03 append remaining steps"
  - "Template Format Note added to context section clarifying ${ENV_VAR} vs {{TOKEN}} distinction"
  - "Service registry includes monitoring as 6th entry alongside the 5 core services"
metrics:
  duration: ~10m
  completed: 2026-04-10
  tasks_completed: 2
  files_changed: 1
---

# Phase 2 Plan 1: SKILL.md Frontmatter + Service Registry Summary

## One-liner

SKILL.md skeleton created with frontmatter (6 allowed tools, all 5 services enumerated in description), service registry table for all 6 services + root Taskfile, and Steps 1–2 of the process (merge detection and first-run setup).

## What Was Built

Created `skills/add-service/SKILL.md` — the foundational skill file that all Phase 2 plans build on:

**Task 1 — Skeleton (frontmatter, objective, context):**
- Frontmatter with `name: add-service`, `description` enumerating all 5 supported services, `argument-hint: "<service-name>"`, and `allowed-tools: Read, Write, Bash, AskUserQuestion`
- `<objective>` section (5 sentences describing interactive flow, confirmation step, idempotency, and credential handling)
- `<context>` section with `$ARGUMENTS` reference, service registry table (6 services: redis, rabbitmq, postgres, mysql, mongodb, monitoring + root Taskfile path), and template format note clarifying `${ENV_VAR}` vs `{{TOKEN}}` usage
- `<process>` block opened (not closed — closed in 02-03)

**Task 2 — Steps 1, 1a, 2:**
- **Step 1 (Detect Installation State):** `SERVICE=$ARGUMENTS` normalization, existence check via `test -f .devtools/${SERVICE}.compose.yml`
- **Step 1a (Merge Detection):** Alias/cancel prompt, derivation of `SERVICE_SLUG`, `SERVICE_SNAKE`, `ENV_PREFIX` variables, alias idempotency check (MERGE-04)
- **Step 2 (First-Run Setup):** `.devtools/` directory creation, `.gitignore` with `.env` only, `COMPOSE_PROJECT_NAME` question with git-derived default, skip if already set (D-20)

## Threat Mitigations Applied

| Threat | Mitigation Applied |
|--------|-------------------|
| AI resolves template paths from CWD (D-07) | Context section explicitly states: "Resolve all paths below relative to the directory containing this SKILL.md file (the skill repo root). The target project's CWD is separate — do NOT mix these paths." |
| SKILL.md description too vague (SKILL-03) | All 5 services enumerated explicitly in the `description` frontmatter field |

## Commits

| Task | Commit | Files |
|------|--------|-------|
| Task 1: Create SKILL.md skeleton | 08f2ca8 | skills/add-service/SKILL.md (created) |
| Task 2: Append Steps 1, 1a, 2 | 98ffc03 | skills/add-service/SKILL.md (appended) |

## Deviations from Plan

None — plan executed exactly as written.

## Known Stubs

- `<process>` block is intentionally left open (no closing tag) — this is by design. Steps 3+ are appended in 02-02 and 02-03. The `</process>` closing tag is written in 02-03.

## Self-Check: PASSED

- [x] `skills/add-service/SKILL.md` exists
- [x] Frontmatter contains `name: add-service`, all 5 services in description, `allowed-tools: Read, Write, Bash, AskUserQuestion`
- [x] `<objective>` section present with 5 sentences
- [x] `<context>` section has `$ARGUMENTS` and service registry for all 6 services + root Taskfile
- [x] Service registry paths are relative to skill repo root
- [x] `<process>` block open (not closed)
- [x] Step 1 includes merge detection branch to Step 1a
- [x] Step 1a asks alias/cancel, checks idempotency, sets SERVICE_SLUG / SERVICE_SNAKE / ENV_PREFIX
- [x] Step 2 creates `.devtools/`, writes `.gitignore` with `.env` only, asks COMPOSE_PROJECT_NAME
- [x] Both tasks committed individually (08f2ca8, 98ffc03)
