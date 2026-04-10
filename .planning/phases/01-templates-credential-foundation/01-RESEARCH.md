# Phase 1: Templates & Credential Foundation — Research

**Researched:** 2026-04-10
**Domain:** Docker Compose v2 template authoring, Taskfile v3, multi-service networking, credential patterns
**Confidence:** HIGH

---

<user_constraints>
## User Constraints (from CONTEXT.md)

### Locked Decisions

**File Naming & Structure**
- Per-service Compose fragment: `compose-templates/<service>/<service>.compose.yml`
- Per-service Taskfile: `taskfile-templates/<service>/<service>.Taskfile.yml`
- Monitoring Compose templates: `compose-templates/monitoring/monitoring.compose.yml`
- Monitoring Taskfile: `taskfile-templates/monitoring/monitoring.Taskfile.yml`
- Root compose file written to user project: `.devtools/compose.yml` (uses Docker Compose `include:`)
- Root Taskfile written to user project: `.devtools/Taskfile.yml` (uses `includes:` with `optional: true`)

**Docker Compose Profiles**
| Profile | Purpose | Default |
|---|---|---|
| `services` | Main service containers (Redis, Postgres, etc.) | ✅ Yes (activated via `COMPOSE_PROFILES=services` in `.env`) |
| `ui` | Management UI companions (opt-in) | ❌ No |
| `monitoring` | Metrics exporters per-service | ❌ No |

**Volume Naming**
- Standard single instance: `<service>_data` (Docker auto-prefixes with `COMPOSE_PROJECT_NAME`)
- Multi-instance (aliased): `<alias>_data`
- UI companions: **ephemeral only** (no named volumes)
- Monitoring containers (Grafana, Prometheus): **ephemeral only** (no named volumes)

**Docker Networks (Three-Tier)**
| Network | Name Pattern | Purpose |
|---|---|---|
| Per-service | `${COMPOSE_PROJECT_NAME}_<service>` | Service + its UI companion only |
| App-access | `${COMPOSE_PROJECT_NAME}_services` | User's app connects here |
| Monitoring | `${COMPOSE_PROJECT_NAME}_monitoring` | Grafana + Prometheus + exporters |

**Template Placeholder Tokens**
```
{{PORT}}, {{VERSION}}, {{PASSWORD}}, {{DB_NAME}}, {{USERNAME}}, {{ALIAS}}, {{PROJECT_NAME}}
```

**MySQL / ARM64 Strategy**
- Default: `mariadb:11` (ARM-native)
- Opt-in MySQL: `mysql:8` with ARM warning comment in compose template

**Credential Strategy**
- All credentials in `.devtools/.env` via `env_file:` — NEVER hardcoded in YAML
- `{{PASSWORD}}` token substituted only in `.env` template, not in compose YAML

**Supported Services**: Redis, RabbitMQ, PostgreSQL, MySQL/MariaDB, MongoDB

### Agent's Discretion
- Exact healthcheck command variants per service (ping vs port check)
- Prometheus scrape config pattern → Deferred to Phase 2

### Deferred Ideas (OUT OF SCOPE)
- SKILL.md writing (Phase 2)
- Merge/idempotency logic (Phase 2)
- Runtime wiring / README (Phase 3)
- Prometheus scrape config placement (Phase 2)
- Root `.devtools/compose.yml` template vs dynamic generation (Phase 2)

</user_constraints>

---

<phase_requirements>
## Phase Requirements

| ID | Description | Research Support |
|----|-------------|------------------|
| TMPL-01 | Docker Compose fragment for Redis (no `version:`, named volume, health check) | Health check: `redis-cli ping`; volume: `redis_data`; profile: `services` |
| TMPL-02 | Docker Compose fragment for RabbitMQ (no `version:`, named volume, health check, management UI opt-in) | Health check: `rabbitmq-diagnostics check_port_connectivity`; mgmt image bundles UI; profile: `ui` |
| TMPL-03 | Docker Compose fragment for PostgreSQL (no `version:`, named volume, health check) | Health check: `pg_isready -U ${POSTGRES_USER}`; volume: `postgres_data` |
| TMPL-04 | Docker Compose fragment for MySQL (no `version:`, named volume, health check) | MariaDB default: `healthcheck.sh --connect --innodb_initialized`; MySQL: `mysqladmin ping` |
| TMPL-05 | Docker Compose fragment for MongoDB (no `version:`, named volume, health check) | Health check: `mongosh --eval "db.adminCommand('ping')"` — requires mongo 6+ image |
| TMPL-06 | Per-service Taskfile with `up`, `down`, `logs`, `restart`; all `-f` flags use `{{.TASKFILE_DIR}}` | `{{.TASKFILE_DIR}}` resolves to the included file's directory (verified) |
| TMPL-07 | Root `Taskfile.yml` with `includes:` + `optional: true` | Taskfile v3.17+ supports `optional: true`; available in current v3.31 |
| TMPL-08 | All generated files written to `.devtools/` — never outside | Static template design constraint; no runtime enforcement needed in Phase 1 |
| CRED-01 | Credentials written to `.devtools/.env`, never hardcoded in compose YAML | Compose `env_file: .env` resolves relative to compose file location; `${VAR}` syntax in YAML |
| CRED-02 | Generated compose references credentials via `env_file: .env` | `env_file: - .env` on each service (relative path, resolved from `.devtools/`) |
| CRED-03 | `.devtools/.env` is appended, not overwritten; per-service var names namespaced | Template pattern: `REDIS_PORT`, `REDIS_PASSWORD`, `POSTGRES_USER`, etc. |

