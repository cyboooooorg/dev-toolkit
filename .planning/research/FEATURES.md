# Feature Research

**Domain:** AI skill / developer tooling — local Docker service scaffolding with Taskfile
**Researched:** 2025-01-26
**Confidence:** HIGH (for table stakes and architecture), MEDIUM (for differentiators — based on competitive analysis with verified sources)

---

## Feature Landscape

### Table Stakes (Users Expect These)

Features users assume exist. Missing these = product feels incomplete.

| Feature | Why Expected | Complexity | Notes |
|---------|--------------|------------|-------|
| Docker Compose service definition per service | Core deliverable; zero value without it | LOW | Redis, RabbitMQ, PostgreSQL, MySQL, MongoDB. Compose v2 syntax (no `version:` key). |
| Sensible local-dev defaults | Developers expect zero friction for first run | LOW | Port defaults (Redis: 6379, PG: 5432, MySQL: 3306, Mongo: 27017, AMQP: 5672), no auth by default for Redis/Mongo, guest/guest for RabbitMQ |
| Named volume for data persistence | Without this, data is lost on `docker compose down` | LOW | Each service gets a named volume (`redis_data`, `postgres_data`, etc.) |
| `up` / `down` / `restart` / `logs` Taskfile tasks | Standard task runner contract every dev expects | LOW | Per-service Taskfile. `up` = start detached; `down` = stop+remove; `logs` = follow logs; `restart` = down+up |
| Interactive config before file generation | Prevents wrong defaults baked in permanently | MEDIUM | Ask port, version, credentials before writing — never assume |
| Merge / no-clobber behavior | Developers re-run the skill to add a second service; must not blow away the first | MEDIUM | If `.devtools/` exists, add new service alongside existing; never overwrite |
| Root `Taskfile.yml` that includes per-service files | Without this the per-service Taskfiles are unreachable by default `task` | LOW | Uses Taskfile `includes:` with `optional: true` so new includes can be appended |
| Clear post-generation output | After files are written, show the developer exactly what was created and how to use it | LOW | Print file paths created, default connection string, key task commands |
| Connection string / DSN shown after setup | Developers immediately need to paste this into their app config | LOW | e.g., `redis://localhost:6379`, `postgresql://postgres:password@localhost:5432/mydb` |
| Docker Compose health checks | Without health checks, `depends_on` is meaningless and services race | MEDIUM | Each service gets an appropriate healthcheck (`redis-cli ping`, `pg_isready`, `mysqladmin ping`, MongoDB ping) |

### Differentiators (Competitive Advantage)

Features that set the product apart. Not required, but valued.

| Feature | Value Proposition | Complexity | Notes |
|---------|-------------------|------------|-------|
| Natural language trigger ("add Redis to this project") | Zero config-file learning curve; developer never reads docs | MEDIUM | AI skill definition that maps intent → skill invocation. Works in Copilot CLI + Claude/MCP |
| Dual-runtime AI skill (Copilot CLI + Claude/MCP) | Works in the AI tool the developer already uses | MEDIUM | Skill metadata defined once, mapped to both runtime formats; no runtime-specific logic in templates |
| Zero dependency execution | The skill runs anywhere — no npm, pip, brew, or package manager needed | LOW | All templates are static text; skill is a shell script or pure markdown+YAML instructions |
| Service-specific shell access tasks | DDEV has `ddev exec`/`ddev ssh`; we offer targeted equivalents | LOW | `redis:cli` → `redis-cli`, `postgres:psql` → `psql`, `mysql:cli`, `mongo:shell`, `rabbitmq:admin` |
| Per-service Taskfile as isolated module | Adding service N doesn't require editing any existing file | LOW | Taskfile `includes:` with `optional: true` means each service is independently droppable |
| Version pinning with explicit tags | Reproducibility — not `redis:latest` which breaks randomly | LOW | Interactive prompt: "Which Redis version? (default: 7)" → writes `redis:7-alpine` |
| `.env` integration in Compose | Credentials never hardcoded; easy to gitignore `.env` | LOW | `compose.yml` references `${REDIS_PORT:-6379}` etc.; write `.devtools/.env.example` with defaults |
| RabbitMQ Management UI port exposed by default | RabbitMQ users always need port 15672; Compose tools routinely miss this | LOW | Expose 15672 by default; include management plugin in image tag |

