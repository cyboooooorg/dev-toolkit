# Phase 2: Skill Core — Interactive Flow & Merge Logic — Research

**Researched:** 2026-04-10
**Domain:** SKILL.md authoring, Docker Compose include: syntax, Taskfile includes:, token substitution flow
**Confidence:** HIGH (all findings sourced from actual files in this repo or verified reference SKILL.md files on this machine)

---

<user_constraints>
## User Constraints (from CONTEXT.md)

### Locked Decisions

- **D-01:** Scripted step-by-step numbered instructions — no AI improvisation of question order
- **D-02:** Configuration summary table before writing; `"Write these files? [Y/n]"` gate
- **D-03:** Done summary: files written, next-step commands, warnings
- **D-04:** Inline defaults per question: `Port? [default: 6379]`
- **D-05:** One service per invocation
- **D-06:** AI reads compose template and substitutes `{{PLACEHOLDER}}` tokens (see implementation note — templates already use `${ENV_VAR}` syntax; substitution required only for alias/multi-instance)
- **D-07:** Paths relative to where SKILL.md lives (skill repo root)
- **D-08:** `metadata.json` used for Q&A only; template file is write-time source of truth
- **D-09:** Two-step token resolution: user answer → `REDIS_PORT=6379` in `.env`; compose file retains `${REDIS_PORT}` references
- **D-10:** Ask `"Also set up Taskfile tasks? [Y/n]"` (default: yes)
- **D-11:** Ask `"Also install monitoring? [y/N]"` (default: no)
- **D-12:** Append to `.devtools/.env`; never overwrite
- **D-13:** Key conflict → `## REPLACED` inline comment + new value on next line
- **D-14:** `.devtools/.env.example` updated alongside `.env` (dummy values only)
- **D-15:** Per-service files + root `.devtools/compose.yml` with `include:` entries
- **D-16:** `compose.yml` created on first service; new services appended thereafter
- **D-17:** Root compose file named `compose.yml`
- **D-18:** Taskfile.yml includes updated atomically with compose.yml include
- **D-19:** First install announces `"Creating .devtools/ directory..."` — no confirmation
- **D-20:** `COMPOSE_PROJECT_NAME` asked once on first install; skipped if already in `.env`
- **D-21:** `.devtools/.gitignore` written on first install with only `.env`
- **D-22:** UI companion ask `"Enable UI companion? [y/N]"` for all services that have one
- **D-23:** If UI enabled → ask `"Enable auth on UI? [y/N]"` (default: no)
- **D-24:** If `.devtools/<service>.compose.yml` exists → ask: add alias or cancel
- **D-25:** Alias applied to filename, container name, volume name, and env var names
- **D-26:** Identical existing alias → report already installed, exit without writing
- **D-27:** Naming: `<service>.compose.yml`, `<service>.Taskfile.yml` (or `<service>-<alias>.*`)

### the agent's Discretion

- Token substitution approach for edge cases (null defaults, optional params like password)
- Exact `include:` YAML format (path format, relative paths)
- Exact `includes:` YAML format in Taskfile with `optional: true` and `{{.TASKFILE_DIR}}`

### Deferred Ideas (OUT OF SCOPE)

- Prometheus scrape config generation per-project
- `docker compose include:` integration for projects already having a root `compose.yaml` outside `.devtools/`
- Service-specific CLI tasks (`redis:cli`, `mongo:shell`)
- Version pinning recommendations / `latest` tag warnings
- Auto-detect existing stack from project files
</user_constraints>

<phase_requirements>
## Phase Requirements

| ID | Description | Research Support |
|----|-------------|------------------|
| CONF-01 | Ask port, version, credentials before writing | §Q&A Flow per Service; §Token Inventory |
| CONF-02 | RabbitMQ: ask about Management UI port | §Service-Specific Notes — RabbitMQ |
| CONF-03 | Each question provides sensible default | §Token Inventory (defaults column) |
| MERGE-01 | Check if service already exists before writing | §Merge Detection Logic |
| MERGE-02 | Existing service → ask: add alias or cancel | §Merge Detection Logic |
| MERGE-03 | Alias namespaces filenames, containers, volumes, env vars | §Alias Substitution Pattern |
| MERGE-04 | Identical alias → idempotent (no write, inform user) | §Merge Detection Logic |
</phase_requirements>

---

## Summary

Phase 2 delivers a single `SKILL.md` file. The SKILL.md contains a scripted, numbered-step Q&A flow — the AI follows it mechanically, not creatively. Research reveals the format used by existing GSD skills on this machine and maps every template token to its env var name and default.

**Critical implementation insight:** The compose template files already use `${ENV_VAR}` references (e.g., `${REDIS_PORT}`) — they do NOT contain `{{TOKEN}}` placeholders. For a standard (first-instance) install, the AI copies compose and Taskfile templates verbatim and writes user answers to `.env`. Token substitution (string replacement in the file content) is only required for multi-instance/alias installs, where container names, volume names, network names, and env var names must all be renamed.

**Primary recommendation:** Write SKILL.md as a single standalone file with all Q&A steps inline (no external workflow references). The `<process>` section contains numbered steps directly. Use `find-skills` SKILL.md and the `gsd-discuss-phase` SKILL.md as format references — the former for inline step style, the latter for frontmatter structure.