</phase_requirements>

---

## Summary

Phase 1 delivers static template files only — no code execution, no skill logic. The primary challenge is **getting the template structure architecturally correct** so that Phase 2 (the skill that installs them) doesn't need to revise the templates. All key design decisions are locked in CONTEXT.md; this research answers the technical "how" for implementing them correctly.

**Three critical findings from live testing:**

1. **Docker Compose does NOT support variable expansion in volume declaration keys.** `volumes: ${VAR}_name:` is invalid YAML. The correct pattern is plain volume names like `redis_data` — Docker Compose automatically prefixes them with `COMPOSE_PROJECT_NAME` at runtime. Template volume names in compose files must be plain (no `{{PROJECT_NAME}}_` prefix).

2. **The "services" profile is not truly default** — services with `profiles: [services]` only start when that profile is active. The activation mechanism is `COMPOSE_PROFILES=services` written to `.devtools/.env`. This is the correct implementation of the CONTEXT.md "services profile starts by default" requirement.

3. **External networks must pre-exist.** `external: true` networks fail at `docker compose up` if the network doesn't exist. The app-access `services` network should be declared as a regular bridge network in each per-service compose file — Docker Compose is idempotent for duplicate named network declarations across included files, and creates the network once.

**Primary recommendation:** Author templates with plain volume names, `name: ${COMPOSE_PROJECT_NAME}_<tier>` for network declarations (no `external: true` for the services network in per-service files), and `COMPOSE_PROFILES=services` in the `.env` template.

---

## Standard Stack

### Core
| Library/Tool | Version | Purpose | Why Standard |
|---|---|---|---|
| Docker Compose v2 | Spec (no version key) | Service orchestration format | Required by project; `version:` key officially deprecated |
| Taskfile | v3 (`version: '3'`) | Task runner format | Required by project; `optional: true` on includes since v3.17 |

### Service Image Defaults
| Service | Default Image | Notes |
|---|---|---|
| Redis | `redis:7-alpine` | 7.x stable; alpine for small footprint |
| PostgreSQL | `postgres:16-alpine` | 16.x LTS; alpine available |
| RabbitMQ | `rabbitmq:3-management-alpine` | Bundles management UI (port 15672) + alpine |
| MariaDB (default) | `mariadb:11` | ARM-native; `healthcheck.sh` built in |
| MySQL (opt-in) | `mysql:8` | Requires ARM warning comment |
| MongoDB | `mongo:7` | No alpine variant; `7.0-jammy` is Ubuntu 22.04 base |

### Exporter Images (monitoring profile)
| Service | Image | Version | Notes |
|---|---|---|---|
| Redis | `oliver006/redis_exporter` | `v1.82.0` | Most widely used Redis exporter |
| PostgreSQL | `prometheuscommunity/postgres_exporter` | `v0.19.1` | Official Prometheus community image |
| MySQL/MariaDB | `prom/mysqld_exporter` | `v0.19.0` | Works with both MySQL and MariaDB |
| MongoDB | `percona/mongodb_exporter` | `0.50.0` | Percona's actively maintained exporter |
| RabbitMQ | Built-in (`rabbitmq_prometheus` plugin) | — | No separate container needed; port 15692 |

[VERIFIED: GitHub releases API — oliver006/redis_exporter v1.82.0, prometheus-community/postgres_exporter v0.19.1, prometheus/mysqld_exporter v0.19.0, percona/mongodb_exporter 0.50.0]

### UI Companion Images (ui profile)
| Service | Image | Host Port | Container Port | Notes |
|---|---|---|---|---|
| Redis | `redis/redisinsight:latest` | 8001 | 8001 | Canonical image (redis org, not redislabs) |
| PostgreSQL | `dpage/pgadmin4:latest` | 8085 | 80 | Lightweight; no named volume needed |
| MySQL/MariaDB | `phpmyadmin:latest` | 8080 | 80 | Official phpMyAdmin image |
| MongoDB | `mongo-express:latest` | 8081 | 8081 | Official Docker Hub image |
| RabbitMQ | Already in management image | 15672 | 15672 | Management UI is always-on in base service |

[VERIFIED: Docker Hub API — redis/redisinsight 3.2.0, dpage/pgadmin4 9.14.0 (latest), phpmyadmin 5.2.3]
[ASSUMED: mongo-express — Hub API returned no results; image name `mongo-express` is conventional]

---

## Architecture Patterns

### Recommended Template Directory Structure
```
compose-templates/
├── redis/
│   ├── redis.compose.yml      # Service + UI companion + metrics exporter
│   └── metadata.json          # Parameters, defaults, exporter, ui_companion info
├── postgres/
│   ├── postgres.compose.yml
│   └── metadata.json
├── rabbitmq/
│   ├── rabbitmq.compose.yml
│   └── metadata.json
├── mysql/
│   ├── mysql.compose.yml      # Contains both mariadb:11 and mysql:8 (with ARM comment)
│   └── metadata.json
├── mongodb/
│   ├── mongodb.compose.yml
│   └── metadata.json
└── monitoring/
    ├── monitoring.compose.yml  # Prometheus + Grafana (both ephemeral)
    └── metadata.json

taskfile-templates/
├── redis/
│   └── redis.Taskfile.yml     # up, down, logs, restart tasks
├── postgres/
│   └── postgres.Taskfile.yml
├── rabbitmq/
│   └── rabbitmq.Taskfile.yml
├── mysql/
│   └── mysql.Taskfile.yml
├── mongodb/
│   └── mongodb.Taskfile.yml
└── monitoring/
    └── monitoring.Taskfile.yml
```