### Anti-Features (Commonly Requested, Often Problematic)

Features that seem good but create problems.

| Feature | Why Requested | Why Problematic | Alternative |
|---------|---------------|-----------------|-------------|
| Automatic port conflict detection | "Don't make me remember ports" | Requires running Docker inspection at skill invocation time; adds dependency on Docker CLI being available during AI skill execution; fragile | Prompt for port with clear default; document how to change it later in the generated Taskfile |
| Reverse proxy / automatic HTTPS (Lando/Traefik style) | DDEV and Lando provide HTTPS for local dev | Local services (Redis, RabbitMQ, Postgres) don't benefit from HTTPS termination; adds Traefik container dependency for zero gain | Expose ports directly; developers who need TLS can add it manually |
| Full web UI dashboard | "I want to see all my services" | Requires a persistent daemon process (Portainer, etc.); this tool writes files, it is not a daemon | Include URLs to existing UIs where appropriate (RabbitMQ management at `:15672`, Mongo Express is a separate add-on) |
| Language-specific scaffolding (app code, env files) | "Also add the Redis client to my Node app" | This is a different problem domain; pollutes the tool's core value with framework-specific knowledge | Scope is infrastructure config only; print the connection string and let the developer wire it up |
| Kubernetes / Helm output | "We use k8s in prod" | Different problem, much higher complexity, separate toolchain | Out of scope per PROJECT.md; recommend Helm community charts for prod |
| Service-to-service networking config | "My app container needs to reach Postgres" | Requires knowledge of the user's app container, which we don't have | Document the Compose network name (`.devtools_default`); developers add their app service manually |
| Database GUI container (Adminer, pgAdmin, Mongo Express) | "Add a DB admin interface" | GUI containers have their own versioning/config surface; doubles the maintenance | Document that these are easy to add as an additional Compose service; provide a note in generated output |
| Auto-detection of existing project stack | "Detect my framework and configure appropriately" | NLP ambiguity, false positives, complex to maintain across frameworks | Interactive prompts are already explicit; no silent detection |
| `docker-compose.override.yml` generation | "I want to customize without editing the main file" | Adds a second file for developers to track; confusion about which file is authoritative | Use `${ENV_VAR:-default}` in the main Compose file; developers override via `.env` |

---

## Feature Dependencies

```
[Interactive config prompts]
    └──required by──> [Docker Compose service file generation]
                           └──required by──> [Named volumes]
                           └──required by──> [Health checks]
                           └──required by──> [Taskfile per-service]
                                                └──required by──> [Root Taskfile includes]

[Root Taskfile includes]
    └──required by──> [Merge / no-clobber behavior]
                           (appending a new `includes:` entry must not break existing ones)

[.env integration]
    └──enhances──> [Docker Compose service file generation]
    └──enhances──> [Connection string output]
    (reading from .env.example makes connection strings self-documenting)

[AI skill definition]
    └──required by──> [Natural language trigger]
    └──required by──> [Dual-runtime support]

[Zero dependency execution]
    └──conflicts with──> [Automatic port conflict detection]
    (port detection requires shelling out to docker CLI at runtime)

[Service-specific shell access tasks]
    └──enhances──> [Taskfile per-service]
    (these are additional tasks appended to the per-service Taskfile)
```

### Dependency Notes