---

## Standard Stack

### Core Technologies

| Technology | Version | Purpose | Source |
|------------|---------|---------|--------|
| SKILL.md format | Current (2026) | AI skill definition — works with GitHub Copilot CLI and Claude | [VERIFIED: ~/.copilot/skills/] |
| Docker Compose Spec | v2 (no `version:` key) | Generated compose fragments and root compose.yml | [VERIFIED: all templates in compose-templates/] |
| Taskfile | v3 | Generated Taskfile fragments and root Taskfile.yml | [VERIFIED: taskfile-templates/root/Taskfile.yml] |
| `{{PLACEHOLDER}}` token notation | — | Q&A mapping in metadata.json (NOT used as literal template tokens in compose files) | [VERIFIED: metadata.json files] |

---

## SKILL.md Format

### Frontmatter Fields

[VERIFIED: ~/.copilot/skills/gsd-discuss-phase/SKILL.md and others]

```yaml
---
name: add-service
description: Add a Docker service (Redis, RabbitMQ, PostgreSQL, MySQL, MongoDB) to any project
argument-hint: "<service-name> [--alias <alias>]"
allowed-tools: Read, Write, Bash, AskUserQuestion
---
```

| Field | Required | Notes |
|-------|----------|-------|
| `name` | Yes | Kebab-case, matches the skill's directory name |
| `description` | Yes | AI runtimes use this to match user intent — enumerate all 5 services explicitly |
| `argument-hint` | No | Shows in help/autocomplete |
| `allowed-tools` | Yes | `Read` (template files), `Write` (.devtools/ files), `Bash` (check files exist, git remote), `AskUserQuestion` (interactive Q&A) |

### Section Structure

For the GSD SKILL.md style [VERIFIED: ~/.copilot/skills/]:

```
---
{frontmatter}
---

<objective>
What the skill does — 2–5 sentence summary.
</objective>

<context>
$ARGUMENTS
</context>

<process>
Step-by-step numbered instructions.
</process>

<success_criteria>
Bullet list — what "done" looks like.
</success_criteria>
```

The `<execution_context>` section (referencing external `@workflow.md` files) is a GSD toolkit pattern for skills that delegate to external workflows. The dev-tools SKILL.md should NOT use `<execution_context>` — all instructions live inline in `<process>`. This matches the `find-skills` SKILL.md pattern where all guidance is self-contained.

### Numbered Step Format

Based on GSD skills and `find-skills` SKILL.md pattern [VERIFIED]:

```markdown
<process>

## Step 1: Detect Installation State

Check if `.devtools/<service>.compose.yml` already exists:
- If file exists → Go to **Step 1a: Merge Detection**
- If file does not exist → Continue to **Step 2**

## Step 1a: Merge Detection

...numbered sub-steps...

## Step 2: First-Run Setup (if .devtools/ does not exist)

Announce: `"Creating .devtools/ directory..."`

...
</process>
```

**Key style conventions observed in reference SKILL.md files:**
- Steps use `##` headings (H2) within `<process>`
- Sub-steps use numbered lists or lettered bullets under the H2
- Conditional branches are `- If X → Go to Step Y`
- User-facing output is quoted: `"Creating .devtools/ directory..."`
- Question format: `Ask: "Port? [default: 6379]"`
- File write operations explicitly name source and destination paths
- The `$ARGUMENTS` variable passes the service name argument to the process

---

## Docker Compose `include:` Syntax

### Root `.devtools/compose.yml` Structure

[VERIFIED: copilot-instructions.md STACK.md section — "Use include: directive in the root compose.yaml"]
[ASSUMED: exact YAML indentation from Docker Compose v2 spec — not verified via live doc fetch]

```yaml
# .devtools/compose.yml
# Root compose file — includes per-service fragments
# Created on first service install; new services appended

include:
  - ./redis.compose.yml
  - ./postgres.compose.yml
```

**Alternative object form** (when `project_directory` or `env_file` override is needed):

```yaml
include:
  - path: ./redis.compose.yml
  - path: ./postgres.compose.yml
```

**Recommendation:** Use the inline list form (`- ./redis.compose.yml`) — simpler, widely supported in Docker Compose v2, and sufficient since all service files are in the same `.devtools/` directory. No `project_directory` override is needed because all compose fragments already reference `.env` as a relative path, which resolves from `.devtools/` correctly when included.

### Append Pattern

When adding the second+ service, the skill appends a new line to the existing `compose.yml`:

```
  - ./postgres.compose.yml     ← append this line
```

The AI reads the existing file, checks the `include:` block, and appends. No full rewrite needed.

### Key Constraint

The root `compose.yml` itself should NOT declare any `services:`, `volumes:`, or `networks:` — it is purely an aggregator using `include:`. Each service file declares its own resources.

---

## Taskfile `includes:` Syntax

### `.devtools/Taskfile.yml` (Root DevTools Taskfile)

[VERIFIED: taskfile-templates/root/Taskfile.yml]