### Pattern 1: Per-Service Compose Fragment Structure

**What:** Each service compose file contains THREE containers gated by profiles, plus volume and network declarations.

**When to use:** All service templates — this is the locked architecture.

```yaml
# compose-templates/redis/redis.compose.yml
# Source: Verified via docker compose config testing

services:

  redis:
    image: redis:{{VERSION}}-alpine
    profiles:
      - services                          # Activated by COMPOSE_PROFILES=services in .env
    container_name: ${COMPOSE_PROJECT_NAME}-redis
    env_file:
      - .env                              # Relative to where compose file is installed (.devtools/)
    ports:
      - "127.0.0.1:{{PORT}}:6379"        # Loopback-only binding (security)
    volumes:
      - redis_data:/data                  # Plain name — Docker prefixes with COMPOSE_PROJECT_NAME
    command: redis-server --requirepass ${REDIS_PASSWORD}
    healthcheck:
      test: ["CMD", "redis-cli", "-a", "${REDIS_PASSWORD}", "ping"]
      interval: 10s
      timeout: 5s
      retries: 5
      start_period: 10s
    networks:
      - redis_net                         # Isolated: service + UI only
      - services                          # App-access tier
    restart: unless-stopped

  redisinsight:
    image: redis/redisinsight:latest
    profiles:
      - ui                                # Opt-in
    container_name: ${COMPOSE_PROJECT_NAME}-redisinsight
    ports:
      - "127.0.0.1:8001:8001"
    # NO volumes — ephemeral by design
    networks:
      - redis_net                         # Same network as redis for direct connection
    restart: unless-stopped

  redis_exporter:
    image: oliver006/redis_exporter:latest
    profiles:
      - monitoring                        # Opt-in
    container_name: ${COMPOSE_PROJECT_NAME}-redis-exporter
    environment:
      REDIS_ADDR: redis:6379
      REDIS_PASSWORD: ${REDIS_PASSWORD}
    # NO volumes — ephemeral by design
    networks:
      - redis_net                         # Reach the redis service
      - monitoring                        # Reachable by Prometheus
    restart: unless-stopped

volumes:
  redis_data:                             # PLAIN name — becomes COMPOSE_PROJECT_NAME_redis_data

networks:
  redis_net:
    name: ${COMPOSE_PROJECT_NAME}_redis   # name: supports variable expansion (verified)
  services:
    name: ${COMPOSE_PROJECT_NAME}_services
    # NO external: true — declare as regular bridge; Docker is idempotent across includes
  monitoring:
    name: ${COMPOSE_PROJECT_NAME}_monitoring
    # NO external: true — created by first compose file that activates monitoring profile
```

### Pattern 2: Root Taskfile with Optional Includes

**What:** Root `.devtools/Taskfile.yml` includes per-service Taskfiles. `optional: true` means the root is valid even before any services are installed.

**When to use:** Always — this is the root Taskfile template written to user projects.

```yaml
# taskfile-templates root .devtools/Taskfile.yml template
# Source: Verified via task CLI v3.31.0 testing

version: '3'

includes:
  redis:
    taskfile: ./redis.Taskfile.yml
    optional: true                        # Missing file silently ignored (v3.17+)
  postgres:
    taskfile: ./postgres.Taskfile.yml
    optional: true
  rabbitmq:
    taskfile: ./rabbitmq.Taskfile.yml
    optional: true
  mysql:
    taskfile: ./mysql.Taskfile.yml
    optional: true
  mongodb:
    taskfile: ./mongodb.Taskfile.yml
    optional: true
  monitoring:
    taskfile: ./monitoring.Taskfile.yml
    optional: true
```

### Pattern 3: Per-Service Taskfile with TASKFILE_DIR

**What:** `{{.TASKFILE_DIR}}` resolves to the directory of the INCLUDED taskfile. Since per-service Taskfiles are installed in `.devtools/`, TASKFILE_DIR correctly points to the same directory as the compose files.

**Critical:** `{{.TASKFILE_DIR}}` is a built-in variable — it resolves at runtime to the directory containing the Taskfile being executed, NOT the root Taskfile.

```yaml
# taskfile-templates/redis/redis.Taskfile.yml
# Source: Verified via task CLI testing — TASKFILE_DIR resolves to /path/to/.devtools/

version: '3'

tasks:
  up:
    desc: Start Redis (and any active profiles)
    cmds:
      - docker compose -f {{.TASKFILE_DIR}}/redis.compose.yml up -d
    
  down:
    desc: Stop Redis
    cmds:
      - docker compose -f {{.TASKFILE_DIR}}/redis.compose.yml down
    
  logs:
    desc: Follow Redis logs
    cmds:
      - docker compose -f {{.TASKFILE_DIR}}/redis.compose.yml logs -f redis
    
  restart:
    desc: Restart Redis container
    cmds:
      - docker compose -f {{.TASKFILE_DIR}}/redis.compose.yml restart redis
```

### Pattern 4: Credential Storage in .env Template

**What:** All secrets stored in `.devtools/.env`. Compose YAML references env vars via `${VAR}` syntax, never `{{TOKEN}}`. The `{{TOKEN}}` substitution only happens in the `.env` template when the skill writes it.

