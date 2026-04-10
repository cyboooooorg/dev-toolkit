---
name: add-service
description: Add a Docker dev service to this project. Supported services: Redis, RabbitMQ, PostgreSQL, MySQL/MariaDB, MongoDB. Writes Docker Compose and Taskfile configs to .devtools/.
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

Resolve all paths below relative to the directory containing this SKILL.md file (the skill
repo root). The target project's CWD is separate — do NOT mix these paths.

| Service | Compose Template | Taskfile Template | Metadata |
|---------|-----------------|-------------------|----------|
| redis | compose-templates/redis/redis.compose.yml | taskfile-templates/redis/redis.Taskfile.yml | compose-templates/redis/metadata.json |
| rabbitmq | compose-templates/rabbitmq/rabbitmq.compose.yml | taskfile-templates/rabbitmq/rabbitmq.Taskfile.yml | compose-templates/rabbitmq/metadata.json |
| postgres | compose-templates/postgres/postgres.compose.yml | taskfile-templates/postgres/postgres.Taskfile.yml | compose-templates/postgres/metadata.json |
| mysql | compose-templates/mysql/mysql.compose.yml | taskfile-templates/mysql/mysql.Taskfile.yml | compose-templates/mysql/metadata.json |
| mongodb | compose-templates/mongodb/mongodb.compose.yml | taskfile-templates/mongodb/mongodb.Taskfile.yml | compose-templates/mongodb/metadata.json |
| monitoring | compose-templates/monitoring/monitoring.compose.yml | taskfile-templates/monitoring/monitoring.Taskfile.yml | compose-templates/monitoring/metadata.json |

Root Taskfile template: `taskfile-templates/root/Taskfile.yml`

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

Read the metadata file for the selected service from the skill repo (path from the service
registry in `<context>`):

```
compose-templates/${SERVICE}/metadata.json
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