```yaml
version: '3'

includes:
  redis:
    taskfile: ./redis.Taskfile.yml
    optional: true    # Silently ignored if Redis is not installed
  rabbitmq:
    taskfile: ./rabbitmq.Taskfile.yml
    optional: true
  postgres:
    taskfile: ./postgres.Taskfile.yml
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

**Critical:** `optional: true` was added in Taskfile v3.17. The root template already has this correct. [VERIFIED: taskfile-templates/root/Taskfile.yml]

### Per-Service Taskfile Uses `{{.TASKFILE_DIR}}`

[VERIFIED: taskfile-templates/redis/redis.Taskfile.yml]

```yaml
tasks:
  up:
    cmds:
      - docker compose -f {{.TASKFILE_DIR}}/redis.compose.yml up -d
```

`{{.TASKFILE_DIR}}` resolves to the directory containing the Taskfile at execution time — i.e., `.devtools/`. This makes tasks portable regardless of where `task` is invoked from. [VERIFIED: taskfile-templates/redis/redis.Taskfile.yml]

### Project Root Inclusion Pattern

[VERIFIED: taskfile-templates/root/Taskfile.yml comment header]

```yaml
# In the project's own root Taskfile.yml:
includes:
  devtools:
    taskfile: ./.devtools/Taskfile.yml
    optional: true
# Then: task devtools:redis:up
```

This is documented in the root Taskfile template header comment. The skill's write scope stops at `.devtools/` — it does NOT modify the project's root `Taskfile.yml` (per TMPL-08 requirement and D-07). Users add the `devtools:` include manually per the README.

### Append Pattern for New Service

When adding a new service to `.devtools/Taskfile.yml`, the skill appends a new entry to the `includes:` block:

```yaml
  postgres:            ← append this block
    taskfile: ./postgres.Taskfile.yml
    optional: true
