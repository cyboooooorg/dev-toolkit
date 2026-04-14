# Phase 03 — Human UAT Items

These items require a live environment and cannot be verified by file inspection.

## UAT-03-01: Skill trigger via natural language

**Test:** In a project where `npx skills add Cyboooooorg/dev-tools` has been run, say:

> "add Redis to this project"

**Expected:** The `add-service` skill activates and begins the interactive Q&A flow (port, image version, credentials).

**Agents to test:** GitHub Copilot CLI, Claude Code (if available)

---

## UAT-03-02: README 10-second comprehension

**Test:** Show `README.md` to a developer unfamiliar with this repo.

**Expected:** Within 10 seconds they understand:

1. What this repo is
2. How to install the skill (`npx skills add Cyboooooorg/dev-tools`)
3. Which services are supported

---

Status: pending human runtime testing
