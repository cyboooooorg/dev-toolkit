# Phase 2: Skill Core — Interactive Flow & Merge Logic — Context

**Gathered:** 2026-04-10
**Status:** Ready for planning

<domain>
## Phase Boundary

Deliver a working `SKILL.md` that guides an AI through a scripted interactive flow — asking port, version, credential, UI, and monitoring questions — and then safely writes `.devtools/` files using token substitution from existing template files. Includes merge detection (no silent overwrites), alias-based multi-instance support, and idempotency on re-run.

**In scope:** `SKILL.md` authoring, `.devtools/` directory bootstrapping, per-service file writing, `compose.yml` root file creation and maintenance, `Taskfile.yml` root maintenance, `.env` and `.env.example` management, `.gitignore` seeding.

**Out of scope:** Cross-runtime wiring (SKILL-01–03), README, DIST-01/DIST-02 (Phase 3).

</domain>

<decisions>
## Implementation Decisions

### SKILL.md Instruction Style

- **D-01:** The SKILL.md uses **scripted step-by-step instructions** — numbered steps spell out each question, what to do with the answer, and the exact file operations. No guided narrative; no AI improvisation of question order.
- **D-02:** Before writing any files, the skill presents a **configuration summary table** (service, port, version, credentials masked) and asks `"Write these files? [Y/n]"`. Nothing is written until confirmed.
- **D-03:** After writing, the skill outputs a **done summary**: list of files written, next-step commands (e.g., `task redis:up`), and any warnings (e.g., `## REPLACED` conflicts found in .env).
- **D-04:** Defaults are communicated **inline** in each question: `Port? [default: 6379]`. User presses enter to accept.
- **D-05:** **One service per invocation.** The skill installs one service per run. User invokes the skill again to add another service.

### Template Materialization

- **D-06:** The AI **reads the compose template file** (`compose-templates/<service>/<service>.compose.yml`) and substitutes `{{PLACEHOLDER}}` tokens with user-supplied values. It does NOT generate files from scratch.
- **D-07:** SKILL.md refers to templates by **path relative to where the SKILL.md lives** (skill repo root). The AI resolves paths from the skill file's location.
- **D-08:** `metadata.json` is read **during Q&A only** (for parameter names, defaults, and env_var names). It is NOT used at write time — the template file is the write-time source of truth.
- **D-09:** **Token resolution is two-step:** `{{PORT}}` → `${REDIS_PORT}` in the compose file output; `REDIS_PORT=6379` written to `.devtools/.env`. Compose file always uses env var references, never literal values.
- **D-10:** The skill asks `"Also set up Taskfile tasks? [Y/n]"` (default: yes). If yes, writes the per-service Taskfile template (`<service>.Taskfile.yml`) to `.devtools/` and updates `Taskfile.yml` includes.
- **D-11:** The skill asks `"Also install monitoring (Grafana + Prometheus)? [y/N]"` (default: no). If yes, also installs the monitoring compose template.

### .env Management

- **D-12:** When `.devtools/.env` already exists, **append** new service env vars. Never overwrite.
- **D-13:** If a key conflict exists (key already present), **comment it out** with `## REPLACED` on the same line and append the new value on the next line. Include a warning in the done summary.
- **D-14:** `.devtools/.env.example` is updated alongside `.devtools/.env` — new service entries are appended with **dummy/placeholder values** (not real credentials).

### Root Compose File Strategy

- **D-15:** Per-service files + a **root `.devtools/compose.yml`** using Docker Compose `include:` entries. One `include:` block per installed service.
- **D-16:** `.devtools/compose.yml` is **created on first service install** (not before). On subsequent installs, the new service's `include:` block is **appended** to the existing file.
- **D-17:** Root compose file is named **`compose.yml`** (Docker Compose v2 preferred filename).
- **D-18:** When a new service's `include:` is added to `compose.yml`, the `.devtools/Taskfile.yml` `includes:` section is **updated in the same step** (atomic — both files updated together).

### First-Run Setup

- **D-19:** On first install (`.devtools/` does not exist), the skill announces `"Creating .devtools/ directory..."` and proceeds — no confirmation needed.
- **D-20:** `COMPOSE_PROJECT_NAME` is asked with `"Project name for Docker namespacing? [default: <git repo name>]"` — **only on first install**. If `.devtools/.env` already contains `COMPOSE_PROJECT_NAME`, this question is skipped.
- **D-21:** A `.devtools/.gitignore` is written on first install (if not already present) with `.env` as its only entry — scoped to `.devtools/` only.

### UI Companion & Auth Flow

- **D-22:** For every service that has a `ui_companion` in `metadata.json` (Redis, MySQL, PostgreSQL, MongoDB, RabbitMQ management UI), the skill asks **`"Enable UI companion? [y/N]"`** (default: no).
- **D-23:** If the user enables a UI companion, a follow-up asks **`"Enable auth on UI? [y/N] [default: N]"`**. Default is no auth (local dev context). If yes, skill sets the relevant auth env vars in `.env`.

### Multi-Instance & Idempotency

- **D-24:** Before writing any files, the skill checks if `.devtools/<service>.compose.yml` already exists. If it does — **stop and ask**: `"<service> is already installed. Add another instance with an alias, or cancel?"`.
- **D-25:** If adding another instance, the skill asks for an **alias** (e.g., `cache`, `session`). The alias is applied to: filename (`redis-cache.compose.yml`), container name (`${COMPOSE_PROJECT_NAME}-redis-cache`), volume (`${COMPOSE_PROJECT_NAME}_redis_cache_data`), and env var names (`REDIS_CACHE_PORT`, `REDIS_CACHE_PASSWORD`, etc.).
- **D-26:** **Idempotent on identical alias** — if `.devtools/<service>-<alias>.compose.yml` already exists, the skill reports `"<service>-<alias> already installed — nothing to do."` and exits without writing anything.