- **Interactive config requires prompt flow before file writes:** The skill must complete the full Q&A before writing a single file; partial writes (write, then ask) break merge logic.
- **Root Taskfile includes requires optional flag:** `optional: true` on each include means the root Taskfile is valid even before any services are added — critical for the "first service" case when no root file exists yet.
- **Merge / no-clobber depends on root Taskfile includes pattern:** If each service is its own `includes:` entry, adding service #2 means appending one line to the root file rather than rewriting it.
- **.env integration enhances but doesn't block:** Compose files work with hardcoded defaults; `.env` support is an enhancement that adds security without blocking core functionality.
- **Zero dependency conflicts with runtime detection:** Any feature requiring Docker/shell inspection at AI skill invocation time (port scan, stack detection) violates the no-dependency constraint.

---

## MVP Definition

### Launch With (v1)

Minimum viable product — validates the concept: "AI adds a service in one conversation."

- [ ] **Interactive config prompts** — without this, the skill writes opinionated defaults the user may immediately need to change, defeating the purpose
- [ ] **Docker Compose service definitions** for all 5 services (Redis, RabbitMQ, PostgreSQL, MySQL, MongoDB) — core deliverable
- [ ] **Named volumes** on each service — data loss on first `docker compose down` is a terrible first impression
- [ ] **Health checks** per service — makes `depends_on: condition: service_healthy` usable by the developer's app compose
- [ ] **Per-service Taskfile** with `up`, `down`, `logs`, `restart` — core task runner contract
- [ ] **Root `Taskfile.yml`** with `includes:` for each added service — makes `task redis:up` work out of the box
- [ ] **Merge / no-clobber** behavior — critical for multi-service projects; cannot be bolted on later
- [ ] **Post-generation output** with file list + connection string — reduces "now what?" confusion
- [ ] **AI skill definition** for both Copilot CLI + Claude/MCP — the entire value prop depends on it

### Add After Validation (v1.x)

Features to add once core is working and patterns are stable.

- [ ] **Service-specific shell access tasks** (`redis:cli`, `postgres:psql`, etc.) — adds polish; trigger: first user feedback requests
- [ ] **`.env` / `.env.example` generation** — improves credential management; trigger: any user hardcoding credentials in compose
- [ ] **Version pinning prompts** — "Which version?" with sensible default; trigger: first "I needed postgres:15 not latest" complaint
- [ ] **RabbitMQ Management UI port** exposed and documented — easy win; can be baked into v1 template but validate port exposure first

### Future Consideration (v2+)

Features to defer until product-market fit is established.

- [ ] **Additional services** (MinIO, Elasticsearch, Kafka, NATS, Valkey) — defer until user demand is clear; each service = new template maintenance surface
- [ ] **Add-ons / plugin system for community services** — high complexity; not worth building before core services are validated
- [ ] **`devtools remove <service>`** command — removing a service requires inverse merge logic; defer until users ask for it
- [ ] **Snapshot / backup tasks** (DDEV-style `ddev snapshot`) — DB dump task per service; v2 differentiator
- [ ] **Diagnose / status task** — `task devtools:status` showing which services are running; requires docker CLI integration

---

## Feature Prioritization Matrix

| Feature | User Value | Implementation Cost | Priority |
|---------|------------|---------------------|----------|
| Docker Compose service definitions (all 5) | HIGH | LOW | P1 |
| Interactive config prompts | HIGH | MEDIUM | P1 |
| Per-service Taskfile (up/down/logs/restart) | HIGH | LOW | P1 |
| Root Taskfile with includes | HIGH | LOW | P1 |
| Merge / no-clobber behavior | HIGH | MEDIUM | P1 |
| Named volumes | HIGH | LOW | P1 |
| Health checks | MEDIUM | LOW | P1 |
| Post-generation output + connection string | HIGH | LOW | P1 |
| AI skill definition (dual-runtime) | HIGH | MEDIUM | P1 |
| Service-specific shell access tasks | MEDIUM | LOW | P2 |
| `.env` / `.env.example` generation | MEDIUM | LOW | P2 |
| Version pinning prompts | MEDIUM | LOW | P2 |
| RabbitMQ Management UI port exposed | MEDIUM | LOW | P2 |
| Additional services (MinIO, Kafka, etc.) | MEDIUM | MEDIUM | P3 |
| `devtools remove` (inverse merge) | MEDIUM | HIGH | P3 |
| Snapshot / backup tasks | LOW | MEDIUM | P3 |
| Diagnose / status task | LOW | HIGH | P3 |

