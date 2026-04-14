---
name: add-service
description: "Add a Docker dev service to this project. Supported services: Redis, RabbitMQ, PostgreSQL, MySQL/MariaDB, MongoDB. Writes Docker Compose and Taskfile configs to .devtools/."
argument-hint: "<service-name>"
allowed-tools: Read, Write, Bash, AskUserQuestion
---

<objective>
Add a Docker development service to the current project by writing configuration files to
`.devtools/`. Supported services: Redis, RabbitMQ, PostgreSQL, MySQL/MariaDB, MongoDB.

The skill asks for port, image version, and credentials before writing anything. A
configuration summary is shown for confirmation before any file is created. The skill is
safe to re-run: it detects existing installs and either exits cleanly or installs a
named second instance (alias). All credentials land in `.devtools/.env` which is gitignored.
</objective>

<context>
Service requested: $ARGUMENTS

## Service Registry

All template paths below are relative to `SKILL_RAW_BASE` (defined below) — fetch them
remotely at runtime. The target project's CWD is separate — do NOT mix these paths.

| Service | Compose Template | Taskfile Template | Metadata |
|---------|-----------------|-------------------|----------|
| redis | compose-templates/redis/redis.compose.yml | taskfile-templates/redis/redis.Taskfile.yml | compose-templates/redis/metadata.json |
| rabbitmq | compose-templates/rabbitmq/rabbitmq.compose.yml | taskfile-templates/rabbitmq/rabbitmq.Taskfile.yml | compose-templates/rabbitmq/metadata.json |
| postgres | compose-templates/postgres/postgres.compose.yml | taskfile-templates/postgres/postgres.Taskfile.yml | compose-templates/postgres/metadata.json |
| mysql | compose-templates/mysql/mysql.compose.yml | taskfile-templates/mysql/mysql.Taskfile.yml | compose-templates/mysql/metadata.json |
| mongodb | compose-templates/mongodb/mongodb.compose.yml | taskfile-templates/mongodb/mongodb.Taskfile.yml | compose-templates/mongodb/metadata.json |
| monitoring | compose-templates/monitoring/monitoring.compose.yml | taskfile-templates/monitoring/monitoring.Taskfile.yml | compose-templates/monitoring/metadata.json |

Root Taskfile template: `taskfile-templates/root/Taskfile.yml`

## Remote Template Base URL

Templates are fetched at runtime from the skill's source repository. All template reads
below use this base URL (so the skill works when installed via `npx skills add` — only
SKILL.md is installed locally):

```
SKILL_RAW_BASE=https://raw.githubusercontent.com/Cyboooooorg/dev-tools/main
```

Fetch any template with:
```bash
curl -fsSL "${SKILL_RAW_BASE}/<template-path>"
```

If `curl` is unavailable, use `wget -qO- "${SKILL_RAW_BASE}/<template-path>"`.

## Template Format Note

Compose templates already use `${ENV_VAR}` references (e.g. `${REDIS_PORT}`) — they do NOT
contain `{{TOKEN}}` placeholders. For standard installs, copy templates verbatim. String
substitution (renaming env vars, container names, volume names) is only required for
alias/multi-instance installs.
</context>

<process>

## Step 1: Detect Installation State

Determine the service name:
- If `$ARGUMENTS` is not empty, `SERVICE=$ARGUMENTS` (normalize to lowercase).
- If `$ARGUMENTS` is empty, skip to **Step 3** to ask the user to select a service.

Check whether the service is already installed:
```bash
test -f .devtools/${SERVICE}.compose.yml
```
- If the file **does not exist** → continue to **Step 2**.
- If the file **exists** → go to **Step 1a**.

## Step 1a: Merge Detection

The service is already installed. Inform the user:

> "`${SERVICE}` is already installed in `.devtools/`."

Ask: `"Add another instance with an alias, or cancel? [alias/cancel]"`

