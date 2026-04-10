# Phase 1 Context: Templates & Credential Foundation

## Phase Goal

Create all static template files (Compose + Taskfile) for every supported service, plus the `.env` credential strategy. These templates are the product — the skill reads and deploys them with user-supplied values substituted in.

---

## Locked Decisions

### File Naming & Structure

- Per-service Compose fragment: `compose-templates/<service>/<service>.compose.yml`
- Per-service Taskfile: `taskfile-templates/<service>/<service>.Taskfile.yml`
- Monitoring Compose templates: `compose-templates/monitoring/monitoring.compose.yml`
- Monitoring Taskfile: `taskfile-templates/monitoring/monitoring.Taskfile.yml`
- Root compose file written to user project: `.devtools/compose.yml` (uses Docker Compose `include:`)
- Root Taskfile written to user project: `.devtools/Taskfile.yml` (uses `includes:` with `optional: true`)

### Docker Compose Profiles

Every service Compose fragment defines containers under one of these profiles:

| Profile | Purpose | Default |
|---|---|---|
| `services` | Main service containers (Redis, Postgres, etc.) | ✅ Yes (starts without `--profile`) |
| `ui` | Management UI companions (opt-in) | ❌ No |
| `monitoring` | Metrics exporters per-service | ❌ No |

Monitoring profile containers (Grafana, Prometheus) live in `compose-templates/monitoring/`.

### UI Companions (profile: ui)

| Service | UI Companion |
|---|---|
| Redis | RedisInsight |
| MySQL / MariaDB | phpMyAdmin |
| PostgreSQL | pgAdmin |
| MongoDB | Mongo Express |
| RabbitMQ | RabbitMQ Management UI (already bundled) |

### Volume Naming

- Standard single instance: `${COMPOSE_PROJECT_NAME}_<service>_data`
- Multi-instance (aliased): `${COMPOSE_PROJECT_NAME}_<alias>_data`
- UI companions: **ephemeral only** (no named volumes)
- Monitoring containers (Grafana, Prometheus): **ephemeral only** (no named volumes)

### Docker Networks (Three-Tier)

Every service participates in up to three networks:

| Network | Name Pattern | Purpose |
|---|---|---|
| Per-service | `${COMPOSE_PROJECT_NAME}_<service>` | Service + its UI companion only |
| App-access | `${COMPOSE_PROJECT_NAME}_services` | User's app connects here; declared `external: true` in user's compose |
| Monitoring | `${COMPOSE_PROJECT_NAME}_monitoring` | Grafana + Prometheus + per-service exporters |

- Per-service networks are regular bridge (not `internal: true`)
- Network names are scoped to `COMPOSE_PROJECT_NAME`
- When monitoring profile is enabled, each service gets a **metrics exporter sidecar** that joins the monitoring network

### Template Placeholder Tokens

Template files use `{{PLACEHOLDER}}` syntax. Tokens are substituted at install time by the skill. Examples:

```
{{PORT}}         — published host port
{{VERSION}}      — image tag / version
{{PASSWORD}}     — credential (written to .env, not inline)
{{DB_NAME}}      — database name
{{USERNAME}}     — service username
{{ALIAS}}        — service alias for multi-instance (e.g., cache, session)
{{PROJECT_NAME}} — COMPOSE_PROJECT_NAME value
```

### Service Metadata

Each template service directory contains a `metadata.json` describing:

- Service name and description
- Configurable parameters (name, description, default, env var name, token)
- UI companion details (image, port)
- Available metrics exporter image

Example structure:
```json
{
  "service": "redis",
  "parameters": [
    { "name": "port", "default": 6379, "env_var": "REDIS_PORT", "token": "{{PORT}}" },
    { "name": "version", "default": "7", "env_var": "REDIS_VERSION", "token": "{{VERSION}}" }
  ],
  "ui_companion": { "image": "redislabs/redisinsight:latest", "port": 8001 },
  "exporter": { "image": "oliver006/redis_exporter:latest" }
}
```

### MySQL / ARM64 Strategy

- **Default**: The skill presents `MariaDB 11` as the MySQL-compatible default (ARM-native, no emulation)
- **Opt-in MySQL**: Skill asks "MySQL or MariaDB?" — if user picks MySQL, uses `mysql:8`
- **ARM warning**: When `mysql:8` is selected, the skill emits a comment in the compose file above the image line:
  ```yaml
  # NOTE: mysql:8 may require 'platform: linux/amd64' on Apple Silicon (ARM64)
  image: mysql:8
  ```

### Credential Strategy

- All credentials written to `.devtools/.env` via `env_file:` in compose fragments
- Credentials NEVER hardcoded in any YAML template
- `.devtools/.env` is gitignored by the skill on first setup
- `{{PASSWORD}}` and similar tokens are substituted with the actual secret only in `.env`, not in compose files

### COMPOSE_PROJECT_NAME

- Skill asks for a project name on first setup
- Written to `.devtools/.env` as `COMPOSE_PROJECT_NAME=<value>`
- All volume and network names derive from this value

---

## Supported Services (Phase 1 Templates)

| Service | Compose Fragment | Taskfile | Metadata |
|---|---|---|---|
| Redis | `compose-templates/redis/` | `taskfile-templates/redis/` | `metadata.json` |
| RabbitMQ | `compose-templates/rabbitmq/` | `taskfile-templates/rabbitmq/` | `metadata.json` |
| PostgreSQL | `compose-templates/postgres/` | `taskfile-templates/postgres/` | `metadata.json` |
| MySQL / MariaDB | `compose-templates/mysql/` | `taskfile-templates/mysql/` | `metadata.json` |
| MongoDB | `compose-templates/mongodb/` | `taskfile-templates/mongodb/` | `metadata.json` |
| Monitoring | `compose-templates/monitoring/` | `taskfile-templates/monitoring/` | `metadata.json` |

---

## Out of Scope for Phase 1

- SKILL.md writing (Phase 2)
- Merge/idempotency logic (Phase 2)
- Runtime wiring / README (Phase 3)
- No dynamic file generation — Phase 1 is static template authoring only

---

## Open Questions / Deferred

- Prometheus scrape config: does it live in `compose-templates/monitoring/` or get generated per-project? → Defer to Phase 2 discussion
- Whether to ship a `.devtools/compose.yml` root template or generate it dynamically → Defer to Phase 2

---

## Key Constraints Reminder

- Docker Compose v2: **NO `version:` key**
- Taskfile v3: `includes:` with `optional: true`; `{{.TASKFILE_DIR}}` in all `-f` flags
- No runtime dependencies — this repo contains only static files + SKILL.md
- Skill NEVER modifies files outside `.devtools/`