```bash
# .env template (written to .devtools/.env by skill)
COMPOSE_PROJECT_NAME={{PROJECT_NAME}}
COMPOSE_PROFILES=services

# Redis
REDIS_PORT={{PORT}}
REDIS_VERSION={{VERSION}}
REDIS_PASSWORD={{PASSWORD}}
```

```yaml
# In compose YAML — uses ${VAR} not {{TOKEN}}
env_file:
  - .env
environment:
  REDIS_PASSWORD: ${REDIS_PASSWORD}   # Resolved from .env at runtime
```

### Pattern 5: metadata.json Structure

**What:** Machine-readable descriptor for each service. The skill reads this to know what tokens to substitute, what defaults to offer, and what UI companion / exporter images to use.

```json
{
  "service": "redis",
  "description": "Redis in-memory key-value store",
  "parameters": [
    { "name": "port",     "default": 6379,  "env_var": "REDIS_PORT",     "token": "{{PORT}}"     },
    { "name": "version",  "default": "7",   "env_var": "REDIS_VERSION",  "token": "{{VERSION}}"  },
    { "name": "password", "default": null,  "env_var": "REDIS_PASSWORD", "token": "{{PASSWORD}}" }
  ],
  "ui_companion": {
    "image": "redis/redisinsight:latest",
    "port": 8001
  },
  "exporter": {
    "image": "oliver006/redis_exporter:latest"
  }
}
```

### Pattern 6: MySQL/MariaDB ARM64 Strategy

**What:** Template has two image options with a comment gate. MariaDB is the default; MySQL is opt-in with ARM warning.

```yaml
# compose-templates/mysql/mysql.compose.yml
services:
  mysql:
    # Default: MariaDB (ARM-native, no emulation required)
    image: mariadb:{{VERSION}}
    # Opt-in MySQL 8: replace the image line above with:
    # NOTE: mysql:8 may require 'platform: linux/amd64' on Apple Silicon (ARM64)
    # image: mysql:8
    profiles:
      - services
    ...
    healthcheck:
      # MariaDB uses healthcheck.sh (included in official image)
      test: ["CMD", "healthcheck.sh", "--connect", "--innodb_initialized"]
      # For MySQL 8, replace with:
      # test: ["CMD", "mysqladmin", "ping", "-h", "localhost"]
```

### Pattern 7: Monitoring Compose Template

**What:** Central monitoring stack (Prometheus + Grafana) installed separately from per-service files.

```yaml
# compose-templates/monitoring/monitoring.compose.yml
services:
  prometheus:
    image: prom/prometheus:latest
    profiles:
      - monitoring
    container_name: ${COMPOSE_PROJECT_NAME}-prometheus
    ports:
      - "127.0.0.1:9090:9090"
    # NO volumes — ephemeral by design
    networks:
      - monitoring
    restart: unless-stopped

  grafana:
    image: grafana/grafana:latest
    profiles:
      - monitoring
    container_name: ${COMPOSE_PROJECT_NAME}-grafana
    ports:
      - "127.0.0.1:3001:3000"
    # NO volumes — ephemeral by design
    networks:
      - monitoring
    restart: unless-stopped

networks:
  monitoring:
    name: ${COMPOSE_PROJECT_NAME}_monitoring
    # Regular bridge — created when monitoring profile first activated
```

### Anti-Patterns to Avoid

- **Variable expansion in volume declaration keys:** `volumes: ${VAR}_data:` is INVALID. [VERIFIED: Docker Compose reports "additional properties not allowed" error]. Use plain `redis_data:` instead — Docker auto-prefixes with project name.
- **`external: true` on the services network in per-service compose:** This would require the network to pre-exist before any `docker compose up`. Declare it as a regular bridge; Docker is idempotent across duplicate named network declarations.
- **`version:` key in any compose file:** Officially obsolete. Docker Compose v2 warns and ignores it.
- **Credentials in compose YAML:** Never substitute `{{PASSWORD}}` directly into compose YAML. Only `${ENV_VAR}` references belong in compose.
- **`docker-compose` (hyphen):** The v1 Python binary is deprecated since 2023. Use `docker compose` (space, v2 plugin).

---

## Don't Hand-Roll

| Problem | Don't Build | Use Instead | Why |
|---|---|---|---|
| Service health checks | Custom ping scripts | Built-in per-service CLIs | `redis-cli ping`, `pg_isready`, `rabbitmq-diagnostics`, `mongosh`, `healthcheck.sh` are all available inside official images |
| Metrics collection | Custom metrics scrapers | Per-service exporter images | oliver006/redis_exporter, prometheuscommunity/postgres_exporter, etc. handle the complexity |
| RabbitMQ metrics | Separate exporter container | Built-in rabbitmq_prometheus plugin | RabbitMQ has native Prometheus support since 3.8 — no third-party container needed |
| Volume prefix logic | Template tokens in volume names | Docker Compose `name:` property on networks, plain names on volumes | Compose auto-prefixes volumes; `name:` property is the right tool for networks |
| Profile activation logic | Conditional logic in templates | `COMPOSE_PROFILES=services` in `.env` | Native Docker Compose feature; no tooling required |

**Key insight:** Every service-specific operational concern (health checking, metrics export, web UI) has a battle-tested container image. The templates should wire these images together, not implement the concerns themselves.

---

## Common Pitfalls

