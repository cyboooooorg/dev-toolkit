---
phase: 06-fix-taskfile-integration-and-ui-port-gap
reviewed: 2025-04-16T17:30:00Z
depth: standard
files_reviewed: 7
files_reviewed_list:
  - taskfile-templates/redis/redis.Taskfile.yml
  - taskfile-templates/rabbitmq/rabbitmq.Taskfile.yml
  - taskfile-templates/postgres/postgres.Taskfile.yml
  - taskfile-templates/mysql/mysql.Taskfile.yml
  - taskfile-templates/mongodb/mongodb.Taskfile.yml
  - taskfile-templates/monitoring/monitoring.Taskfile.yml
  - skills/add-service/SKILL.md
findings:
  critical: 0
  warning: 3
  info: 2
  total: 5
status: issues_found
---

# Phase 06: Code Review Report

**Reviewed:** 2025-04-16T17:30:00Z
**Depth:** standard
**Files Reviewed:** 7
**Status:** issues_found

## Summary

Six per-service Taskfile templates and the `add-service` SKILL.md were reviewed. Cross-reading the root Taskfile template (`taskfile-templates/root/Taskfile.yml`, loaded for context) was necessary to verify the `COMPOSE_PROFILES` loading chain.

The most significant finding is a broken contract: per-service `up` task descriptions declare "requires `COMPOSE_PROFILES=services` in `.devtools/.env`", but no mechanism in the Taskfile layer actually loads that file into Docker Compose's environment. Docker Compose reads `.env` from the working directory (project root), not from `.devtools/.env`. Without `dotenv` in the Taskfile or `--env-file` / explicit `--profile services` in the commands, calling `task redis:up` silently starts **zero containers** unless the user has separately exported `COMPOSE_PROFILES=services` into their shell.

A related but distinct issue: `up-ui` and `up-monitoring` tasks only activate their companion profile (`--profile ui` / `--profile monitoring`) without also activating `services`. Their own task descriptions say they "Start Redis + RedisInsight UI" (i.e., both), but the commands only start the companion — the main service won't start if it's profile-gated and the env var isn't loaded.

The monitoring Taskfile is actually **correct** on this axis: it explicitly passes `--profile monitoring` to `up`, making it self-contained. The per-service templates should follow the same pattern.

One minor description inconsistency (RabbitMQ says `.env`; others say `.devtools/.env`) and two quality/uniformity notes round out the findings.

---

## Warnings

### WR-01: `COMPOSE_PROFILES=services` is never loaded — per-service `up` starts no containers

**Files:**
- `taskfile-templates/redis/redis.Taskfile.yml:12-14`
- `taskfile-templates/postgres/postgres.Taskfile.yml:11-13`
- `taskfile-templates/mysql/mysql.Taskfile.yml:12-14`
- `taskfile-templates/mongodb/mongodb.Taskfile.yml:11-13`
- `taskfile-templates/rabbitmq/rabbitmq.Taskfile.yml:13-15`

**Issue:**
All five per-service `up` tasks call `docker compose ... up -d` without passing `--profile services` and without loading `.devtools/.env`. Docker Compose resolves `.env` from the **current working directory** (the project root), not from `.devtools/.env`. The root Taskfile template has no `dotenv:` directive. As a result, `COMPOSE_PROFILES=services` written to `.devtools/.env` by the skill is never picked up by `docker compose`. Services whose compose definitions have `profiles: [services]` will not start — the command exits successfully but creates no containers.

The monitoring Taskfile correctly avoids this problem by passing `--profile monitoring` explicitly on its `up` task (line 14). The same pattern should be applied to service `up` tasks.

**Fix (option A — explicit flag, most robust):**
```yaml
# redis.Taskfile.yml — same change needed in rabbitmq, postgres, mysql, mongodb
up:
  desc: Start Redis
  cmds:
    - docker compose -f .devtools/redis/redis.compose.yml --profile services up -d --remove-orphans
```

**Fix (option B — env-file, keeps profiles in one place):**
```yaml
up:
  desc: Start Redis
  cmds:
    - docker compose --env-file .devtools/.env -f .devtools/redis/redis.compose.yml up -d --remove-orphans
```

---

### WR-02: `up-ui` and `up-monitoring` tasks start only the companion, not the main service