```

The AI reads the existing file to check that the service isn't already in `includes:`.

---

## Token Inventory

All `{{TOKEN}}` names from metadata.json files mapped to their env var names and defaults.

[VERIFIED: compose-templates/*/metadata.json]

### Redis

| Token | env_var | Default | Notes |
|-------|---------|---------|-------|
| `{{PORT}}` | `REDIS_PORT` | `6379` | Standard Redis port |
| `{{VERSION}}` | `REDIS_VERSION` | `7` | Alpine image: `redis:7-alpine` |
| `{{PASSWORD}}` | `REDIS_PASSWORD` | `null` | Optional — Redis runs without password if empty |
| `{{UI_PORT}}` | `REDIS_UI_PORT` | `8001` | RedisInsight only when UI enabled |

### RabbitMQ

| Token | env_var | Default | Notes |
|-------|---------|---------|-------|
| `{{PORT}}` | `RABBITMQ_PORT` | `5672` | AMQP port |
| `{{UI_PORT}}` | `RABBITMQ_UI_PORT` | `15672` | Management UI always-on (bundled in image) |
| `{{VERSION}}` | `RABBITMQ_VERSION` | `3` | Image: `rabbitmq:3-management-alpine` |
| `{{USERNAME}}` | `RABBITMQ_USERNAME` | `admin` | |
| `{{PASSWORD}}` | `RABBITMQ_PASSWORD` | `null` | Required for RabbitMQ |

### PostgreSQL

| Token | env_var | Default | Notes |
|-------|---------|---------|-------|
| `{{PORT}}` | `POSTGRES_PORT` | `5432` | |
| `{{VERSION}}` | `POSTGRES_VERSION` | `16` | |
| `{{USERNAME}}` | `POSTGRES_USER` | `postgres` | |
| `{{PASSWORD}}` | `POSTGRES_PASSWORD` | `null` | Required |
| `{{DB_NAME}}` | `POSTGRES_DB` | `app` | |
| `{{UI_PORT}}` | `POSTGRES_UI_PORT` | `8085` | pgAdmin4 UI port |
| (no token) | `PGADMIN_DEFAULT_EMAIL` | `admin@local.dev` | pgAdmin login email — ask if UI enabled |
| (no token) | `PGADMIN_DEFAULT_PASSWORD` | `null` | pgAdmin login password — ask if UI enabled |

### MySQL / MariaDB

| Token | env_var | Default | Notes |
|-------|---------|---------|-------|
| `{{PORT}}` | `MYSQL_PORT` | `3306` | |
| `{{VERSION}}` | `MYSQL_VERSION` | `11` | MariaDB 11 default; MySQL 8 opt-in |
| `{{USERNAME}}` | `MYSQL_USER` | `app` | App-level user |
| `{{PASSWORD}}` | `MYSQL_PASSWORD` | `null` | App user password |
| `{{DB_NAME}}` | `MYSQL_DATABASE` | `app` | |
| (no token) | `MYSQL_ROOT_PASSWORD` | `null` | Root password — always ask; not in token map |
| `{{UI_PORT}}` | `MYSQL_UI_PORT` | `8080` | phpMyAdmin |

### MongoDB

| Token | env_var | Default | Notes |
|-------|---------|---------|-------|
| `{{PORT}}` | `MONGODB_PORT` | `27017` | |
| `{{VERSION}}` | `MONGODB_VERSION` | `7` | |
| `{{USERNAME}}` | `MONGO_INITDB_ROOT_USERNAME` | `admin` | Note: env var name differs from token name convention |
| `{{PASSWORD}}` | `MONGO_INITDB_ROOT_PASSWORD` | `null` | Required |
| `{{UI_PORT}}` | `MONGODB_UI_PORT` | `8081` | Mongo Express |

### MongoDB UI Auth (Extra Vars)

These are NOT in metadata.json tokens but ARE in `.env.example` — written when UI auth is enabled.

| env_var | Default in .env.example | Notes |
|---------|------------------------|-------|
| `MONGO_EXPRESS_BASICAUTH` | `false` | Set to `true` if user enables UI auth |
| `MONGO_EXPRESS_USER` | `{{PLACEHOLDER}}` | Only written if auth enabled |
| `MONGO_EXPRESS_PASSWORD` | `{{PLACEHOLDER}}` | Only written if auth enabled |

### Monitoring (No Tokens — Fixed Env Vars)

| env_var | Default (template) | Notes |
|---------|--------------------|-------|
| `GRAFANA_ADMIN_USER` | `admin` | Written to `.env` at monitoring install time |
| `GRAFANA_ADMIN_PASSWORD` | `admin` | Skill should offer to change this (per monitoring.compose.yml comment) |

Monitoring has `"parameters": []` in metadata.json — no user questions required by default. The SKILL.md can ask `"Grafana admin password? [default: admin]"` (D-11 implies monitoring is always optional).

### Global / First-Run Token

| Token | env_var | Default | Notes |
|-------|---------|---------|-------|
| `{{PROJECT_NAME}}` | `COMPOSE_PROJECT_NAME` | `<git repo name>` | Asked once on first install only (D-20) |

**Default derivation for `COMPOSE_PROJECT_NAME`:** [VERIFIED: CONTEXT.md specifics section]
```bash
basename $(git remote get-url origin 2>/dev/null || pwd) | sed 's/\.git$//'
```

---

## Q&A Flow Per Service

### Universal Questions (all services)

1. **Port?** `[default: <port from metadata>]`
2. **Image version?** `[default: <version from metadata>]`
3. **Password?** — if default is null, no default shown; if service can run without password (Redis), show `[optional — press enter to skip]`
4. **Enable UI companion? [y/N]** — only for services with `ui_companion` in metadata.json (all except monitoring)
   - If yes: **Enable auth on UI? [y/N]**
5. **Also set up Taskfile tasks? [Y/n]** (D-10)
6. **Also install monitoring? [y/N]** (D-11) — shown once; if yes, runs monitoring install flow after main service

### Service-Specific Q&A Additions

**RabbitMQ:**
- Ask **Username?** `[default: admin]`
- Ask **Password?** (required — no default)
- Management UI port is always-on. Ask: **Management UI port? [default: 15672]** — CONF-02 requires this question

**PostgreSQL:**
- Ask **Username?** `[default: postgres]`
- Ask **Password?** (required — no default)
- Ask **Database name?** `[default: app]`
- If UI enabled: ask **pgAdmin email?** `[default: admin@local.dev]` and **pgAdmin password?**

**MySQL / MariaDB:**
- Ask **Username?** `[default: app]`
- Ask **Password?** (required — no default)
- Ask **Database name?** `[default: app]`
- Ask **Root password?** — no default, required (MYSQL_ROOT_PASSWORD has no token in metadata.json but must be written)
- Note: Image variant (MariaDB vs MySQL) — default is MariaDB 11. Skill may ask or document in summary

**MongoDB:**
- Ask **Username?** `[default: admin]`
- Ask **Password?** (required — no default)
- If UI enabled: ask **Enable Mongo Express auth? [y/N]**; if yes, ask username/password for `MONGO_EXPRESS_USER` / `MONGO_EXPRESS_PASSWORD`

**Monitoring (standalone, when D-11 is answered yes):**
- No version/port questions (fixed ports: Prometheus 9090, Grafana 3001)
- Ask: **Grafana admin password? [default: admin]** — surface the `## REPLACED` warning if grafana vars already in `.env`

---

## Architecture Patterns

### Write-Time Flow (Standard Install)

```
1. READ compose-templates/<service>/<service>.compose.yml   ← verbatim copy
2. READ taskfile-templates/<service>/<service>.Taskfile.yml ← verbatim copy
3. WRITE .devtools/<service>.compose.yml                    ← copied as-is
4. WRITE .devtools/<service>.Taskfile.yml                   ← copied as-is
5. APPEND .devtools/.env                                    ← KEY=value lines
6. APPEND .devtools/.env.example                            ← KEY=placeholder lines
7. APPEND/CREATE .devtools/compose.yml                      ← add include: entry
8. APPEND/CREATE .devtools/Taskfile.yml                     ← add includes: entry
```

**Critical insight:** The compose templates already use `${ENV_VAR}` references throughout. The skill copies them verbatim. No in-file token substitution is required for standard installs. [VERIFIED: redis.compose.yml, rabbitmq.compose.yml, mongodb.compose.yml]

### Write-Time Flow (Alias / Multi-Instance)