- If user answers **cancel** → output `"Nothing written. Exiting."` and stop.
- If user provides an **alias** (e.g. `cache`, `session`):
  - Set `ALIAS=<alias>` (lowercase, letters/numbers/hyphens only).
  - Set `SERVICE_SLUG=${SERVICE}-${ALIAS}` (e.g. `redis-cache`).
  - Set `SERVICE_SNAKE=${SERVICE}_${ALIAS}` (e.g. `redis_cache`) — used for YAML service keys, volume keys, network keys.
  - Set `ENV_PREFIX=<SERVICE_UPPER>_<ALIAS_UPPER>` (e.g. `REDIS_CACHE`) — used for env var names.
  - Check if alias already installed:
    ```bash
    test -f .devtools/${SERVICE_SLUG}.compose.yml
    ```
  - If file **exists** → output `"${SERVICE_SLUG} is already installed — nothing to do."` and stop. (MERGE-04)
  - If file **does not exist** → set `MODE=alias` and continue to **Step 3**.
- No alias set and no existing file → set `MODE=standard` and continue to **Step 2**.

## Step 2: First-Run Setup

**Only run this step if `.devtools/` does not yet exist.**

Check:
```bash
test -d .devtools
```
- If `.devtools/` **exists** → skip to **Step 3**.
- If `.devtools/` **does not exist** → proceed:

  1. Announce: `"Creating .devtools/ directory..."`.
  2. Create the directory:
     ```bash
     mkdir -p .devtools
     ```
  3. Write `.devtools/.gitignore` with exactly this content (do not add any other entries):
     ```
     .env
     ```
  4. Ask: `"Project name for Docker namespacing? [default: <derive from git remote or pwd>]"`
     - Derive the default: run `basename $(git remote get-url origin 2>/dev/null || echo $(pwd)) | sed 's/\.git$//'`
     - Set `COMPOSE_PROJECT_NAME=<user answer or default>`.
  5. Continue to **Step 3**.

  If `.devtools/.env` already exists and contains `COMPOSE_PROJECT_NAME`, skip the project
  name question entirely (D-20).

## Step 3: Service Selection

If `SERVICE` is already set from `$ARGUMENTS`, skip this step.

Ask the user:
> `"Which service would you like to add? (redis / rabbitmq / postgres / mysql / mongodb)"`

- Set `SERVICE=<user answer>` (normalize to lowercase).
- If the answer is not one of the five supported services, respond:
  > `"Service '<answer>' is not supported. Supported: redis, rabbitmq, postgres, mysql, mongodb."` and stop.

Return to **Step 1** to run the merge detection check for this service before continuing.

## Step 4: Read Service Metadata

Read the metadata file for the selected service from the skill repo using:

```bash
curl -fsSL "${SKILL_RAW_BASE}/compose-templates/${SERVICE}/metadata.json"
```

Extract:
- `parameters[]` — list of `{ name, default, env_var, token }` entries
- `ui_companion` — present or absent
- `exporter` — present or absent (used later for monitoring)

Store these for use in Step 5.

## Step 5: Per-Service Q&A

Ask each question individually using `AskUserQuestion`. Show the inline default on every
question as `[default: X]`. Accept the user's answer or use the default if they press enter.

### Universal questions (all services):

1. **Port?** `[default: <port from metadata>]`
   - Store answer as `ANSWERS[port]`.

2. **Image version/tag?** `[default: <version from metadata>]`
   - Store answer as `ANSWERS[version]`.

---

### Per-service credential questions:

#### redis
*(Port default: `[default: 6379]`)*
3. `"Password? [optional — press enter to skip]"`
   - If user presses enter → `ANSWERS[password]=""` (empty string — Redis runs without auth).
   - Otherwise → `ANSWERS[password]=<user input>`.

#### rabbitmq
*(Port default: `[default: 5672]`)*
3. `"Username? [default: admin]"` → `ANSWERS[username]`
4. `"Password? (required — no default)"` → `ANSWERS[password]` (must be non-empty; re-ask if blank)
5. `"Management UI port? [default: 15672]"` → `ANSWERS[ui_port]`
   *(Note: RabbitMQ management UI is always-on — bundled in the rabbitmq:*-management image.
   Step 6 "Enable UI companion?" is SKIPPED for RabbitMQ.)*

#### postgres
*(Port default: `[default: 5432]`)*
3. `"Username? [default: postgres]"` → `ANSWERS[username]`
4. `"Password? (required — no default)"` → `ANSWERS[password]` (re-ask if blank)
5. `"Database name? [default: app]"` → `ANSWERS[db_name]`

