---
phase: 01-templates-credential-foundation
reviewed: 2026-04-10T12:32:00Z
depth: standard
files_reviewed: 14
files_reviewed_list:
  - compose-templates/redis/redis.compose.yml
  - compose-templates/rabbitmq/rabbitmq.compose.yml
  - compose-templates/postgres/postgres.compose.yml
  - compose-templates/mysql/mysql.compose.yml
  - compose-templates/mongodb/mongodb.compose.yml
  - compose-templates/monitoring/monitoring.compose.yml
  - compose-templates/.env.example
  - taskfile-templates/redis/redis.Taskfile.yml
  - taskfile-templates/rabbitmq/rabbitmq.Taskfile.yml
  - taskfile-templates/postgres/postgres.Taskfile.yml
  - taskfile-templates/mysql/mysql.Taskfile.yml
  - taskfile-templates/mongodb/mongodb.Taskfile.yml
  - taskfile-templates/monitoring/monitoring.Taskfile.yml
  - taskfile-templates/root/Taskfile.yml
findings:
  critical: 0
  warning: 3
  info: 2
  total: 5
status: issues_found
---

# Phase 01: Code Review Report

**Reviewed:** 2026-04-10T12:32:00Z
**Depth:** standard
**Files Reviewed:** 14
**Status:** issues_found

## Summary

Reviewed all 6 compose fragment templates, the shared `.env.example`, and all 7 Taskfile templates produced in Phase 1.

**The foundation is solid.** All compose files correctly omit the deprecated `version:` key, all host ports are bound to `127.0.0.1`, no credentials are hardcoded, no `external: true` network declarations exist, and volume key names are all plain strings with no invalid `${VAR}_name:` patterns. All Taskfiles use `version: '3'`, `docker compose` (v2 space syntax), and `{{.TASKFILE_DIR}}` references correctly.

**Three issues need attention before Phase 2:**
1. Mongo Express ships with auth disabled — a security default that should be reversed
2. The root `task down` doesn't stop the monitoring stack, leaving Prometheus/Grafana dangling after `task up-monitoring`
3. Grafana defaults to `admin/admin` with no override — a well-known insecure default even for localhost tools

Two informational notes round out the findings.

---

## Warnings

### WR-01: Mongo Express authentication disabled (`ME_CONFIG_BASICAUTH: "false"`)

**File:** `compose-templates/mongodb/mongodb.compose.yml:45`
**Issue:** `ME_CONFIG_BASICAUTH` is set to `"false"`, disabling the Mongo Express web UI's built-in HTTP basic auth. Any process running on localhost — or any tool/script with access to the loopback interface — can browse, query, and mutate the full MongoDB database through the UI with zero authentication. This is especially risky in shared dev machines, CI environments, or whenever port-forwarding is active.
**Fix:** Enable basic auth and supply credentials from the env file:
```yaml
# mongodb.compose.yml — mongo_express service environment block
ME_CONFIG_BASICAUTH: "true"
ME_CONFIG_BASICAUTH_USERNAME: ${MONGO_INITDB_ROOT_USERNAME}
ME_CONFIG_BASICAUTH_PASSWORD: ${MONGO_INITDB_ROOT_PASSWORD}
```
Also add `MONGO_EXPRESS_USERNAME` / `MONGO_EXPRESS_PASSWORD` entries to `.env.example` if you prefer separate UI credentials, or document the re-use of the root credentials in comments.

---

### WR-02: Root `task down` does not stop the monitoring stack

**File:** `taskfile-templates/root/Taskfile.yml:48-60`
**Issue:** The `up-monitoring` task (line 76-90) starts six service exporters **and** Prometheus + Grafana via `task: monitoring:up`. However, the `down` task (lines 48-60) only stops the six service containers — it never calls `task: monitoring:down`. After running `task up-monitoring`, a user running `task down` is left with dangling Prometheus and Grafana containers consuming memory and holding the monitoring network open. They would have to discover and run `task monitoring:down` manually.
**Fix:** Add `monitoring:down` to the `down` task, consistent with how `monitoring:up` is called from `up-monitoring`:
```yaml
  down:
    desc: Stop all installed services
    cmds:
      - task: redis:down
        ignore_error: true
      - task: rabbitmq:down
        ignore_error: true
      - task: postgres:down
        ignore_error: true
      - task: mysql:down
        ignore_error: true
      - task: mongodb:down
        ignore_error: true
      - task: monitoring:down   # ← add this
        ignore_error: true
```