For `redis-cache` alias, the AI must perform string substitution within the template content:

| What to replace | Old value | New value |
|----------------|-----------|-----------|
| Container name suffix | `-redis` | `-redis-cache` |
| Volume key name | `redis_data:` | `redis_cache_data:` |
| Volume reference | `redis_data` | `redis_cache_data` |
| Network key name | `redis_net:` | `redis_cache_net:` |
| Network reference | `redis_net` | `redis_cache_net` |
| `REDIS_PORT` env var name | `REDIS_PORT` | `REDIS_CACHE_PORT` |
| `REDIS_PASSWORD` env var name | `REDIS_PASSWORD` | `REDIS_CACHE_PASSWORD` |
| `REDIS_UI_PORT` env var name | `REDIS_UI_PORT` | `REDIS_CACHE_UI_PORT` |
| `REDIS_VERSION` env var name | `REDIS_VERSION` | `REDIS_CACHE_VERSION` |
| Service key in YAML | `redis:` | `redis-cache:` or `redis_cache:` (see note) |

**Note on YAML service key for alias:** Docker Compose service names cannot contain hyphens in all versions. Safe choice: `redis_cache:` (underscore). Container name can still be `${COMPOSE_PROJECT_NAME}-redis-cache` using hyphens.

### `.env` Append Format

```bash
# ── Redis ──────────────────────────────────────────
REDIS_PORT=6379
REDIS_VERSION=7
REDIS_PASSWORD=mysecretpassword
REDIS_UI_PORT=8001
```

### `.env` Conflict Format (D-13)

```bash
REDIS_PORT=6379 ## REPLACED
REDIS_PORT=6380
```

### `.devtools/compose.yml` Root File

Created on first service install:

```yaml
# .devtools/compose.yml
# Root compose aggregator — managed by dev-tools skill
# Do not edit manually; run the skill to add or remove services.

include:
  - ./redis.compose.yml
```

Subsequent services append a line:

```yaml
  - ./postgres.compose.yml
```

### `.devtools/Taskfile.yml`

Copied from `taskfile-templates/root/Taskfile.yml` on first service install (if task install confirmed). The root Taskfile already has all 6 service includes with `optional: true` — no per-install append needed. [VERIFIED: taskfile-templates/root/Taskfile.yml]

**Key insight:** The root Taskfile template already contains ALL service includes. The skill copies the whole file on first install. No per-service append needed for the root Taskfile. The Taskfile IS already idempotent via `optional: true`.

---

## Merge Detection Logic

### Detection Order (maps to D-24 → D-26)

```
Step 0 (pre-flight): 
  Does .devtools/<service>.compose.yml exist?
  
  NO  → Proceed with standard install flow
  YES → 
    Present: "<service> is already installed."
    Ask: "Add another instance with an alias, or cancel? [alias/cancel]"
    
    User enters alias (e.g. "cache"):
      Does .devtools/<service>-<alias>.compose.yml exist?
      YES → Report: "<service>-<alias> is already installed — nothing to do." → EXIT
      NO  → Proceed with alias install flow (set ALIAS=cache for all steps)
    
    User enters "cancel":
      EXIT without writing anything
```

### Idempotency Check

Two levels:
1. **Service level:** `.devtools/<service>.compose.yml` existence
2. **Alias level:** `.devtools/<service>-<alias>.compose.yml` existence

Both checks use `Bash` tool: `test -f .devtools/redis.compose.yml`

---

## Alias Substitution Pattern

For alias install, the SKILL.md step must instruct the AI to perform these substitutions throughout the copied template content:

```
SERVICE_SLUG = <service>-<alias>   (e.g., "redis-cache")
SERVICE_SNAKE = <service>_<alias>  (e.g., "redis_cache")  ← for YAML keys and env var segments
ENV_PREFIX = <SERVICE_UPPER>_<ALIAS_UPPER>  (e.g., "REDIS_CACHE")
```

All env var names become `REDIS_CACHE_PORT`, `REDIS_CACHE_VERSION`, etc.

Compose YAML service key becomes `redis_cache:` (underscore — Docker Compose compatible).

Container name: `${COMPOSE_PROJECT_NAME:-devtools}-redis-cache` (hypen OK in container names).

This substitution is the agent's discretion area but must be complete — every occurrence of the bare service name in env vars, container names, volume names, and network names.

---

## Service-Specific Notes

### RabbitMQ UI Companion

RabbitMQ uses `rabbitmq:*-management-alpine` — the management UI is always on. The `ui_companion` note in metadata.json says:
> "Management UI is bundled in rabbitmq:*-management image — always active on RABBITMQ_UI_PORT"

The skill should NOT ask "Enable UI companion?" for RabbitMQ — the UI is always present. Instead, just confirm/ask the UI port (CONF-02). The `profiles:` key in the RabbitMQ template does NOT include a `ui` profile service.

### Monitoring Has No Parameters

`metadata.json` for monitoring has `"parameters": []`. The skill install for monitoring:
1. No Q&A questions (except optional Grafana password)
2. Copy `monitoring.compose.yml` verbatim
3. Write `GRAFANA_ADMIN_USER=admin` and `GRAFANA_ADMIN_PASSWORD=<user answer>` to `.env`
4. No per-service Taskfile question (monitoring has its own Taskfile)

