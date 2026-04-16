---
phase: 04-subfolder-output-structure-per-service-subdirectory-layout-i
verified: 2026-04-16T11:30:00Z
status: gaps_found
score: 17/18 decisions verified
overrides_applied: 0
gaps:
  - truth: "All compose-file write paths in Step 9a use the subfolder layout .devtools/<slug>/<slug>.compose.yml"
    status: failed
    reason: "Line 402 of SKILL.md contains a stale flat-path instruction: 'Write the modified content (not the raw template) to `.devtools/redis.compose.yml`.' inside the Redis no-password special-case block. The correct path is `.devtools/redis/redis.compose.yml`. An AI agent executing Step 9a for a passwordless Redis install would follow this instruction and write to the old flat location, not the subfolder."
    artifacts:
      - path: "skills/add-service/SKILL.md"
        issue: "Line 402: `Write the modified content (not the raw template) to \`.devtools/redis.compose.yml\`.` — flat path, not subfolder path"
    missing:
      - "Change line 402 from `.devtools/redis.compose.yml` to remove the explicit write destination (the canonical destination is defined at lines 415-420) — or update it to `.devtools/${SERVICE_SLUG}/${SERVICE_SLUG}.compose.yml` to match the rest of Step 9a."
---

# Phase 4: Subfolder Output Structure Verification Report

**Phase Goal:** Update `SKILL.md` so each installed service writes to `.devtools/<slug>/` (subfolder per instance) with per-service `.env`, updated root compose/Taskfile include paths, and a rename-before-add flow for multi-instance UX.
**Verified:** 2026-04-16T11:30:00Z
**Status:** ⚠️ GAPS FOUND (1 gap)
**Re-verification:** No — initial verification

---

## Goal Achievement

### Observable Truths

| # | Truth | Status | Evidence |
|---|-------|--------|----------|
| 1 | Each service writes to `.devtools/<slug>/` subfolder (D-01/D-02) | ✓ VERIFIED | Line 410: `mkdir -p .devtools/${SERVICE_SLUG}`; Lines 412-413 explain slug convention |
| 2 | Three files per subfolder: compose, Taskfile, .env (D-03/D-04) | ✓ VERIFIED | Lines 417-420 (compose), 438-441 (Taskfile), 448-451 (.env write target) |
| 3 | Per-service `.env` goes to subfolder, not root (D-05) | ✓ VERIFIED | Step 10a header (line 448): "Write per-service `.devtools/${SERVICE_SLUG}/.env`" |
| 4 | Root `.devtools/.env` is minimal (COMPOSE_PROJECT_NAME + COMPOSE_PROFILES only) (D-06) | ✓ VERIFIED | Step 10a-root (line 483); "project-level vars only" constraint enforced |
| 5 | Root compose include paths use `<slug>/<filename>.compose.yml` format (D-09) | ✓ VERIFIED | Lines 578, 587: `- path: <slug>/<filename>.compose.yml` |
| 6 | Root Taskfile template includes use subfolder-relative paths (D-10) | ✓ VERIFIED | `taskfile-templates/root/Taskfile.yml` lines 15/18/21/24/27/30: all 6 includes use `service/service.Taskfile.yml` |
| 7 | Detection checks subfolder path, not flat path (D-11) | ✓ VERIFIED | Line 77: `test -f .devtools/${SERVICE}/${SERVICE}.compose.yml`; Line 192: alias variant; old flat paths absent (0 matches) |
| 8 | `.gitignore` covers both `.env` and `**/.env` (D-12) | ✓ VERIFIED | Lines 215-216 in Step 2: `.env` + `**/.env` exact content specified |
| 9 | No migration of existing flat files (D-13) | ✓ VERIFIED | Line 220: "(D-13: no migration)" with explicit note |
| 10 | Frontmatter and objective updated to mention subfolder layout (D-14) | ✓ VERIFIED | Line 3: `Writes Docker Compose and Taskfile configs to .devtools/<service>/`; objective lines 10-20 consistent |
| 11 | `env_file: .env` resolution explained — no template changes needed (D-07) | ✓ VERIFIED | Lines 422-425 with explicit `(D-07)` reference |
| 12 | Root files remain at `.devtools/` root (D-08) | ✓ VERIFIED | Lines 613-616 "Root file locations (D-08)" note |
| 13 | Rename-before-add offered first when service slot occupied (D-15) | ✓ VERIFIED | Lines 88-93: "First, offer to rename the existing instance (D-15)" |
| 14 | Full 8-step rename flow implemented (D-16) — folder mv, file mv, substitution, compose update, Taskfile update, .env key rename | ✓ VERIFIED | Lines 101-174: all 8 operations present; RENAME_SLUG appears 16 times |
| 15 | Rename confirmation message present (D-17) | ✓ VERIFIED | Line 170: "Existing ${SERVICE} renamed to ${RENAME_SLUG}. Now installing new ${SERVICE} instance..." |
| 16 | Rename is atomic — all operations complete before new writes (D-18) | ✓ VERIFIED | Lines 173-174: "(D-18: all rename operations above are completed before any new files are written.)" |
| 17 | Branch B fallback: alias-or-cancel preserved when user skips rename | ✓ VERIFIED | Lines 180-195: Branch B fully intact |
| 18 | All compose write paths in Step 9a use subfolder layout — including Redis no-password case | ✗ FAILED | Line 402 contains stale flat path `.devtools/redis.compose.yml` — see Gaps section |

