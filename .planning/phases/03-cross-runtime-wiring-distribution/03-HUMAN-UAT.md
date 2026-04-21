# Phase 03 — Human UAT Items

These items require a live environment and cannot be verified by file inspection.

---
status: complete
phase: 03-cross-runtime-wiring-distribution
started: 2026-04-21T12:19:00Z
updated: 2026-04-21T12:25:41Z
---

## Current Test

[testing complete]

## UAT-03-01: Skill trigger via natural language

**Test:** In a project where `npx skills add Cyboooooorg/dev-tools` has been run, say:

> "add Redis to this project"

**Expected:** The `add-service` skill activates and begins the interactive Q&A flow (port, image version, credentials).

**Agents to test:** GitHub Copilot CLI, Claude Code (if available)

result: pass

---

## UAT-03-02: README 10-second comprehension

**Test:** Show `README.md` to a developer unfamiliar with this repo.

**Expected:** Within 10 seconds they understand:

1. What this repo is
2. How to install the skill (`npx skills add Cyboooooorg/dev-tools`)
3. Which services are supported

result: pass

---

## Summary

total: 2
passed: 2
issues: 0
pending: 0
skipped: 0
blocked: 0

## Gaps

[none]