#### mysql
*(Port default: `[default: 3306]`)*
3. `"Username? [default: app]"` → `ANSWERS[username]`
4. `"Password? (required — no default)"` → `ANSWERS[password]` (re-ask if blank)
5. `"Database name? [default: app]"` → `ANSWERS[db_name]`
6. `"Root password? (required — no default)"` → `ANSWERS[root_password]`
   *(MYSQL_ROOT_PASSWORD — required by MariaDB/MySQL even though it has token: null in metadata.json)*
7. *(MariaDB is the default image variant. MariaDB 11 ≈ MySQL 8 compatible. If user needs MySQL:
   they can override the version tag to use mysql:8 instead.)*

#### mongodb
*(Port default: `[default: 27017]`)*
3. `"Username? [default: admin]"` → `ANSWERS[username]`
4. `"Password? (required — no default)"` → `ANSWERS[password]` (re-ask if blank)

## Step 6: UI Companion Prompt

**Skip this step entirely for `rabbitmq`** — its management UI is always-on (handled in Step 5).

For all other services (redis, postgres, mysql, mongodb) that have a `ui_companion` entry
in their metadata:

Ask: `"Enable UI companion? [y/N]"` (default: N)

- If **No** → set `ANSWERS[ui_enabled]=false` and skip to **Step 7**.
- If **Yes** → set `ANSWERS[ui_enabled]=true`.

  Ask: `"Enable auth on UI? [y/N]"` (default: N) → `ANSWERS[ui_auth]=<true/false>`

  **Service-specific follow-ups when UI is enabled:**

  #### postgres (pgAdmin credentials)
  Ask: `"pgAdmin login email? [default: admin@local.dev]"` → `ANSWERS[pgadmin_email]`
  Ask: `"pgAdmin login password? (required)"` → `ANSWERS[pgadmin_password]` (re-ask if blank)

  #### mysql (phpMyAdmin — no extra credentials needed beyond service creds)
  *(No additional questions — phpMyAdmin connects using MYSQL_USER / MYSQL_PASSWORD.)*

  #### mongodb (Mongo Express auth)
  If `ANSWERS[ui_auth]=true`:
    Ask: `"Mongo Express username? [default: admin]"` → `ANSWERS[me_username]`
    Ask: `"Mongo Express password? (required)"` → `ANSWERS[me_password]` (re-ask if blank)

  #### redis (RedisInsight — no extra credentials needed)
  *(RedisInsight authenticates via the Redis password already captured in Step 5.)*

## Step 7: Optional Feature Prompts

Ask: `"Also set up Taskfile tasks? [Y/n]"` (default: Y) → `ANSWERS[taskfile]=<true/false>`

Ask: `"Also install monitoring (Grafana + Prometheus)? [y/N]"` (default: N) → `ANSWERS[monitoring]=<true/false>`

If `ANSWERS[monitoring]=true`:
  Ask: `"Grafana admin password? [default: admin]"` → `ANSWERS[grafana_password]`
  *(Default is intentionally weak — surfaced here so user can change it.)*

## Step 8: Configuration Summary + Confirmation Gate

Display a summary table of all collected values. Mask passwords with `****`.

Example layout (adapt to the actual service and answers):

```
Service:        redis
Mode:           standard install  (or: alias install as "redis-cache")
─────────────────────────────────────────────
Port:           6379
Version:        7
Password:       ****
UI companion:   disabled
Taskfile tasks: yes
Monitoring:     no
─────────────────────────────────────────────
Files to write:
  .devtools/.gitignore              (first install only)
  .devtools/redis.compose.yml
  .devtools/redis.Taskfile.yml      (if Taskfile=yes)
  .devtools/compose.yml             (created or appended)
  .devtools/Taskfile.yml            (created on first install only)
  .devtools/.env                    (appended)
  .devtools/.env.example            (appended)
```

Ask: `"Write these files? [Y/n]"`

- If **No** → output `"Cancelled. Nothing written."` and stop.
- If **Yes** → continue to **Step 9**.

**Do NOT write any files before this confirmation.** Steps 9–13 perform all writes.

## Step 9: Write Per-Service Files

User confirmed "Yes" at Step 8. Proceed with file writes.

### 9a: Write Compose File