### Pitfall 1: Variable Expansion in Volume Names
**What goes wrong:** Template author writes `volumes: ${COMPOSE_PROJECT_NAME}_redis_data:` expecting a namespaced volume. Docker Compose rejects it with "additional properties not allowed."
**Why it happens:** Variable expansion works in `name:` properties and in service-level config, but NOT in top-level volume declaration keys.
**How to avoid:** Use plain name `redis_data:` — Docker automatically produces `COMPOSE_PROJECT_NAME_redis_data` as the Docker volume name. The `{{PROJECT_NAME}}` token is NOT needed in compose volume declarations.
**Warning signs:** `docker compose config` returns validation error mentioning "additional properties."
[VERIFIED: live `docker compose config` test]

### Pitfall 2: External Network Pre-existence Failure
**What goes wrong:** Template declares `services: external: true`. On first `docker compose up`, Docker fails with "network myapp_services declared as external, but could not be found."
**Why it happens:** `external: true` tells Docker Compose to look for but never create the network.
**How to avoid:** Declare the `services` network WITHOUT `external: true` in each per-service compose. When multiple included compose files declare the same named network, Docker Compose consolidates them and creates it once. User's app then declares it as `external: true` in their own compose.
**Warning signs:** `docker compose up` exits immediately with network-not-found error.
[VERIFIED: live `docker compose up` test — confirmed error, confirmed fix]

### Pitfall 3: Profile=services Makes Services NOT Start by Default
**What goes wrong:** Compose file has `profiles: [services]`. Running `docker compose up` starts NO containers because no profile is active.
**Why it happens:** Profiles in Docker Compose are opt-in — any container with a `profiles:` list only starts when that profile is explicitly activated.
**How to avoid:** Write `COMPOSE_PROFILES=services` in the `.devtools/.env` template. Docker Compose reads this automatically. With this in `.env`, `docker compose up` activates the services profile.
**Warning signs:** `docker compose up` completes with "Network default Creating" but no containers start.
[VERIFIED: live `docker compose config` showed `services: {}` when COMPOSE_PROFILES not set]

### Pitfall 4: TASKFILE_DIR Scope Confusion
**What goes wrong:** Root Taskfile's `TASKFILE_DIR` is used in per-service tasks, resolving to the root's directory instead of the per-service Taskfile's directory.
**Why it happens:** `{{.TASKFILE_DIR}}` resolves to the directory of the FILE being executed, not a global root. In per-service includes, it correctly resolves to the included file's directory.
**How to avoid:** Always write `{{.TASKFILE_DIR}}` in the per-service Taskfiles themselves (not in the root). Since both per-service Taskfiles and compose files are installed in `.devtools/`, TASKFILE_DIR is the correct path for compose `-f` flags.
**Warning signs:** `docker compose -f {{.TASKFILE_DIR}}/redis.compose.yml` returns "no such file" despite files existing.
[VERIFIED: task CLI v3.31.0 test — TASKFILE_DIR resolves to /path/to/.devtools/]

### Pitfall 5: RabbitMQ Prometheus Plugin Not Enabled
**What goes wrong:** `rabbitmq:3-management-alpine` image doesn't serve metrics on port 15692 out of the box — the `rabbitmq_prometheus` plugin is NOT enabled in the base management image.
**Why it happens:** RabbitMQ plugins are opt-in; the management image only enables the management plugin.
**How to avoid:** For the monitoring profile, either (a) mount a custom `enabled_plugins` file listing both `[rabbitmq_management, rabbitmq_prometheus].`, or (b) add a startup command that enables the plugin. Include this in the compose monitoring profile section.
**Warning signs:** `curl http://localhost:15692/metrics` returns 404 or connection refused.
[ASSUMED: Based on RabbitMQ documentation conventions; plugin enablement mechanism not live-tested]

### Pitfall 6: MongoDB Healthcheck Requires mongosh (not mongo)
**What goes wrong:** Template uses `["CMD", "mongo", "--eval", "db.adminCommand('ping')"]` — this fails in MongoDB 6+/7+ because the legacy `mongo` shell was removed.
**Why it happens:** MongoDB 6.0 replaced the `mongo` shell with `mongosh` (MongoDB Shell).
**How to avoid:** Use `["CMD", "mongosh", "--eval", "db.adminCommand('ping')", "--quiet"]`.
**Warning signs:** Healthcheck shows "unhealthy" with exit code 127 (command not found).
[VERIFIED: MongoDB 7.0 uses mongosh; Docker Hub tags confirmed 7.0-jammy]

### Pitfall 7: Port Bindings on All Interfaces
**What goes wrong:** `ports: - "5432:5432"` exposes PostgreSQL on all network interfaces (0.0.0.0), making it accessible over the local network (e.g., at coffee shops, open offices).
**Why it happens:** Docker default is 0.0.0.0 for host port bindings.
**How to avoid:** Always use `127.0.0.1:PORT:PORT` in port bindings for database services. This is a security best practice for local dev.
**Warning signs:** `docker ps` shows `0.0.0.0:5432->5432/tcp` instead of `127.0.0.1:5432->5432/tcp`.

---

## Healthcheck Reference

| Service | Command | Notes |
|---|---|---|
| Redis | `["CMD", "redis-cli", "-a", "${REDIS_PASSWORD}", "ping"]` | Requires `-a` flag when `requirepass` is set |
| PostgreSQL | `["CMD-SHELL", "pg_isready -U ${POSTGRES_USER:-postgres}"]` | CMD-SHELL for shell expansion |
| RabbitMQ | `["CMD", "rabbitmq-diagnostics", "check_port_connectivity"]` | Verifies AMQP port accepting connections |
| MariaDB | `["CMD", "healthcheck.sh", "--connect", "--innodb_initialized"]` | Built into MariaDB official image (10.4+) |
| MySQL | `["CMD", "mysqladmin", "ping", "-h", "localhost"]` | Standard for MySQL 8; not in MariaDB |
| MongoDB | `["CMD", "mongosh", "--eval", "db.adminCommand('ping')", "--quiet"]` | `mongosh` only (MongoDB 6+); no auth needed for ping |