**Files:**
- `taskfile-templates/redis/redis.Taskfile.yml:16-19, 21-24`
- `taskfile-templates/postgres/postgres.Taskfile.yml:15-18, 20-23`
- `taskfile-templates/mysql/mysql.Taskfile.yml:16-19, 21-24`
- `taskfile-templates/mongodb/mongodb.Taskfile.yml:15-18, 20-23`

**Issue:**
`up-ui` activates only `--profile ui`; `up-monitoring` activates only `--profile monitoring`. If the user calls `task redis:up-ui` without having previously called `task redis:up`, and without `COMPOSE_PROFILES=services` in their environment (see WR-01), RedisInsight starts but Redis itself does not. The task description `"Start Redis + RedisInsight UI"` is incorrect — only the UI companion starts. This produces a confusing failure mode: RedisInsight launches but immediately fails to connect.

**Fix:** Activate both the `services` profile and the companion profile together:
```yaml
# redis.Taskfile.yml
up-ui:
  desc: Start Redis + RedisInsight UI
  cmds:
    - docker compose -f .devtools/redis/redis.compose.yml --profile services --profile ui up -d --remove-orphans

up-monitoring:
  desc: Start Redis + metrics exporter
  cmds:
    - docker compose -f .devtools/redis/redis.compose.yml --profile services --profile monitoring up -d --remove-orphans
```
Apply the same pattern to postgres, mysql, mongodb. (RabbitMQ's management UI is always-on — its `up-monitoring` should similarly add `--profile services`.)

---

### WR-03: RabbitMQ `up` desc says `.env`, others say `.devtools/.env` — inconsistent and misleading

**File:** `taskfile-templates/rabbitmq/rabbitmq.Taskfile.yml:13`

**Issue:**
```yaml
# rabbitmq.Taskfile.yml, line 13
desc: Start RabbitMQ with management UI on RABBITMQ_UI_PORT (requires COMPOSE_PROFILES=services in .env)
#                                                                                                  ^^^^^
# All other services say: "requires COMPOSE_PROFILES=services in .devtools/.env"
```
RabbitMQ's `up` description references `.env` (project root) while every other service references `.devtools/.env`. The SKILL.md explicitly writes `COMPOSE_PROFILES` to `.devtools/.env`, not to the project root `.env`. This creates confusion about where the value must be set — and points to the root issue (WR-01) where neither location is actually loaded automatically.

**Fix:**
```yaml
desc: Start RabbitMQ with management UI on RABBITMQ_UI_PORT (requires COMPOSE_PROFILES=services in .devtools/.env)
```
Then address WR-01 so the description matches the actual loading mechanism.

---

## Info

### IN-01: RabbitMQ `up-ui` duplicates `up` verbatim — DRY violation

**File:** `taskfile-templates/rabbitmq/rabbitmq.Taskfile.yml:17-20`

**Issue:**
```yaml
up-ui:
  desc: Start RabbitMQ (management UI is bundled — this is equivalent to up)
  cmds:
    - docker compose -f .devtools/rabbitmq/rabbitmq.compose.yml up -d --remove-orphans  # identical to up
```
`up-ui` is identical to `up` in both description and command. Any future change to `up` must be duplicated in `up-ui`. Taskfile supports task delegation natively.

**Fix:**
```yaml
up-ui:
  desc: Start RabbitMQ (management UI is always-on — equivalent to up)
  cmds:
    - task: up
```

---

### IN-02: `monitoring.Taskfile.yml` missing `up-ui` task — breaks uniform API contract

**File:** `taskfile-templates/monitoring/monitoring.Taskfile.yml`

**Issue:**
All five service Taskfiles expose `up`, `up-ui`, `up-monitoring`, `down`, `logs`, `restart`. The monitoring Taskfile exposes `up`, `down`, `logs`, `logs-grafana`, `restart` — no `up-ui`. Grafana IS the UI for monitoring, so `up` already brings it up, but the absence of `up-ui` breaks the structural consistency that users and scripts may rely on when treating all services uniformly (`task <service>:up-ui`).

**Fix (minimal — alias for discoverability):**
```yaml
up-ui:
  desc: Start Prometheus + Grafana (Grafana is the UI — same as up)
  cmds:
    - task: up
```

---

_Reviewed: 2025-04-16T17:30:00Z_
_Reviewer: gsd-code-reviewer_
_Depth: standard_