Fetch the compose template from the skill repo:
```bash
curl -fsSL "${SKILL_RAW_BASE}/compose-templates/${SERVICE}/${SERVICE}.compose.yml"
```

**For standard install (`MODE=standard`):** Copy the file content verbatim.

**Special case — Redis, no password:** If `SERVICE=redis` and `ANSWERS[password]` is empty
string (user pressed enter to skip), after loading the template content:
- Remove the `--requirepass ${REDIS_PASSWORD}` argument from the `command:` line.
- Remove `-a "${REDIS_PASSWORD}"` from the `healthcheck.test` command.
Write the modified content (not the raw template) to `.devtools/redis.compose.yml`.

**For alias install (`MODE=alias`):** Before writing, perform all string substitutions in the
file content — see **Step 12** for the complete substitution map. Apply substitutions to the
in-memory copy before writing. Do NOT modify the original template file.

Write to the target project:
```
.devtools/<filename>.compose.yml
```
Where `<filename>` is `${SERVICE}` for standard, or `${SERVICE_SLUG}` for alias
(e.g. `redis-cache`).

### 9b: Write Taskfile (if ANSWERS[taskfile]=true)

Fetch the per-service Taskfile template from the skill repo:
```bash
curl -fsSL "${SKILL_RAW_BASE}/taskfile-templates/${SERVICE}/${SERVICE}.Taskfile.yml"
```

**For alias install:** Perform string substitutions (see Step 12) on the in-memory copy.

Write to the target project:
```
.devtools/<filename>.Taskfile.yml
```
Where `<filename>` is `${SERVICE}` for standard, or `${SERVICE_SLUG}` for alias.

**Note:** Always write the per-service file (Step 9) BEFORE updating `compose.yml` (Step 11).
This ensures `compose.yml` never references a file that failed to write.

## Step 10: .env Management

### 10a: Write .devtools/.env

Append a section block to `.devtools/.env` (create the file if it does not exist):

```
# ── <SERVICE> ───────────────────────────────────────
<ENV_VAR_NAME>=<value>
...
```

For **standard** install — env var names from metadata `env_var` field:

| Service | Env vars to append |
|---------|-------------------|
| redis | REDIS_PORT, REDIS_VERSION, REDIS_PASSWORD (empty string if skipped) |
| rabbitmq | RABBITMQ_PORT, RABBITMQ_VERSION, RABBITMQ_USERNAME, RABBITMQ_PASSWORD, RABBITMQ_UI_PORT |
| postgres | POSTGRES_PORT, POSTGRES_VERSION, POSTGRES_USER, POSTGRES_PASSWORD, POSTGRES_DB, POSTGRES_UI_PORT (if UI enabled), PGADMIN_DEFAULT_EMAIL (if UI enabled), PGADMIN_DEFAULT_PASSWORD (if UI enabled) |
| mysql | MYSQL_PORT, MYSQL_VERSION, MYSQL_USER, MYSQL_PASSWORD, MYSQL_DATABASE, MYSQL_ROOT_PASSWORD, MYSQL_UI_PORT (if UI enabled) |
| mongodb | MONGODB_PORT, MONGODB_VERSION, MONGO_INITDB_ROOT_USERNAME, MONGO_INITDB_ROOT_PASSWORD, MONGODB_UI_PORT (if UI enabled), MONGO_EXPRESS_BASICAUTH (if UI enabled: `true` if auth, `false` if not), MONGO_EXPRESS_USER (if UI auth), MONGO_EXPRESS_PASSWORD (if UI auth) |

For **alias** install — prefix all env var names with `ENV_PREFIX_`:
e.g. `REDIS_CACHE_PORT`, `REDIS_CACHE_VERSION`, `REDIS_CACHE_PASSWORD`, etc.

**Conflict detection (D-13):** Before appending each key, check if the key already exists
in `.devtools/.env`. If it does:
- Mark the old line: append ` ## REPLACED` as an inline comment on that line.
- Append the new value on the next line.
- Record the conflict in a warnings list for the done summary.

If `COMPOSE_PROJECT_NAME` is not yet in `.devtools/.env`, prepend it as the very first entry:
```
COMPOSE_PROJECT_NAME=<value>
```

### 10b: Write .devtools/.env.example

