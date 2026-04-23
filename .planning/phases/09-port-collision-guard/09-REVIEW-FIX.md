---
phase: 09-port-collision-guard
fixed_at: 2025-07-14T00:00:00Z
review_path: .planning/phases/09-port-collision-guard/09-REVIEW.md
iteration: 1
findings_in_scope: 3
fixed: 3
skipped: 0
status: all_fixed
---

# Phase 9: Code Review Fix Report

**Fixed at:** 2025-07-14
**Source review:** `.planning/phases/09-port-collision-guard/09-REVIEW.md`
**Iteration:** 1

**Summary:**
- Findings in scope: 3 (WR-01, WR-02, WR-03 — IN-01 and IN-02 excluded per scope)
- Fixed: 3
- Skipped: 0

## Fixed Issues

### WR-02: `.devtools/` absence check inside Port Scan sub-step is unreachable dead code

**Files modified:** `skills/add-service/SKILL.md`
**Commit:** `4c4a5d6`
**Applied fix:**
Removed the dead-code step 1 block (the `test -d .devtools` check with its conditional branches). Replaced it by merging a first-install parenthetical note directly into the Primary scan header: _"(`.devtools/` is always present at this point — Step 2 above creates it on first install — but may contain no compose files yet; the scan simply returns empty results and `BOUND_PORTS={}` with no user-visible message.)"_

Renumbered the remaining sub-steps (old 2 → 1, old 3 → 2) and updated the closing sentence from "After steps 2–3" to "After steps 1–2".

---

### WR-01: Primary scan grep misses non-`127.0.0.1` port binding formats

**Files modified:** `skills/add-service/SKILL.md`
**Commit:** `43f00bc`
**Applied fix:**
Replaced the narrow grep:
```bash
grep -h "127\.0\.0\.1:"
```
with a broader ERE pattern:
```bash
grep -hE '^\s*-\s*"?([0-9a-f.:]+:)?(\$\{[A-Z_]+\}|[0-9]+):[0-9]+'
```
This matches all four relevant Docker Compose port binding formats: `127.0.0.1:HOST:CONTAINER`, `0.0.0.0:HOST:CONTAINER`, bare `HOST:CONTAINER`, and `${ENV_VAR}` host-port variants.

Replaced the `127.0.0.1:`-specific format bullets and "capture the token between `127.0.0.1:` and the second `:`" extraction description with a general **second-to-last colon-separated field** rule, illustrated with examples for all three bind-address variants.

---

### WR-03: `${ENV_VAR}` resolution failure produces a silent false negative in BOUND_PORTS

**Files modified:** `skills/add-service/SKILL.md`
**Commit:** `8d9fa2c`
**Applied fix:** Two targeted changes:

1. **Primary scan resolution paragraph**: Changed from "read the service's `.devtools/<slug>/.env` for the variable value" (no failure case) to: _"If the `.env` file is absent or does not contain the variable, **skip that port and continue** — the scan is best-effort for broken installs."_ This makes the failure behaviour explicit rather than leaving it as a silent false negative.

2. **Supplemental scan description**: Extended the purpose statement to explicitly note that step 2 also resolves `${ENV_VAR}` host-port references from step 1 that couldn't be resolved inline. Added the same skip-on-failure language: _"If the `.env` file or expected key is absent, skip — this is a best-effort scan for broken installs."_

---

## Skipped Issues

None — all in-scope findings were fixed.

---

_Fixed: 2025-07-14_
_Fixer: gsd-code-fixer (AI agent)_
_Iteration: 1_