**Priority key:**
- P1: Must have for launch
- P2: Should have, add when possible
- P3: Nice to have, future consideration

---

## Competitor Feature Analysis

| Feature | Lando | DDEV | Devbox | DevContainers | Our Approach |
|---------|-------|------|--------|---------------|--------------|
| Service definitions (Redis, Postgres, etc.) | ✅ via `.lando.yml` services block | ✅ via add-on registry | ✅ via Nix packages (not Docker) | ✅ via features marketplace | ✅ Compose v2 — explicit, auditable files |
| Interactive setup | ❌ YAML config only | ❌ CLI flags only | ❌ config file | ❌ JSON config | ✅ AI conversation → prompted config |
| Task runner integration | ✅ `lando tooling:` custom commands | ✅ `ddev exec` + custom commands | ✅ shell hooks | ✅ `postCreateCommand` | ✅ Taskfile v3 with named tasks |
| Merge / additive services | ✅ add to `.lando.yml` | ✅ add-ons are additive | ✅ add to devbox.json | ✅ features are additive | ✅ no-clobber with per-file modularity |
| Zero dependency to run | ❌ requires Lando CLI | ❌ requires DDEV CLI | ❌ requires Devbox CLI | ❌ requires VS Code or CLI | ✅ writes static files; tool itself has no deps |
| AI-native triggering | ❌ | ❌ | ❌ | ❌ (Copilot in VS Code helps, not native) | ✅ core differentiator |
| Multi-AI-runtime | ❌ | ❌ | ❌ | ❌ | ✅ Copilot CLI + Claude/MCP |
| Reverse proxy / HTTPS | ✅ Traefik built-in | ✅ mkcert + router | ❌ | ❌ | ❌ deliberate anti-feature |
| Framework/CMS recipes | ✅ Drupal, WordPress, etc. | ✅ PHP frameworks | ❌ | ✅ via features | ❌ deliberate anti-feature |
| DB shell access | ✅ `lando db-import/export` | ✅ `ddev mysql`, `ddev sequelace` | Manual | Partial | ✅ P2: per-service CLI tasks |
| Health checks | Implicit (via Docker) | Implicit | ❌ | Partial | ✅ explicit in Compose |
| Named volumes / persistence | ✅ | ✅ | ✅ | ✅ | ✅ |
| Output connection strings | ✅ `lando info` | ✅ `ddev describe` | ❌ | ❌ | ✅ printed after generation |

**Key insight:** Every established tool requires installing its own CLI. Our tool has **no install step** — the AI skill generates plain files that any developer can read, edit, and run with only Docker and Task (both already present in the target dev environment). The AI is the "CLI."

---

## Sources

- **Taskfile v3 schema and includes documentation:** https://taskfile.dev/docs/guide.md (verified live, HIGH confidence)
- **DDEV feature set:** https://github.com/ddev/ddev README (verified live, HIGH confidence)
- **Lando features page:** https://lando.dev/features (HTML SPA, verified metadata, MEDIUM confidence — rendered content not parseable)
- **Devbox:** https://www.jetpack.io/devbox (training data + SPA, MEDIUM confidence — Nix-based, not Docker)
- **DevContainers spec:** https://containers.dev (training data, MEDIUM confidence — well-known VS Code standard)
- **Docker Compose v2 conventions** (named volumes, healthcheck syntax, no `version:` key): training data cross-referenced with PROJECT.md constraints, HIGH confidence
- **Default port conventions** (Redis 6379, PG 5432, MySQL 3306, Mongo 27017, RabbitMQ 5672/15672): industry standard, HIGH confidence

---
*Feature research for: dev-tools AI skill toolkit (Docker Compose + Taskfile scaffolding)*
*Researched: 2025-01-26*