Append the same keys to `.devtools/.env.example` (create if not exists) with **dummy/placeholder
values only** — never real credentials:

```
# ── <SERVICE> ───────────────────────────────────────
REDIS_PORT=6379
REDIS_VERSION=7
REDIS_PASSWORD=CHANGE_ME
REDIS_UI_PORT=8001
```

Use obvious placeholder strings for credentials: `CHANGE_ME`, `admin@example.com`, etc.

### 10c: Monitoring .env (if ANSWERS[monitoring]=true)

Append to `.devtools/.env`:
```
# ── Monitoring ──────────────────────────────────────
GRAFANA_ADMIN_USER=admin
GRAFANA_ADMIN_PASSWORD=<ANSWERS[grafana_password]>
```

Apply conflict detection (D-13) for `GRAFANA_ADMIN_USER` and `GRAFANA_ADMIN_PASSWORD`.

Append to `.devtools/.env.example` (dummy values only):
```
# ── Monitoring ──────────────────────────────────────
GRAFANA_ADMIN_USER=admin
GRAFANA_ADMIN_PASSWORD=CHANGE_ME
```

### 10d: Monitoring files (if ANSWERS[monitoring]=true)

Copy monitoring templates verbatim (no token substitution needed):

1. Fetch and write the compose template:
   ```bash
   curl -fsSL "${SKILL_RAW_BASE}/compose-templates/monitoring/monitoring.compose.yml"
   ```
   Write fetched content to `.devtools/monitoring.compose.yml` in the target project.

2. If `ANSWERS[taskfile]=true`, fetch and write the Taskfile:
   ```bash
   curl -fsSL "${SKILL_RAW_BASE}/taskfile-templates/monitoring/monitoring.Taskfile.yml"
   ```
   Write fetched content to `.devtools/monitoring.Taskfile.yml` in the target project.

3. After writing `monitoring.compose.yml` (step 1 above), update `.devtools/compose.yml`
   to add the monitoring include (using the same create-or-append logic as Step 11a):
   ```yaml
     - ./monitoring.compose.yml
   ```

**Important:** Write `monitoring.compose.yml` before updating `compose.yml` include —
same write-order rule as Step 9 (Pitfall 6).

## Step 11: Root File Management

### 11a: Root Compose File (.devtools/compose.yml)

Check if `.devtools/compose.yml` exists:

**If it does NOT exist (first service install):**
Create `.devtools/compose.yml` with this content:
```yaml
# .devtools/compose.yml
# Root compose aggregator — managed by dev-tools skill
# Do not edit manually; run the skill to add or remove services.

include:
  - ./<filename>.compose.yml
```
Where `<filename>` is `${SERVICE}` (standard) or `${SERVICE_SLUG}` (alias).

**If it DOES exist (subsequent service install):**
Read the file, locate the `include:` block, and append a new line:
```yaml
  - ./<filename>.compose.yml
```
Do NOT rewrite the entire file — only add the new entry.

### 11b: Root Taskfile (.devtools/Taskfile.yml)

Check if `.devtools/Taskfile.yml` exists:
```bash
test -f .devtools/Taskfile.yml
```

**If it does NOT exist:** Fetch and write the root Taskfile:
```bash
curl -fsSL "${SKILL_RAW_BASE}/taskfile-templates/root/Taskfile.yml"
```
Write the fetched content verbatim to `.devtools/Taskfile.yml`.

**If it DOES exist:** Do NOT overwrite it. The root Taskfile template already contains
`optional: true` for all 6 service includes — missing service files are silently skipped
at runtime. No append is needed.

**Important:** Never overwrite an existing `.devtools/Taskfile.yml`. Users may have
customized it. The `optional: true` pattern makes per-install updates unnecessary.

## Step 12: Alias Substitution (alias installs only)

**Skip this step entirely for standard installs (`MODE=standard`).**

For alias installs, perform the following string substitutions throughout the compose file
content (in memory, before writing) and the Taskfile content (in memory, before writing).

Variables set in Step 1a:
- `SERVICE_SLUG` = `${SERVICE}-${ALIAS}` (e.g. `redis-cache`) — used in filenames and container names
- `SERVICE_SNAKE` = `${SERVICE}_${ALIAS}` (e.g. `redis_cache`) — used in YAML keys, volumes, networks
- `ENV_PREFIX` = `<SERVICE_UPPER>_<ALIAS_UPPER>` (e.g. `REDIS_CACHE`) — used in env var names