### PostgreSQL pgAdmin Credentials

`pgadmin_email` and `pgadmin_password` have `token: null` in metadata.json — they are not mapped to template tokens. But they ARE written to `.env`. These are asked only when UI companion is enabled:
- `PGADMIN_DEFAULT_EMAIL=admin@local.dev` (default in metadata.json: "admin@local.dev")
- `PGADMIN_DEFAULT_PASSWORD` (null default — must ask)

### MySQL Root Password

`root_password` has `token: null` in metadata.json. But `MYSQL_ROOT_PASSWORD` IS in the template and IS required by MariaDB/MySQL. The skill must ask for it even though it has no template token. Write it to `.env` as `MYSQL_ROOT_PASSWORD=<value>`.

### Redis Password Is Optional

Redis `password` default is `null`. Redis can run without authentication. The skill should present: `"Password? [optional — press enter to skip]"`. If skipped, write `REDIS_PASSWORD=` (empty string) to `.env`. The redis template has `--requirepass ${REDIS_PASSWORD}` which will be empty string — Redis treats empty password as no auth.

---

## Don't Hand-Roll

| Problem | Don't Build | Use Instead | Why |
|---------|-------------|-------------|-----|
| Compose include aggregation | Custom merge logic | Docker Compose `include:` at root level | Native Compose feature — handles multi-file compose correctly |
| Taskfile namespacing | Manual prefix in task names | Taskfile `includes:` with namespace key | Official Taskfile pattern — tasks become `redis:up` automatically |
| Credential isolation | Custom env management | `env_file: .env` already in templates | All compose templates already have this pattern |

---

## Common Pitfalls

### Pitfall 1: Overwriting `.devtools/Taskfile.yml` on Every Install