[VERIFIED: All healthcheck syntaxes validated via `docker compose config` parsing]

---

## Code Examples

### Complete Redis Template (Annotated)

```yaml
# compose-templates/redis/redis.compose.yml
# Template tokens: {{PORT}}, {{VERSION}}, {{PASSWORD}}
# These are substituted at install time by the skill

services:

  # ── Main service (profile: services) ──────────────────────────────────────
  redis:
    image: redis:{{VERSION}}-alpine
    profiles:
      - services
    container_name: ${COMPOSE_PROJECT_NAME}-redis
    env_file:
      - .env
    ports:
      - "127.0.0.1:{{PORT}}:6379"
    volumes:
      - redis_data:/data
    command: redis-server --requirepass ${REDIS_PASSWORD}
    healthcheck:
      test: ["CMD", "redis-cli", "-a", "${REDIS_PASSWORD}", "ping"]
      interval: 10s
      timeout: 5s
      retries: 5
      start_period: 10s
    networks:
      - redis_net
      - services
    restart: unless-stopped

  # ── UI companion (profile: ui) ─────────────────────────────────────────────
  redisinsight:
    image: redis/redisinsight:latest
    profiles:
      - ui
    container_name: ${COMPOSE_PROJECT_NAME}-redisinsight
    ports:
      - "127.0.0.1:8001:8001"
    networks:
      - redis_net
    restart: unless-stopped

  # ── Metrics exporter (profile: monitoring) ─────────────────────────────────
  redis_exporter:
    image: oliver006/redis_exporter:latest
    profiles:
      - monitoring
    container_name: ${COMPOSE_PROJECT_NAME}-redis-exporter
    environment:
      REDIS_ADDR: redis:6379
      REDIS_PASSWORD: ${REDIS_PASSWORD}
    networks:
      - redis_net
      - monitoring
    restart: unless-stopped

volumes:
  redis_data:                           # Auto-becomes: COMPOSE_PROJECT_NAME_redis_data

networks:
  redis_net:
    name: ${COMPOSE_PROJECT_NAME}_redis
  services:
    name: ${COMPOSE_PROJECT_NAME}_services
  monitoring:
    name: ${COMPOSE_PROJECT_NAME}_monitoring
```

### Complete Redis .env Template

```bash
# Written to .devtools/.env by skill — tokens substituted at install time
COMPOSE_PROJECT_NAME={{PROJECT_NAME}}
COMPOSE_PROFILES=services

# Redis
REDIS_PORT={{PORT}}
REDIS_VERSION={{VERSION}}
REDIS_PASSWORD={{PASSWORD}}
```

### Complete Redis metadata.json

```json
{
  "service": "redis",
  "description": "Redis in-memory key-value store",
  "parameters": [
    { "name": "port",     "default": 6379,  "env_var": "REDIS_PORT",     "token": "{{PORT}}"     },
    { "name": "version",  "default": "7",   "env_var": "REDIS_VERSION",  "token": "{{VERSION}}"  },
    { "name": "password", "default": null,  "env_var": "REDIS_PASSWORD", "token": "{{PASSWORD}}" }
  ],
  "ui_companion": {
    "image": "redis/redisinsight:latest",
    "port": 8001
  },
  "exporter": {
    "image": "oliver006/redis_exporter:latest"
  }
}
```

### Per-Service Taskfile Template

```yaml
# taskfile-templates/redis/redis.Taskfile.yml
# {{.TASKFILE_DIR}} resolves to the installed location (.devtools/) at runtime

version: '3'

tasks:
  up:
    desc: Start Redis
    cmds:
      - docker compose -f {{.TASKFILE_DIR}}/redis.compose.yml up -d

  down:
    desc: Stop Redis
    cmds:
      - docker compose -f {{.TASKFILE_DIR}}/redis.compose.yml down

  logs:
    desc: Follow Redis logs
    cmds:
      - docker compose -f {{.TASKFILE_DIR}}/redis.compose.yml logs -f redis

  restart:
    desc: Restart Redis container
    cmds:
      - docker compose -f {{.TASKFILE_DIR}}/redis.compose.yml restart redis
```

### PostgreSQL Healthcheck with User Variable

```yaml
# The CMD-SHELL form is required to expand ${POSTGRES_USER}
healthcheck:
  test: ["CMD-SHELL", "pg_isready -U ${POSTGRES_USER:-postgres}"]
  interval: 10s
  timeout: 5s
  retries: 5
  start_period: 20s
```

### MySQL/MariaDB Template ARM Strategy

```yaml
services:
  mysql:
    # Default: MariaDB 11 (ARM-native, no emulation required on Apple Silicon)
    image: mariadb:{{VERSION}}
    #
    # ── To use MySQL 8 instead ──────────────────────────────────────────────
    # NOTE: mysql:8 may require 'platform: linux/amd64' on Apple Silicon (ARM64)
    # image: mysql:8
    # ───────────────────────────────────────────────────────────────────────
    profiles:
      - services
    environment:
      MARIADB_ROOT_PASSWORD: ${MYSQL_ROOT_PASSWORD}
      MARIADB_USER: ${MYSQL_USER}
      MARIADB_PASSWORD: ${MYSQL_PASSWORD}
      MARIADB_DATABASE: ${MYSQL_DB}
    healthcheck:
      # MariaDB: uses healthcheck.sh (built into official image 10.4+)
      test: ["CMD", "healthcheck.sh", "--connect", "--innodb_initialized"]
      # For MySQL 8, replace with:
      # test: ["CMD", "mysqladmin", "ping", "-h", "localhost"]
      interval: 10s
      timeout: 5s
      retries: 5
      start_period: 30s
```