**Score:** 17/18 truths verified

---

## Required Artifacts

| Artifact | Expected | Status | Details |
|----------|----------|--------|---------|
| `skills/add-service/SKILL.md` | Full subfolder-layout skill definition | ✓ VERIFIED | All 6 phase commits present (ae8f2fc, 1a4d653, 8bfe402, b4d0ea0, ac43da1, 99fd2e0) |
| `taskfile-templates/root/Taskfile.yml` | Subfolder-relative Taskfile includes | ✓ VERIFIED | All 6 service includes use `service/service.Taskfile.yml` pattern |

---

## Key Link Verification

| From | To | Via | Status | Details |
|------|----|-----|--------|---------|
| Step 1 detection | subfolder path | `test -f .devtools/${SERVICE}/${SERVICE}.compose.yml` | ✓ WIRED | Line 77 |
| Step 1a alias detection | subfolder path | `test -f .devtools/${SERVICE_SLUG}/${SERVICE_SLUG}.compose.yml` | ✓ WIRED | Line 192 |
| Step 9a mkdir | subfolder creation | `mkdir -p .devtools/${SERVICE_SLUG}` | ✓ WIRED | Line 410 |
| Step 9a write | subfolder compose path | `.devtools/${SERVICE_SLUG}/<filename>.compose.yml` | ⚠️ PARTIAL | General case: correct (line 417). Redis no-password special case: STALE at line 402 (`.devtools/redis.compose.yml`) |
| Step 10a write | per-service .env | `.devtools/${SERVICE_SLUG}/.env` | ✓ WIRED | Line 451 |
| Step 11a include | subfolder compose include | `path: <slug>/<filename>.compose.yml` | ✓ WIRED | Lines 578, 587 |
| Step 2 .gitignore | `**/.env` pattern | `.gitignore` content spec | ✓ WIRED | Lines 215-216 |
| Step 1a Branch A rename | all 8 operations | mv + substitution + reference updates | ✓ WIRED | Lines 107-174 |

---

## Commit Verification

All 6 commits confirmed present in git log:

| Commit | Description | Status |
|--------|-------------|--------|
| `ae8f2fc` | feat(04-01): update frontmatter, objective, Step 1/1a detection, Step 2 .gitignore | ✓ EXISTS |
| `1a4d653` | feat(04-01): update Step 8 summary and Step 9a/9b write paths with mkdir | ✓ EXISTS |
| `8bfe402` | feat(04-02): rewrite Step 10 .env management — split per-service subfolder vs root | ✓ EXISTS |
| `b4d0ea0` | feat(04-02): update Step 11 compose include paths and root Taskfile template | ✓ EXISTS |
| `ac43da1` | feat(04-03): rewrite Step 1a with rename-before-add flow (D-15 to D-18) | ✓ EXISTS |
| `99fd2e0` | feat(04-03): update Step 13 done summary and success_criteria for subfolder layout | ✓ EXISTS |

---

## Anti-Patterns Found

| File | Line | Pattern | Severity | Impact |
|------|------|---------|----------|--------|
| `skills/add-service/SKILL.md` | 402 | Stale flat-path write instruction: `.devtools/redis.compose.yml` | 🛑 Blocker | An AI agent executing Step 9a for a no-password Redis install follows this line and writes to the old flat path, bypassing the subfolder layout established by the rest of Phase 4 |

