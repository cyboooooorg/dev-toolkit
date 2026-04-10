---
status: all_fixed
phase: 01-templates-credential-foundation
fix_scope: critical_warning
findings_in_scope: 3
fixed: 3
skipped: 0
iteration: 1
completed: 2026-04-10T12:53:18Z
---

# Phase 01 Code Review Fix Report

**Fixed at:** 2026-04-10T12:53:18Z
**Source review:** `.planning/phases/01-templates-credential-foundation/01-REVIEW.md`
**Iteration:** 1

**Summary:**
- Findings in scope: 3
- Fixed: 3
- Skipped: 0

## Fixes Applied

### WR-01 — Mongo Express Basic Auth

**Files modified:** `compose-templates/mongodb/mongodb.compose.yml`, `compose-templates/.env.example`
**Commit:** `ab95b1f`
**Applied fix:** Changed `ME_CONFIG_BASICAUTH: "false"` to `"true"` in the `mongo_express` service environment block. Added `ME_CONFIG_BASICAUTH_USERNAME: ${MONGO_EXPRESS_USER:-mexpress}` and `ME_CONFIG_BASICAUTH_PASSWORD: ${MONGO_EXPRESS_PASSWORD:-mexpress}` to surface dedicated UI credentials. Added `MONGO_EXPRESS_USER` and `MONGO_EXPRESS_PASSWORD` entries (with `{{PLACEHOLDER}}` values) to the MongoDB section of `.env.example`.

### WR-02 — Root `down` Task Missing Monitoring Teardown

**Files modified:** `taskfile-templates/root/Taskfile.yml`
**Commit:** `c8e9eb2`
**Applied fix:** Added `- task: monitoring:down` with `ignore_error: true` as the final entry in the `down` task's `cmds` list, consistent with the pattern used for all other service teardown calls. Prometheus and Grafana are now stopped when `task down` is run.

### WR-03 — Grafana Default Admin Password

**Files modified:** `compose-templates/monitoring/monitoring.compose.yml`, `compose-templates/.env.example`
**Commit:** `364592e`
**Applied fix:** Added an `environment:` block to the `grafana` service with `GF_SECURITY_ADMIN_USER: ${GRAFANA_ADMIN_USER:-admin}` and `GF_SECURITY_ADMIN_PASSWORD: ${GRAFANA_ADMIN_PASSWORD}`. Added a new `# ── Monitoring ──` section to `.env.example` with `GRAFANA_ADMIN_USER` and `GRAFANA_ADMIN_PASSWORD` entries using `{{PLACEHOLDER}}` values, consistent with the rest of the credential substitution pattern in the file.

## Skipped

None.

---

_Fixed: 2026-04-10T12:53:18Z_
_Fixer: gsd-code-fixer (agent)_
_Iteration: 1_