---

## Network Architecture Reference

```
Docker networking per-project (COMPOSE_PROJECT_NAME=myapp):

User's app container
       │
       ▼ (declares: external: true in their compose)
myapp_services          ← app-access tier (regular bridge in devtools composes)
  │     │
  ▼     ▼
redis  postgres         ← main service containers
  │     │
  ▼     ▼
myapp_redis             ← per-service isolated networks
myapp_postgres
  │     │
  ▼     ▼
redisinsight  pgadmin   ← UI companions (same per-service network as their service)
              
redis_exporter ─────────────────────────────► myapp_monitoring ◄─── Prometheus
postgres_exporter ──────────────────────────► myapp_monitoring ◄─── Grafana
```

**Network creation responsibility:**
- `myapp_redis`, `myapp_postgres` etc: created by their respective compose files on first `up`
- `myapp_services`: created by the FIRST per-service compose file that includes it (Docker is idempotent)
- `myapp_monitoring`: created by the monitoring compose file OR the first exporter that references it
- User's app: declares `myapp_services` as `external: true` — it already exists after any devtools service is started

---

## State of the Art

| Old Approach | Current Approach | When Changed | Impact |
|---|---|---|---|
| `version: "3.9"` in compose | No version key | Compose Spec 2024+ | Must omit entirely — generates deprecation warning |
| `docker-compose` (hyphen, v1) | `docker compose` (space, v2 plugin) | Docker Desktop 4.x, 2023 | v1 removed from modern Docker Desktop; use v2 |
| `mongo` shell for healthcheck | `mongosh` | MongoDB 6.0 | Legacy `mongo` shell removed from official images |
| `redislabs/redisinsight` | `redis/redisinsight` | 2023 | Org rename; both currently work but `redis/` is canonical |
| Separate RabbitMQ exporter (kbudde) | Built-in rabbitmq_prometheus plugin | RabbitMQ 3.8+ | No third-party container needed; plugin is native |

---

## Assumptions Log

| # | Claim | Section | Risk if Wrong |
|---|---|---|---|
| A1 | `mongo-express:latest` is the correct Docker Hub image name for MongoDB web UI | Standard Stack / UI Companions | Wrong image name → compose pull fails at install time; easy to fix |
| A2 | `rabbitmq_prometheus` plugin is NOT enabled by default in `rabbitmq:3-management-alpine` | Pitfall 5 | If plugin IS enabled by default, monitoring just works without extra config; harmless |
| A3 | Taskfile `optional: true` requires v3.17 minimum; v3.31 (current env) supports it | Standard Stack | Version constraint in root Taskfile template may be wrong; test at install time |

---

## Open Questions (RESOLVED)

1. **Prometheus scrape config location**
   - What we know: Prometheus needs to know each exporter's host:port
   - What's unclear: Does `monitoring.compose.yml` ship a static `prometheus.yml`, or is it generated by the skill per-project?
   - **RESOLVED:** Deferred to Phase 2 per CONTEXT.md — Phase 1 only authors per-service templates; skill logic is Phase 2 scope.

2. **RabbitMQ prometheus plugin enablement mechanism**
   - What we know: `rabbitmq_prometheus` plugin enables port 15692 metrics; not on by default
   - What's unclear: Best template-friendly approach (custom enabled_plugins file mount vs command override)
   - **RESOLVED:** Phase 1 template includes a comment explaining plugin activation. Phase 2 skill will mount the `enabled_plugins` file when the monitoring profile is requested.

3. **Root compose.yml template**
   - What we know: The root `.devtools/compose.yml` uses `include:` to pull in per-service files
   - What's unclear: Is this file a static template in `compose-templates/root/`, or generated dynamically per-project by the skill?
   - **RESOLVED:** Deferred to Phase 2 per CONTEXT.md — Phase 1 only authors per-service templates; root compose generation is Phase 2 skill logic.

---

## Environment Availability

| Dependency | Required By | Available | Version | Fallback |
|---|---|---|---|---|
| Docker Engine | Compose template validation | ✓ | 29.4.0 | — |
| docker compose | Template testing | ✓ | v2 (bundled) | — |
| task (Taskfile CLI) | Taskfile template testing | ✓ | v3.31.0 | — |

All dependencies available. No blockers for template authoring and local validation.

---

## Validation Architecture

### Test Framework
| Property | Value |
|---|---|
| Framework | Shell validation (no test framework — these are static YAML/JSON files) |
| Config file | None — run commands directly |
| Quick run command | `docker compose -f <file> config --quiet` |
| Full suite command | See wave checklist below |

### Phase Requirements → Test Map