---

## Gap Detail

### Stale Write Destination in Redis No-Password Special Case (Step 9a)

**Location:** `skills/add-service/SKILL.md`, line 402

**Current text:**
```
Write the modified content (not the raw template) to `.devtools/redis.compose.yml`.
```

**Problem:** This is the only remaining instance of a flat-path write target in the file. Every other write instruction correctly uses the subfolder pattern (`.devtools/${SERVICE_SLUG}/<filename>.compose.yml`). This line was not updated during the Phase 4 changes.

**Why it's a blocker:** An AI agent executing Step 9a in the Redis + no-password scenario reads the special-case block (lines 398-402), sees the explicit write destination `.devtools/redis.compose.yml`, and writes to the old flat location. The general write destination defined at lines 415-420 comes *after* the special-case block — the agent may treat line 402 as an earlier, more specific instruction that supersedes the later general rule.

**Fix:** Either:
1. Delete the write destination from line 402 (the write target is already covered at lines 415-420), changing it to:
   ```
   Apply these modifications to the in-memory content before writing (see write destination below).
   ```
2. Or update line 402 to use the subfolder path explicitly:
   ```
   Write the modified content (not the raw template) to `.devtools/${SERVICE_SLUG}/${SERVICE_SLUG}.compose.yml`.
   ```

**Root cause:** The Redis no-password special case was added before Phase 4 and contains an embedded write destination. When Phase 4 updated Step 9a's write paths, the special-case prose block was not updated alongside the general write target.

---

## Requirements Coverage

Phase 4 requirements per ROADMAP.md: TMPL-08, CRED-01, CRED-02, CRED-03, MERGE-01, MERGE-02, MERGE-03, MERGE-04, SKILL-03

| Requirement | Description | Status | Evidence |
|-------------|-------------|--------|----------|
| TMPL-08 | All generated files written to `.devtools/` | ✓ SATISFIED | All write instructions target `.devtools/<slug>/` |
| CRED-01 | Credentials in `.devtools/.env`, not hardcoded | ✓ SATISFIED | Step 10a writes per-service .env; no hardcoded values |
| CRED-02 | Compose uses `env_file: .env` | ✓ SATISFIED | D-07 note (lines 422-425) confirms no template changes needed |
| CRED-03 | `.env` appended (not overwritten); vars namespaced | ✓ SATISFIED | Step 10a conflict detection (lines 472-477); alias prefix in Step 10a (lines 469-471) |
| MERGE-01 | Checks for existing service before writing | ✓ SATISFIED | Step 1 detection at subfolder path (line 77) |
| MERGE-02 | Informs user and asks: rename/alias/cancel | ✓ SATISFIED | Step 1a Branch A + Branch B (lines 82-195) |
| MERGE-03 | Alias installs namespace service/volume/env vars | ✓ SATISFIED | Branch B alias setup + Step 12 substitution map |
| MERGE-04 | Identical alias re-install → "already installed, nothing to do" | ✓ SATISFIED | Lines 193-194 |
| SKILL-03 | Frontmatter enumerates all 5 supported services | ✓ SATISFIED | Line 3 frontmatter description lists all 5 services |

---

## Human Verification Required

None — all verification items are programmatically checkable. The SKILL.md is a declarative instruction document; correctness can be assessed by reading and grepping.

---

## Gaps Summary

**1 gap** found blocking full goal achievement.

17 of 18 decisions across Phase 4 are correctly implemented. The single gap is a stale write-destination prose sentence at line 402 of `skills/add-service/SKILL.md` — inside the Redis no-password special case — that still names the old flat path (`.devtools/redis.compose.yml`) instead of the subfolder path. This line was left unupdated when Phase 4 rewrote Step 9a's write logic.

All 3 major goal pillars are otherwise complete:
- ✓ Subfolder detection, mkdir, write paths — implemented throughout
- ✓ `.env` split (per-service subfolder vs minimal root) — implemented
- ✓ Rename-before-add flow (D-15 to D-18) — fully implemented

The gap is narrow (Redis + no password only) but is a live instruction that would mislead an AI agent executing the skill.

---

_Verified: 2026-04-16T11:30:00Z_
_Verifier: the agent (gsd-verifier)_
