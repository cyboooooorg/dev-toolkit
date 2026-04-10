# Requirements — dev-tools

## v1 Requirements

### Skill Format (SKILL)

- [ ] **SKILL-01**: AI invokes the skill when a user says something like "add Redis to this project" or "install RabbitMQ"
- [ ] **SKILL-02**: Skill is defined as a single canonical `SKILL.md` file installed in both `.github/skills/` (Copilot CLI) and `.agents/skills/` (Claude/MCP)
- [ ] **SKILL-03**: Skill description in frontmatter explicitly enumerates all 5 supported services so AI runtimes can reliably trigger it

### Service Templates (TMPL)

- [ ] **TMPL-01**: Docker Compose service fragment for Redis (no `version:` key, named volume, health check)
- [ ] **TMPL-02**: Docker Compose service fragment for RabbitMQ (no `version:` key, named volume, health check, management UI port opt-in)
- [ ] **TMPL-03**: Docker Compose service fragment for PostgreSQL (no `version:` key, named volume, health check)
- [ ] **TMPL-04**: Docker Compose service fragment for MySQL (no `version:` key, named volume, health check)
- [ ] **TMPL-05**: Docker Compose service fragment for MongoDB (no `version:` key, named volume, health check)
- [ ] **TMPL-06**: Per-service Taskfile template for each service (e.g. `redis.yml`) with tasks: `up`, `down`, `logs`, `restart`; all `-f` flags use `{{.TASKFILE_DIR}}` for portable paths
- [ ] **TMPL-07**: Root `Taskfile.yml` (or append to existing) with `includes:` entries pointing to `.devtools/*.yml` using `optional: true`
- [ ] **TMPL-08**: All generated files written to `.devtools/` directory — the skill never modifies files outside `.devtools/`

### Interactive Configuration (CONF)

- [ ] **CONF-01**: Before writing any files, the skill asks for: service port, image version/tag, and credentials (username, password, database name where applicable)
- [ ] **CONF-02**: For RabbitMQ, additionally ask whether to expose the Management UI port (15672)
- [ ] **CONF-03**: Each configuration question provides a sensible default the user can accept or override

### Credentials Handling (CRED)

- [ ] **CRED-01**: Credentials are written to `.devtools/.env`, never hardcoded into `docker-compose.yml`
- [ ] **CRED-02**: Generated `docker-compose.yml` references credentials via `env_file: .env` (relative to `.devtools/`)
- [ ] **CRED-03**: `.devtools/.env` is appended to (not overwritten) when additional services are added; per-service variable names are namespaced (e.g. `POSTGRES_USER`, `REDIS_PORT`)

### Multi-instance & Idempotency (MERGE)

- [ ] **MERGE-01**: Before writing, the skill checks if the service already exists in `.devtools/`
- [ ] **MERGE-02**: If the service already exists, the skill informs the user and asks: add another instance, or cancel
- [ ] **MERGE-03**: If adding another instance, the skill asks for an alias/name (e.g. `redis-cache`, `redis-session`) used to namespace the service, Taskfile, volume, and env vars
- [ ] **MERGE-04**: Install is idempotent for identical alias — running the skill again for `redis-cache` that already exists does not overwrite it

### Distribution (DIST)

- [ ] **DIST-01**: Canonical `SKILL.md` is symlinked (or copied) into both `.github/skills/add-service/` and `.agents/skills/add-service/`
- [ ] **DIST-02**: Repository includes a `README.md` explaining how to install the skill into a project and which services are supported

---

## v2 Requirements (Deferred)

- Auto-detect existing stack from project files (package.json, requirements.txt, etc.) and suggest relevant services
- `.devtools/.env` generation as a separate "generate env" skill step
- Service-specific CLI tasks (e.g. `redis:cli`, `mongo:shell`, `psql`)
- Version pinning recommendations surfaced during config (warn if `latest` tag used)
- `docker compose include:` integration for projects with an existing root `compose.yaml`
- Kafka / Elasticsearch service templates
- Health-check wait task (poll until service is ready)

---

## Out of Scope

- Kubernetes / Helm chart generation — different toolchain, different problem
- Cloud provisioning (AWS RDS, ElastiCache, etc.) — local dev only
- Language-specific app scaffolding — only infrastructure config
- Service monitoring dashboards — out of scope for v1
- Automatic port conflict detection — violates zero-dependency constraint
- MCP server implementation — overkill; SKILL.md format is sufficient

---

## Traceability

| REQ-ID | Phase | Notes |
|--------|-------|-------|
| SKILL-01, SKILL-02, SKILL-03 | Phase 3 | Depends on templates being locked |
| TMPL-01–TMPL-08 | Phase 1 | Core deliverable |
| CONF-01–CONF-03 | Phase 2 | Embedded in SKILL.md flow logic |
| CRED-01–CRED-03 | Phase 1 (template) + Phase 2 (flow) | Template defines structure; flow asks the questions |
| MERGE-01–MERGE-04 | Phase 2 | Merge logic in SKILL.md |
| DIST-01–DIST-02 | Phase 3 | Wiring + README |