| Req ID | Behavior | Test Type | Automated Command | File Exists? |
|---|---|---|---|---|
| TMPL-01 | Redis compose has valid syntax, no `version:`, named volume, healthcheck | lint | `docker compose -f compose-templates/redis/redis.compose.yml config --quiet` | ❌ Wave 0 |
| TMPL-02 | RabbitMQ compose valid, management UI in `ui` profile | lint | `docker compose -f compose-templates/rabbitmq/rabbitmq.compose.yml config --quiet` | ❌ Wave 0 |
| TMPL-03 | PostgreSQL compose valid, healthcheck uses pg_isready | lint | `docker compose -f compose-templates/postgres/postgres.compose.yml config --quiet` | ❌ Wave 0 |
| TMPL-04 | MySQL compose valid, mariadb default, mysql opt-in | lint | `docker compose -f compose-templates/mysql/mysql.compose.yml config --quiet` | ❌ Wave 0 |
| TMPL-05 | MongoDB compose valid, mongosh healthcheck | lint | `docker compose -f compose-templates/mongodb/mongodb.compose.yml config --quiet` | ❌ Wave 0 |
| TMPL-06 | Per-service Taskfiles have up/down/logs/restart tasks with TASKFILE_DIR | lint | `task --taskfile taskfile-templates/<service>/<service>.Taskfile.yml --list-all` | ❌ Wave 0 |
| TMPL-07 | Root Taskfile includes all services with optional: true | lint | `task --taskfile taskfile-templates/root/Taskfile.yml --list-all` | ❌ Wave 0 |
| CRED-01 | No compose YAML contains hardcoded passwords (grep check) | static analysis | `grep -r "password:" compose-templates/ --include="*.yml"` (must find 0 non-commented lines with literal passwords) | ❌ Wave 0 |
| CRED-02 | All compose services have `env_file: - .env` | static analysis | `grep -l "env_file" compose-templates/**/*.yml` (must match all service files) | ❌ Wave 0 |
| CRED-03 | `.env` templates use namespaced var names | manual review | Inspect `.env` template tokens: REDIS_PORT, POSTGRES_USER, etc. | ❌ Wave 0 |

### No version: Key Invariant
```bash
# Run after all templates are written — must return 0 matches
grep -r "^version:" compose-templates/ --include="*.yml"
```

### Sampling Rate
- **Per file created:** `docker compose -f <file> config --quiet && echo PASS`
- **Per wave merge:** Full grep sweep + all config validations
- **Phase gate:** All 6 service compose files pass `config --quiet`; all 6 Taskfiles pass `--list-all`

### Wave 0 Gaps
- [ ] All `compose-templates/<service>/<service>.compose.yml` files — don't exist yet (this is the work)
- [ ] All `taskfile-templates/<service>/<service>.Taskfile.yml` files — don't exist yet
- [ ] All `metadata.json` files — don't exist yet
- [ ] `compose-templates/monitoring/monitoring.compose.yml` — doesn't exist yet
- [ ] `taskfile-templates/monitoring/monitoring.Taskfile.yml` — doesn't exist yet

*(This entire phase is Wave 0 — creating files from scratch)*

---

## Security Domain

### Applicable ASVS Categories

| ASVS Category | Applies | Standard Control |
|---|---|---|
| V2 Authentication | No | Credentials are generated config — no auth layer in templates themselves |
| V3 Session Management | No | Static files, no sessions |
| V4 Access Control | No | Local dev only; no multi-user access control |
| V5 Input Validation | No | Static template authoring; no runtime input in Phase 1 |
| V6 Cryptography | Partial | Credentials in `.env` (not compose YAML); `.env` must be gitignored |

### Threat Patterns for Local Dev Tooling

| Pattern | STRIDE | Standard Mitigation |
|---|---|---|
| Database exposed on all interfaces | Information Disclosure | `127.0.0.1:PORT:PORT` binding in all port mappings (not `PORT:PORT`) |
| Credentials hardcoded in compose YAML | Information Disclosure | `env_file: .env` pattern; credential tokens only in `.env` template |
| `.env` committed to git | Information Disclosure | Skill gitignores `.devtools/.env` on first setup (Phase 2 responsibility) |
| Volume collision across projects | Tampering | `COMPOSE_PROJECT_NAME` in `.env` ensures project-scoped volume names |

---

## Sources

### Primary (HIGH confidence)
- Live `docker compose config` testing — Docker Engine 29.4.0 — volume expansion behavior, network naming, profiles, external networks, include: directive
- Live `task --taskfile` testing — Task v3.31.0 — `{{.TASKFILE_DIR}}` resolution, `optional: true` includes behavior
- `copilot-instructions.md` / STACK.md — project conventions, confirmed Compose v2 no-version requirement

### Secondary (MEDIUM confidence)
- GitHub Releases API — verified exporter versions: redis_exporter v1.82.0, postgres_exporter v0.19.1, mysqld_exporter v0.19.0, mongodb_exporter 0.50.0
- Docker Hub API — verified RedisInsight 3.2.0, pgAdmin4 9.14.0, phpmyadmin 5.2.3

### Tertiary (LOW confidence — marked [ASSUMED])
- mongo-express image name and version (Docker Hub API returned no results)
- RabbitMQ prometheus plugin default-disabled status (not live-tested)

---

## Metadata

**Confidence breakdown:**
- Standard stack: HIGH — live tested with Docker 29.4 and Task v3.31
- Architecture: HIGH — all key patterns verified via `docker compose config` 
- Pitfalls: HIGH — three of the seven pitfalls caught by live testing; rest from documented behavior
- Exporter/UI images: MEDIUM — versions verified from GitHub releases API; images not pulled

**Research date:** 2026-04-10
**Valid until:** 2026-05-10 (image versions may change; patterns are stable)

---

## RESEARCH COMPLETE
