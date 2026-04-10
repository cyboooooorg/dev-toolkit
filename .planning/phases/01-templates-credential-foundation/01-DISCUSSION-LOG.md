# Phase 1 Discussion Log

## Session: Templates & Credential Foundation

---

### Area 1: Output filename / file structure ✅

**Decision:** Per-service Compose fragments named `<service>.compose.yml`, stored in `compose-templates/<service>/`. Root `.devtools/compose.yml` uses Docker Compose `include:` to pull in each fragment.

**Docker Compose profiles:**
- `services` — default (starts without `--profile`)
- `ui` — management UI companions (opt-in)
- `monitoring` — Grafana, Prometheus, exporters (opt-in)

**Taskfile tasks:** Both global profile tasks (`up`, `up-ui`, `up-monitoring`, `down`, `logs`) and per-service namespace tasks (`redis:up`, `redis:ui`, `redis:down`, etc.)

---

### Area 2: Volume naming ✅

**Decision:**
- Standard: `${COMPOSE_PROJECT_NAME}_<service>_data`
- Multi-instance: `${COMPOSE_PROJECT_NAME}_<alias>_data`
- UI companions: ephemeral only (no named volumes)
- Monitoring (Grafana, Prometheus): ephemeral only

---

### Area 3: Template storage format ✅

**Decision:**
- Compose templates: `compose-templates/<service>/` per service
- Taskfile templates: `taskfile-templates/<service>/` per service
- Monitoring in dedicated top-level dirs: `compose-templates/monitoring/`, `taskfile-templates/monitoring/`
- Templates use `{{PLACEHOLDER}}` tokens substituted at install time
- Each service dir includes a `metadata.json` with parameters, defaults, env var names, UI companion details, and exporter image

---

### Area 4: MySQL / ARM64 ✅

**Decision:**
- Default: `mariadb:11` (ARM-native, MySQL-compatible)
- Opt-in: Skill asks "MySQL or MariaDB?" — uses `mysql:8` if explicitly chosen
- ARM64 warning comment emitted in compose file when `mysql:8` is selected

---

### Area 5: Docker networks ✅

**Decision (three-tier networking):**

1. **Per-service network** (`${COMPOSE_PROJECT_NAME}_<service>`) — service container + its UI companion; regular bridge
2. **App-access network** (`${COMPOSE_PROJECT_NAME}_services`) — all service containers join this; user's app declares it as `external: true`
3. **Monitoring network** (`${COMPOSE_PROJECT_NAME}_monitoring`) — Grafana + Prometheus + per-service exporter sidecars

Network names scoped to `COMPOSE_PROJECT_NAME`. Per-service networks are NOT `internal: true`. Metrics exporters are sidecar containers that join the monitoring network when the monitoring profile is enabled.

---

### Scope Redirects

None during this session.