**What goes wrong:** Skill copies root Taskfile template on each service install, overwriting manual customizations.
**Why it happens:** The root template has all 6 services — feels like the source of truth.
**How to avoid:** Copy root Taskfile ONLY on first install (when `.devtools/Taskfile.yml` doesn't exist). On subsequent installs, the file already exists and `optional: true` handles missing service files gracefully — no append needed.
**Warning sign:** User reports losing custom tasks in `.devtools/Taskfile.yml` after second install.

### Pitfall 2: Missing `MYSQL_ROOT_PASSWORD` Question

**What goes wrong:** Skill asks for username/password but skips root password. MySQL/MariaDB fails to start.
**Why it happens:** `root_password` has `token: null` in metadata.json so it's easy to overlook.
**How to avoid:** Always include `MYSQL_ROOT_PASSWORD` question in the MySQL Q&A flow. It has no template token but IS required in `.env`. [VERIFIED: mysql.compose.yml uses `${MYSQL_ROOT_PASSWORD}`]

### Pitfall 3: Using `{{TOKEN}}` Format in Compose Template Files

**What goes wrong:** Agent expects compose files to have `{{PORT}}` but they already have `${REDIS_PORT}`.
**Why it happens:** D-06 describes "substituting {{PLACEHOLDER}} tokens" which could be misread as the template files containing `{{PORT}}`.
**How to avoid:** SKILL.md must be explicit: templates use `${ENV_VAR}` references throughout. The "substitution" is writing the `.env` file. Compose files are copied verbatim for standard installs.

### Pitfall 4: RabbitMQ UI Companion Ask

**What goes wrong:** Skill asks "Enable UI companion? [y/N]" for RabbitMQ but UI is always-on.
**Why it happens:** All other services have optional UI companions.
**How to avoid:** RabbitMQ handling: skip the "Enable UI?" question. Only ask about the UI port. Document in SKILL.md: `"Note: RabbitMQ management UI is always enabled (bundled in the management image)."` [VERIFIED: rabbitmq.compose.yml comment]

### Pitfall 5: Alias with Hyphens in YAML Service Keys

**What goes wrong:** Alias `redis-cache` becomes YAML service key `redis-cache:` — Docker Compose may reject hyphenated service names in some contexts (network auto-naming).
**Why it happens:** Natural to use hyphens in aliases.
**How to avoid:** YAML service key uses underscore: `redis_cache:`. Container name uses hyphen: `${COMPOSE_PROJECT_NAME}-redis-cache`. Filename uses hyphen: `redis-cache.compose.yml`. [ASSUMED — YAML key constraint; verify against Compose spec if needed]

### Pitfall 6: `.devtools/compose.yml` Write Order

**What goes wrong:** Skill writes `.devtools/compose.yml` before it writes the per-service file, making include: reference a non-existent file.
**Why it happens:** Compose.yml is updated as part of the same operation.
**How to avoid:** Write per-service file first, then update `compose.yml`. On failure, `compose.yml` still doesn't reference the broken/missing file.

---

## Code Examples

### SKILL.md Frontmatter (Recommended)

```yaml
---
name: add-service
description: Add a Docker dev service to this project. Supported services: Redis, RabbitMQ, PostgreSQL, MySQL/MariaDB, MongoDB. Writes Docker Compose and Taskfile configs to .devtools/.
argument-hint: "<service-name>"
allowed-tools: Read, Write, Bash, AskUserQuestion
---
```

[VERIFIED: format from ~/.copilot/skills/gsd-discuss-phase/SKILL.md]

### Root `.devtools/compose.yml`

```yaml
# .devtools/compose.yml
# Root compose aggregator — managed by dev-tools skill
# Add services: run the skill again for each service you want

include:
  - ./redis.compose.yml
```

[VERIFIED: Docker Compose v2 include: syntax documented in copilot-instructions.md stack section]

### Root `.devtools/Taskfile.yml` (from template)

[VERIFIED: taskfile-templates/root/Taskfile.yml — copied verbatim on first install]

```yaml
version: '3'

includes:
  redis:
    taskfile: ./redis.Taskfile.yml
    optional: true
  # ... other services
```

### `.devtools/.gitignore` (first install only)

```
.env
```

[VERIFIED: D-21]

### `.env` Section Format

```bash
# ── Redis ──────────────────────────────────────────────────────────────────────
# Default port: 6379 | Default version: 7 | Default UI port: 8001
REDIS_PORT=6379
REDIS_VERSION=7
REDIS_PASSWORD=
REDIS_UI_PORT=8001
```

[VERIFIED: compose-templates/.env.example — matches this exact format]

### Configuration Summary Table (D-02)

```markdown
## Configuration Summary — Redis

| Setting | Value |
|---------|-------|
| Service | redis |
| Port | 6379 |
| Version | 7 |
| Password | *** (set) |
| UI Companion | RedisInsight on port 8001 |
| UI Auth | Disabled |
| Taskfile tasks | Yes |
| Monitoring | No |

Files to be written:
- .devtools/redis.compose.yml
- .devtools/redis.Taskfile.yml
- .devtools/compose.yml (created, include added)
- .devtools/Taskfile.yml (created)
- .devtools/.env (appended)
- .devtools/.env.example (appended)
- .devtools/.gitignore (created)

Write these files? [Y/n]
```

---

## State of the Art

| Old Approach | Current Approach | When Changed | Impact |
|--------------|------------------|--------------|--------|
| `version: "3.9"` in compose files | No `version:` key | Compose Spec 2024+ | All templates already correct |
| `docker-compose` (v1, hyphen) | `docker compose` (v2, space) | 2023/Docker Desktop 4.x | All Taskfile templates use `docker compose` |
| Monolithic single compose file | `include:` aggregator + per-service fragments | Compose Spec 2.20+ | This is the D-15 pattern |
| `optional` added in Taskfile v3.17 | Standard for includes now | Taskfile v3.17 | Already in root template |

---

## Environment Availability

> Step 2.6: SKIPPED for SKILL.md authoring — no external runtime dependencies. The skill is a static Markdown file that gets interpreted by the AI at runtime. No tools need to be installed to write the SKILL.md itself.

---

## Validation Architecture

### Test Framework

| Property | Value |
|----------|-------|
| Framework | Manual inspection + AI invocation test |
| Config file | None — SKILL.md is not code |
| Quick run command | Inspect SKILL.md structure visually |
| Full suite command | Invoke skill in Copilot CLI / Claude: "add Redis to this project" |

### Phase Requirements → Test Map

| Req ID | Behavior | Test Type | Automated Command | File Exists? |
|--------|----------|-----------|-------------------|-------------|
| CONF-01 | Skill asks port, version, credentials before writing | manual-only | Invoke skill, verify no files written before confirmation | ❌ Wave 0 |
| CONF-02 | RabbitMQ: Management UI port question present | manual-only | Read SKILL.md, grep for `RABBITMQ_UI_PORT` or "Management UI" step | ❌ Wave 0 |
| CONF-03 | Each question has default in brackets | manual-only | grep `\[default:` in SKILL.md `<process>` section | ❌ Wave 0 |
| MERGE-01 | Check for existing service file before writing | manual-only | Read SKILL.md for `test -f .devtools/` step | ❌ Wave 0 |
| MERGE-02 | Existing service → ask alias or cancel | manual-only | Read SKILL.md for alias/cancel branch | ❌ Wave 0 |
| MERGE-03 | Alias namespaces all identifiers | manual-only | Read alias substitution step in SKILL.md | ❌ Wave 0 |
| MERGE-04 | Identical alias → no write + inform user | manual-only | Read SKILL.md for idempotency exit path | ❌ Wave 0 |

**Note:** Since SKILL.md is a static Markdown instruction file (not executable code), all validation is either structural inspection (grep/read) or live invocation testing with a real AI runtime. No automated test framework applies.

### Wave 0 Gaps

- [ ] Bash script to validate SKILL.md structure: check frontmatter fields exist, check all service token defaults present, check merge detection step exists
- [ ] Human verification checklist for Phase 2 success criteria (from ROADMAP.md)

---

## Security Domain

| ASVS Category | Applies | Standard Control |
|---------------|---------|-----------------|
| V2 Authentication | No | N/A — local dev tool |
| V3 Session Management | No | N/A |
| V4 Access Control | No | N/A |
| V5 Input Validation | Partial | SKILL.md instructs AI not to write until confirmed; no untrusted input path |
| V6 Cryptography | No | Credentials written in plaintext to `.env` — intentional for local dev; `.gitignore` protects them |

**Relevant security note:** The `.devtools/.env` file contains real credentials and must never be committed. The skill enforces this by writing `.devtools/.gitignore` with `.env` on first install (D-21). [VERIFIED: CONTEXT.md]

---

## Assumptions Log

| # | Claim | Section | Risk if Wrong |
|---|-------|---------|---------------|
| A1 | Docker Compose v2 `include:` accepts inline path strings (`- ./redis.compose.yml`) without `path:` key | Docker Compose include: syntax | Would need to use object form `- path: ./redis.compose.yml` instead — minor rework |
| A2 | YAML service keys cannot safely use hyphens for network auto-naming in Docker Compose | Alias Substitution Pattern (Pitfall 5) | Hyphens in service keys may work fine — skip underscore conversion |
| A3 | Redis runs without password when `REDIS_PASSWORD` is empty string in `--requirepass ${REDIS_PASSWORD}` | Token Inventory — Redis | Empty password may cause auth error; may need `command: redis-server` without `--requirepass` when no password set |

**Mitigation for A3:** The Redis template has `command: redis-server --requirepass ${REDIS_PASSWORD}`. Passing empty string `--requirepass ""` may reject connections. SKILL.md should note: if password is skipped, the AI may need to omit the `--requirepass` arg from the compose file. This is an alias-level template edit, not a standard copy. Agent discretion.

---

## Open Questions

1. **Redis password omission**
   - What we know: Template has `command: redis-server --requirepass ${REDIS_PASSWORD}`; Redis may reject empty password arg
   - What's unclear: Does Redis accept `--requirepass ""` as "no auth"?
   - Recommendation: SKILL.md step should say: "If no password entered, remove the `--requirepass ${REDIS_PASSWORD}` flag from the command line in the copied template"

2. **RabbitMQ UI companion ask**
   - What we know: RabbitMQ UI is always-on; metadata.json has `ui_companion` with a note saying it's bundled
   - What's unclear: Should the skill still ask "Enable UI?" (for consistency) or skip it?
   - Recommendation: Skip "Enable UI?" for RabbitMQ; explain in SKILL.md that management UI is always active. Ask only for UI port per CONF-02.

3. **MySQL variant question**
   - What we know: MySQL metadata has `default_variant: mariadb` and supports `mysql` opt-in
   - What's unclear: Should the skill ask "MariaDB (default) or MySQL 8?" per the metadata variants section?
   - Recommendation: The agent's discretion — suggest asking only for Apple Silicon users (where MySQL 8 requires `platform: linux/amd64`). Default to MariaDB silently, mention in summary.

---

## Sources

### Primary (HIGH confidence)
- `compose-templates/*/metadata.json` — all 6 metadata files, VERIFIED in this session
- `compose-templates/redis/redis.compose.yml` — template structure, token format, VERIFIED
- `compose-templates/rabbitmq/rabbitmq.compose.yml` — VERIFIED
- `compose-templates/mongodb/mongodb.compose.yml` — VERIFIED (includes mongo-express auth vars)
- `compose-templates/monitoring/monitoring.compose.yml` — VERIFIED
- `compose-templates/.env.example` — master env var list, 31 vars, VERIFIED
- `taskfile-templates/root/Taskfile.yml` — includes: pattern with optional: true, VERIFIED
- `taskfile-templates/redis/redis.Taskfile.yml` — {{.TASKFILE_DIR}} pattern, VERIFIED
- `taskfile-templates/rabbitmq/rabbitmq.Taskfile.yml` — VERIFIED
- `.planning/phases/02-skill-core-interactive-flow-merge-logic/02-CONTEXT.md` — all decisions D-01 through D-27
- `~/.copilot/skills/gsd-discuss-phase/SKILL.md` — SKILL.md frontmatter format reference, VERIFIED
- `~/.agents/skills/find-skills/SKILL.md` — inline step style reference, VERIFIED
- `copilot-instructions.md` — stack section with Docker Compose and Taskfile version confirmation

### Secondary (MEDIUM confidence)
- `~/.copilot/skills/gsd-next/SKILL.md`, `gsd-explore/SKILL.md` — additional format confirmation

### Tertiary (LOW confidence)
- Docker Compose `include:` inline path syntax (string vs object) — [ASSUMED] from training knowledge; official spec URL: https://docs.docker.com/compose/multiple-compose-files/include/

---

## Metadata

**Confidence breakdown:**
- Token inventory: HIGH — verified from all 6 metadata.json files
- SKILL.md format: HIGH — verified from 5+ reference SKILL.md files on this machine
- Compose include: syntax: MEDIUM — structure confirmed from stack docs in copilot-instructions.md; exact inline vs object form ASSUMED
- Taskfile includes: syntax: HIGH — verified from root Taskfile template
- Alias substitution: MEDIUM — logical derivation from D-25 decisions; no existing aliased template to verify against

**Research date:** 2026-04-10
**Valid until:** 2026-05-10 (stable tools; SKILL.md format stable; metadata.json will not change until new service added)