### Compose file substitutions

| What | Old value (pattern) | New value |
|------|---------------------|-----------|
| YAML service key | `${SERVICE}:` | `${SERVICE_SNAKE}:` |
| Container name suffix | `-${SERVICE}` | `-${SERVICE_SLUG}` |
| Volume key declaration | `${SERVICE}_data:` | `${SERVICE_SNAKE}_data:` |
| Volume reference (under volumes:) | `${SERVICE}_data` | `${SERVICE_SNAKE}_data` |
| Network key declaration | `${SERVICE}_net:` | `${SERVICE_SNAKE}_net:` |
| Network reference (under networks:) | `${SERVICE}_net` | `${SERVICE_SNAKE}_net` |
| Env var names in content | `${SERVICE_UPPER}_PORT` | `${ENV_PREFIX}_PORT` |
| Env var names in content | `${SERVICE_UPPER}_VERSION` | `${ENV_PREFIX}_VERSION` |
| Env var names in content | `${SERVICE_UPPER}_PASSWORD` | `${ENV_PREFIX}_PASSWORD` |
| Env var names in content | `${SERVICE_UPPER}_USERNAME` | `${ENV_PREFIX}_USERNAME` |
| Env var names in content | `${SERVICE_UPPER}_UI_PORT` | `${ENV_PREFIX}_UI_PORT` |
| Other service-specific env vars | `${SERVICE_UPPER}_*` | `${ENV_PREFIX}_*` |

**Service key note:** YAML service keys must use underscore (`redis_cache:`) — hyphens in
Compose service names cause network auto-naming issues. Container names and filenames use
hyphen (`redis-cache`) — that is correct and expected.

### Taskfile substitutions

Replace all references to the compose filename and service name within task commands:
- `${SERVICE}.compose.yml` → `${SERVICE_SLUG}.compose.yml`
- `logs -f ${SERVICE}` → `logs -f ${SERVICE_SNAKE}`
- `restart ${SERVICE}` → `restart ${SERVICE_SNAKE}`

## Step 13: Done Summary

Output a completion summary:

```
✓ Done! Files written:
```

List every file actually written (skip files that were skipped):
```
  .devtools/.gitignore              (first install)
  .devtools/redis.compose.yml       (or redis-cache.compose.yml for alias)
  .devtools/redis.Taskfile.yml      (if Taskfile=yes)
  .devtools/compose.yml             (created or updated)
  .devtools/Taskfile.yml            (if first install)
  .devtools/.env                    (appended)
  .devtools/.env.example            (appended)
```

Next steps:
```
  Start service:    task redis:up
  With UI:          task redis:up-ui
  With monitoring:  task redis:up-monitoring
  Stop:             task redis:down
```

If the service was installed with an alias, show the aliased task names:
```
  Start service:    task redis-cache:up
```

If any `## REPLACED` conflicts were found in `.devtools/.env`, add a warning:
```
⚠ Warning: The following keys were already present in .devtools/.env and have been replaced:
  - REDIS_PORT  (old value marked ## REPLACED)
Review .devtools/.env if this is unexpected.
```

Always add the reminder:
```
⚠ Do not commit .devtools/.env — it contains real credentials.
   .devtools/.gitignore already excludes it.
```

</process>

<success_criteria>
- Service compose file exists at `.devtools/<service>.compose.yml` (or `<service>-<alias>.compose.yml`)
- Service Taskfile exists at `.devtools/<service>.Taskfile.yml` (if Taskfile=yes)
- `.devtools/.env` contains all service env vars with real values
- `.devtools/.env.example` contains all service env var names with dummy/placeholder values
- `.devtools/.gitignore` exists and contains `.env`
- `.devtools/compose.yml` exists and includes the new service's compose file
- `.devtools/Taskfile.yml` exists (created or pre-existing)
- `task <service>:up` starts the service successfully
- Re-running the skill for the same service asks alias-or-cancel (no silent overwrite)
- Re-running with an identical alias produces "already installed — nothing to do"
</success_criteria>