### Output File Naming

- **D-27:** Each service gets its own named file in `.devtools/`: `<service>.compose.yml` (or `<service>-<alias>.compose.yml` for multi-instance), `<service>.Taskfile.yml` (or `<service>-<alias>.Taskfile.yml`).

### the agent's Discretion

- Token substitution approach for edge cases (e.g., `null` defaults in metadata.json for optional params like password) — agent decides how to handle missing/null defaults gracefully.
- Exact `include:` YAML format in `compose.yml` (path format, relative paths) — agent follows Docker Compose v2 spec.
- Exact `includes:` YAML format in `Taskfile.yml` — agent follows Taskfile v3 spec with `optional: true` and `{{.TASKFILE_DIR}}`.

</decisions>

<canonical_refs>
## Canonical References

**Downstream agents MUST read these before planning or implementing.**

### Service Templates (source of truth for all template files)
- `compose-templates/redis/redis.compose.yml` — Redis compose template with {{PLACEHOLDER}} tokens
- `compose-templates/rabbitmq/rabbitmq.compose.yml` — RabbitMQ template
- `compose-templates/postgres/postgres.compose.yml` — PostgreSQL template
- `compose-templates/mysql/mysql.compose.yml` — MySQL/MariaDB template
- `compose-templates/mongodb/mongodb.compose.yml` — MongoDB template
- `compose-templates/monitoring/monitoring.compose.yml` — Grafana + Prometheus template

### Metadata (Q&A source of truth — parameters, defaults, env_var names)
- `compose-templates/redis/metadata.json`
- `compose-templates/rabbitmq/metadata.json`
- `compose-templates/postgres/metadata.json`
- `compose-templates/mysql/metadata.json`
- `compose-templates/mongodb/metadata.json`
- `compose-templates/monitoring/metadata.json`

### Taskfile Templates
- `taskfile-templates/redis/redis.Taskfile.yml`
- `taskfile-templates/rabbitmq/rabbitmq.Taskfile.yml`
- `taskfile-templates/postgres/postgres.Taskfile.yml`
- `taskfile-templates/mysql/mysql.Taskfile.yml`
- `taskfile-templates/mongodb/mongodb.Taskfile.yml`
- `taskfile-templates/monitoring/monitoring.Taskfile.yml`
- `taskfile-templates/root/Taskfile.yml` — Root Taskfile template (includes pattern)

### Phase 1 Locked Decisions
- `.planning/phases/01-templates-credential-foundation/01-CONTEXT.md` — Token format, volume naming, network naming, credential strategy, profile names

### Requirements
- `.planning/REQUIREMENTS.md` — CONF-01, CONF-02, CONF-03, MERGE-01–04 are the Phase 2 requirements

### Credential baseline
- `compose-templates/.env.example` — Master env var list; SKILL.md must keep `.devtools/.env.example` in sync

</canonical_refs>

<code_context>
## Existing Code Insights

### Reusable Assets
- All 5 service compose templates: ready to read-and-substitute at SKILL.md execution time
- All 5 service Taskfile templates: same — ready to copy with token substitution
- `compose-templates/.env.example`: defines all 31 env vars across 6 services — use as reference when generating `.devtools/.env.example` entries
- Root `taskfile-templates/root/Taskfile.yml`: defines the `includes:` pattern with `optional: true` to follow

### Established Patterns
- `{{PLACEHOLDER}}` token format is the ONLY substitution mechanism — no shell expansion in template files
- `env_file: .env` pattern is already set in all compose fragments — `.devtools/.env` is always relative to `.devtools/`
- Docker Compose profiles: `services` (default), `ui`, `monitoring` — already baked into templates
- `${COMPOSE_PROJECT_NAME:-devtools}` as the safe fallback for container/volume/network names
- All Taskfile `-f` flags use `{{.TASKFILE_DIR}}` for portability — maintain this pattern

### Integration Points
- Skill writes files into target project's `.devtools/` — the skill repo's templates are READ-ONLY source material
- `.devtools/compose.yml` root file will be a new file managed by the skill (not present in repo yet — created per target project)
- `.devtools/Taskfile.yml` root file is templated from `taskfile-templates/root/Taskfile.yml`

</code_context>

<specifics>
## Specific Ideas

- Token substitution flow: `{{PORT}}` → `${REDIS_PORT}` in compose output + `REDIS_PORT=6379` in `.env`. Two outputs from one question answer.
- `.env` conflict marker: `## REPLACED` as inline comment on old line, new value appended below. Visible and human-readable.
- `.devtools/.gitignore` contains only `.env` — keeps it minimal and focused.
- UI auth default explicitly OFF for local dev; only enabled if user says yes to "Enable auth on UI?".
- COMPOSE_PROJECT_NAME defaulted to git repo name (basename of `git remote get-url origin` or `basename $(pwd)`).

</specifics>

<deferred>
## Deferred Ideas

- Prometheus scrape config generation per-project (deferred from Phase 1 — still deferred; monitoring template is shipped as-is)
- `docker compose include:` integration for projects that already have a root `compose.yaml` outside `.devtools/` (v2 requirement — deferred to post-v1)
- Service-specific CLI tasks (`redis:cli`, `mongo:shell`) — in v2 requirements
- Version pinning recommendations / `latest` tag warnings — v2
- Auto-detect existing stack from project files — v2

None — discussion stayed within phase scope.

</deferred>

---

*Phase: 02-skill-core-interactive-flow-merge-logic*
*Context gathered: 2026-04-10*