---

### WR-03: Grafana defaults to `admin`/`admin` — no override supplied

**File:** `compose-templates/monitoring/monitoring.compose.yml:23-32`
**Issue:** The `grafana` service sets no `GF_SECURITY_ADMIN_PASSWORD` (or `GF_SECURITY_ADMIN_USER`) environment variable. Grafana's upstream image defaults to `admin`/`admin`, a universally known credential pair. Even though the port is bound to `127.0.0.1:3001`, this credential is trivially guessable by any local process or script. It also trains users to accept `admin/admin` as a valid dev workflow, which bleeds into habits.
**Fix:** Wire a configurable password from the env file. Add a new variable to `.env.example`:
```dotenv
# ── Monitoring ──────────────────────────────────────────────────────────────────
GRAFANA_ADMIN_PASSWORD={{PASSWORD}}
```
Then surface it in the grafana service:
```yaml
  grafana:
    image: grafana/grafana:latest
    environment:
      GF_SECURITY_ADMIN_PASSWORD: ${GRAFANA_ADMIN_PASSWORD}
      GF_SECURITY_ADMIN_USER: admin
```
This integrates cleanly with the rest of the `{{TOKEN}}`-substitution flow.

---

## Info

### IN-01: `PGADMIN_DEFAULT_EMAIL` is hardcoded in `.env.example`, breaking token consistency

**File:** `compose-templates/.env.example:46`
**Issue:** Every other credential in `.env.example` uses a `{{TOKEN}}` placeholder (e.g., `{{PASSWORD}}`, `{{USERNAME}}`). The pgAdmin email is the only value hardcoded as a literal string: `PGADMIN_DEFAULT_EMAIL=admin@local.dev`. Users who want to log in to pgAdmin with a different email must manually edit the installed `.devtools/.env` rather than having it substituted at install time.
**Fix:** Replace with a placeholder consistent with the rest of the file:
```dotenv
PGADMIN_DEFAULT_EMAIL={{EMAIL}}
```
The skill's installer can then either prompt for the value or default-substitute `admin@local.dev` as a pre-filled suggestion.

---

### IN-02: UI tools and exporters use `:latest` while base service images are version-pinned

**Files:** All compose templates (see list below)
**Issue:** Primary service images are correctly pinned to a major version (e.g., `redis:${REDIS_VERSION}-alpine`, `postgres:${POSTGRES_VERSION}-alpine`). However, all companion images use `:latest`, meaning a `docker pull` after an upstream release can silently upgrade them and change behavior:

| Image | File | Line |
|---|---|---|
| `redis/redisinsight:latest` | redis.compose.yml | 32 |
| `oliver006/redis_exporter:latest` | redis.compose.yml | 44 |
| `kbudde/rabbitmq-exporter:latest` | rabbitmq.compose.yml | 36 |
| `dpage/pgadmin4:latest` | postgres.compose.yml | 35 |
| `prometheuscommunity/postgres-exporter:latest` | postgres.compose.yml | 50 |
| `phpmyadmin:latest` | mysql.compose.yml | 45 |
| `prom/mysqld-exporter:latest` | mysql.compose.yml | 60 |
| `mongo-express:latest` | mongodb.compose.yml | 35 |
| `percona/mongodb_exporter:latest` | mongodb.compose.yml | 52 |
| `prom/prometheus:latest` | monitoring.compose.yml | 11 |
| `grafana/grafana:latest` | monitoring.compose.yml | 23 |

**Fix:** Pin each companion image to a stable major or minor tag (e.g., `dpage/pgadmin4:8`, `prom/prometheus:v3`, `grafana/grafana:11`). For tools that change infrequently (pgAdmin, phpMyAdmin), a major-version pin is sufficient and easy to maintain. Alternatively, add corresponding `*_UI_VERSION` / `*_EXPORTER_VERSION` vars to `.env.example` so users can control pins explicitly, consistent with the version-pin pattern used for primary services.

---

_Reviewed: 2026-04-10T12:32:00Z_
_Reviewer: gsd-code-reviewer (agent)_
_Depth: standard_
