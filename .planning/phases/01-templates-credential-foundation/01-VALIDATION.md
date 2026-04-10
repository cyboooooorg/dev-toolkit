---
phase: 1
slug: templates-credential-foundation
status: draft
nyquist_compliant: false
wave_0_complete: false
created: 2026-04-10
---

# Phase 1 — Validation Strategy

> Per-phase validation contract for feedback sampling during execution.

---

## Test Infrastructure

| Property | Value |
|----------|-------|
| **Framework** | Shell validation — static YAML/JSON files, no test runner needed |
| **Config file** | None — run commands directly |
| **Quick run command** | `docker compose -f <file> config --quiet && echo PASS` |
| **Full suite command** | See wave checklist below |
| **Estimated runtime** | ~10 seconds |

---

## Sampling Rate

- **After every task commit:** Run `docker compose -f <created_file> config --quiet && echo PASS`
- **After every plan wave:** Run full grep sweep + all `config --quiet` validations
- **Before `/gsd-verify-work`:** All 6 service compose files pass `config --quiet`; all Taskfiles pass `--list-all`
- **Max feedback latency:** ~10 seconds

---

## Per-Task Verification Map

| Task ID | Plan | Wave | Requirement | Threat Ref | Secure Behavior | Test Type | Automated Command | File Exists | Status |
|---------|------|------|-------------|------------|-----------------|-----------|-------------------|-------------|--------|
| Redis compose | 01-01 | 1 | TMPL-01 | — | No hardcoded passwords; env_file used | lint | `docker compose -f compose-templates/redis/redis.compose.yml config --quiet` | ❌ W0 | ⬜ pending |
| RabbitMQ compose | 01-01 | 1 | TMPL-02 | — | No hardcoded passwords; env_file used | lint | `docker compose -f compose-templates/rabbitmq/rabbitmq.compose.yml config --quiet` | ❌ W0 | ⬜ pending |
| PostgreSQL compose | 01-01 | 1 | TMPL-03 | — | No hardcoded passwords; env_file used | lint | `docker compose -f compose-templates/postgres/postgres.compose.yml config --quiet` | ❌ W0 | ⬜ pending |
| MySQL compose | 01-01 | 1 | TMPL-04 | — | No hardcoded passwords; env_file used | lint | `docker compose -f compose-templates/mysql/mysql.compose.yml config --quiet` | ❌ W0 | ⬜ pending |
| MongoDB compose | 01-01 | 1 | TMPL-05 | — | No hardcoded passwords; env_file used | lint | `docker compose -f compose-templates/mongodb/mongodb.compose.yml config --quiet` | ❌ W0 | ⬜ pending |
| Monitoring compose | 01-01 | 1 | TMPL-02 | — | No secrets in monitoring config | lint | `docker compose -f compose-templates/monitoring/monitoring.compose.yml config --quiet` | ❌ W0 | ⬜ pending |
| Per-service Taskfiles | 01-02 | 2 | TMPL-06 | — | N/A | lint | `task --taskfile taskfile-templates/<service>/<service>.Taskfile.yml --list-all` | ❌ W0 | ⬜ pending |
| Root Taskfile | 01-02 | 2 | TMPL-07 | — | N/A | lint | `task --taskfile taskfile-templates/root/Taskfile.yml --list-all` | ❌ W0 | ⬜ pending |
| No version: key | 01-01 | 1 | TMPL-01–05 | — | N/A | static analysis | `grep -rn "^version:" compose-templates/ --include="*.yml"` (must return 0) | ❌ W0 | ⬜ pending |
| No hardcoded passwords | 01-03 | 3 | CRED-01 | T-CRED-01 | Credentials never hardcoded in YAML | static analysis | `grep -rn "password:" compose-templates/ --include="*.yml"` (0 literal values) | ❌ W0 | ⬜ pending |
| env_file present | 01-03 | 3 | CRED-02 | T-CRED-01 | All services reference .env | static analysis | `grep -rL "env_file" compose-templates/**/*.yml` (must return empty) | ❌ W0 | ⬜ pending |
| .env token namespacing | 01-03 | 3 | CRED-03 | — | Env var names are namespaced per service | manual review | Inspect metadata.json env_var names for REDIS_, POSTGRES_, etc. prefixes | ❌ W0 | ⬜ pending |

*Status: ⬜ pending · ✅ green · ❌ red · ⚠️ flaky*

---

## Wave 0 Requirements

- [ ] All `compose-templates/<service>/<service>.compose.yml` files — Plan 01-01 creates these
- [ ] All `taskfile-templates/<service>/<service>.Taskfile.yml` files — Plan 01-02 creates these
- [ ] All `metadata.json` files per service — Plan 01-01 creates these
- [ ] `compose-templates/monitoring/monitoring.compose.yml` — Plan 01-01 creates this
- [ ] `taskfile-templates/monitoring/monitoring.Taskfile.yml` — Plan 01-02 creates this
- [ ] `.env.example` template — Plan 01-03 creates this

*(This entire phase is Wave 0 — creating static template files from scratch)*

---

## Manual-Only Verifications

| Behavior | Requirement | Why Manual | Test Instructions |
|----------|-------------|------------|-------------------|
| `.env` template uses namespaced var names | CRED-03 | No executable assertion for naming convention | Inspect `metadata.json` for each service — confirm `env_var` fields use `<SERVICE>_` prefix (e.g., `REDIS_PORT`, `POSTGRES_PASSWORD`) |
| ARM64 warning comment in mysql compose | TMPL-04 | Comment presence is a content check, not syntax | `grep -n "ARM" compose-templates/mysql/mysql.compose.yml` — must find comment above mysql:8 image line |
| UI companion in `ui` profile only | TMPL-01–05 | Profile assignment not validated by `config --quiet` | `grep -A2 "profiles:" compose-templates/*/ui*.compose.yml` — confirm `- ui` profile |

---

## Validation Sign-Off

- [ ] All tasks have `<automated>` verify or Wave 0 dependencies
- [ ] Sampling continuity: no 3 consecutive tasks without automated verify
- [ ] Wave 0 covers all MISSING references
- [ ] No watch-mode flags
- [ ] Feedback latency < 10s
- [ ] `nyquist_compliant: true` set in frontmatter

**Approval:** pending
