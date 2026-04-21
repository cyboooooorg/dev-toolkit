---
name: add-service
description: "Add a Docker dev service to this project. Supported services: Redis, RabbitMQ, PostgreSQL, MySQL/MariaDB, MongoDB. Writes Docker Compose and Taskfile configs to .devtools/<service>/ (one subfolder per service instance)."
argument-hint: "<service-name>"
allowed-tools: Read, Write, Bash, AskUserQuestion
---

<objective>
Add a Docker development service to the current project by writing configuration files to
`.devtools/<service>/`. Supported services: Redis, RabbitMQ, PostgreSQL, MySQL/MariaDB, MongoDB.

Each service gets its own subdirectory under `.devtools/` containing its compose fragment,
Taskfile, and per-service `.env` (credentials for that service only). Root `.devtools/.env`
holds only project-level vars (`COMPOSE_PROJECT_NAME`, `COMPOSE_PROFILES`).

The skill asks for port, image version, and credentials before writing anything. A
configuration summary is shown for confirmation before any file is created. The skill is
safe to re-run: it detects existing installs and either renames the existing instance (to free
the slot), installs an aliased second instance, or exits cleanly. All credentials land in
`.devtools/<service>/.env` which is gitignored via `**/.env` in `.devtools/.gitignore`.
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
test -f .devtools/${SERVICE}/${SERVICE}.compose.yml
```
- If the file **does not exist** → offer the optional alias prompt before proceeding (D-10–D-13):

  Ask:
  > `"Install with a custom alias? (optional — press Enter to install as '${SERVICE}'): _"`

  - **If the user presses Enter (no input):** Set `MODE=standard`. continue to **Step 2**. (D-11)

  - **If the user provides an alias:** Apply the same normalization and validation sequence
    used in Branch B of Step 1a (steps 2–4 of that loop):
    1. Normalize: lowercase, replace non-`[a-z0-9-]` with hyphens, collapse, trim.
    2. If empty after normalization: re-prompt once with
       `"Alias cannot be empty. Enter alias (e.g. 'cache' → ${SERVICE}-cache): _"`;
       if empty again: output `"Nothing written. Exiting."` and stop.
    3. If alias equals `${SERVICE}`: re-prompt with
       `"Alias must differ from the service name. Enter alias (e.g. 'cache' → ${SERVICE}-cache): _"`.

    On valid alias: set `ALIAS=<normalized>`, `SERVICE_SLUG=${SERVICE}-${ALIAS}`,
    `SERVICE_SNAKE=${SERVICE}_${ALIAS}`, `ENV_PREFIX=<SERVICE_UPPER>_<ALIAS_UPPER>`,
    `MODE=alias`. continue to **Step 2**. (D-12)

    *(No conflict check is needed here — the slot was just confirmed free by the `test -f`
    check above.)* (D-13: alias is locked before Step 5 Q&A)
- If the file **exists**:
  - If `${SERVICE}` is one of the 6 base service names (`redis`, `rabbitmq`, `postgres`, `mysql`, `mongodb`, `monitoring`) → go to **Step 1a** (the user may want to rename-before-add or install a new alias).
  - Otherwise (`${SERVICE}` is an alias slug — already installed) → output:
    > "`${SERVICE}` is already installed — nothing to do."
    Stop.

## Step 1a: Merge Detection — Rename-Before-Add Flow

The service is already installed. Inform the user:

> "`${SERVICE}` is already installed at `.devtools/${SERVICE}/`."

**First, offer to rename the existing instance** (D-15). This lets the user free up the
`${SERVICE}` slot for a fresh install, rather than being forced to install an alias:

Ask:
> `"Before adding a new instance, rename the existing '${SERVICE}' to free up the slot?
>  Enter a rename suffix (e.g. 'cache' → renames to '${SERVICE}-cache'), or press Enter to skip: [skip]"`

---

### Branch A: User provides a rename suffix (D-16, D-17, D-18)

If the user provides a non-empty rename suffix (e.g. `cache`):

1. **Set rename variables:**
   - `RENAME_ALIAS=<rename suffix>` (lowercase, letters/numbers/hyphens only)
   - `RENAME_SLUG=${SERVICE}-${RENAME_ALIAS}` (e.g. `redis-cache`)
   - `RENAME_SNAKE=${SERVICE}_${RENAME_ALIAS}` (e.g. `redis_cache`)
   - `RENAME_PREFIX=<SERVICE_UPPER>_<RENAME_ALIAS_UPPER>` (e.g. `REDIS_CACHE`)

2. **Rename the folder:**
   ```bash
   if [ -d ".devtools/${RENAME_SLUG}" ]; then
     echo "Error: .devtools/${RENAME_SLUG} already exists. Choose a different alias." && exit 1
   fi
   mv .devtools/${SERVICE} .devtools/${RENAME_SLUG}
   ```

3. **Rename files inside the folder:**
   ```bash
   mv .devtools/${RENAME_SLUG}/${SERVICE}.compose.yml .devtools/${RENAME_SLUG}/${RENAME_SLUG}.compose.yml
   ```
   If `${SERVICE}.Taskfile.yml` exists in the folder:
   ```bash
   mv .devtools/${RENAME_SLUG}/${SERVICE}.Taskfile.yml .devtools/${RENAME_SLUG}/${RENAME_SLUG}.Taskfile.yml
   ```

4. **Apply string substitutions inside the renamed compose and Taskfile** (same substitution
   logic as Step 12 alias installs, applied to the existing file content):
   - YAML service key: `${SERVICE}:` → `${RENAME_SNAKE}:`
   - Container name suffix: `-${SERVICE}` → `-${RENAME_SLUG}`
   - Volume key declaration: `${SERVICE}_data:` → `${RENAME_SNAKE}_data:`
   - Volume reference: `${SERVICE}_data` → `${RENAME_SNAKE}_data`
   - Network key declaration: `${SERVICE}_net:` → `${RENAME_SNAKE}_net:`
   - Network reference: `${SERVICE}_net` → `${RENAME_SNAKE}_net`
   - All env var names: `${SERVICE_UPPER}_*` → `${RENAME_PREFIX}_*`
     (e.g. `REDIS_PORT` → `REDIS_CACHE_PORT`, `REDIS_PASSWORD` → `REDIS_CACHE_PASSWORD`)
   - Compose filename reference in Taskfile: `${SERVICE}.compose.yml` → `${RENAME_SLUG}.compose.yml`

5. **Update `.devtools/compose.yml` include reference:**
   Read `.devtools/compose.yml`, find the line:
   ```
   - path: ${SERVICE}/${SERVICE}.compose.yml
   ```
   Replace with:
   ```
   - path: ${RENAME_SLUG}/${RENAME_SLUG}.compose.yml
   ```

6. **Update `.devtools/Taskfile.yml` include reference:**
   Read `.devtools/Taskfile.yml`, find the block:
   ```
     ${SERVICE}:
       taskfile: ${SERVICE}/${SERVICE}.Taskfile.yml
   ```
   Replace with:
   ```
     ${RENAME_SLUG}:
       taskfile: ${RENAME_SLUG}/${RENAME_SLUG}.Taskfile.yml
   ```

7. **Update `.devtools/${RENAME_SLUG}/.env` env var keys** (D-16):
   For each env var key in the file that starts with `${SERVICE_UPPER}_` (e.g. `REDIS_`):
   - Append ` ## REPLACED` as an inline comment on the old line.
   - Add a new line below with the renamed key: `${RENAME_PREFIX}_<SUFFIX>=<value>`.
   - Record each renamed key in a warnings list for the done summary.
   Example transformation:
   ```
   REDIS_PORT=6379 ## REPLACED
   REDIS_CACHE_PORT=6379
   REDIS_PASSWORD=secret ## REPLACED
   REDIS_CACHE_PASSWORD=secret
   ```

8. **Announce rename completion and proceed** (D-17):
   Output:
   > `"Existing ${SERVICE} renamed to ${RENAME_SLUG}. Now installing new ${SERVICE} instance..."`

9. **Proceed to Step 3** with `MODE=standard`, `SERVICE=${SERVICE}`. The `.devtools/${SERVICE}/`
   slot is now free — the new install will land there. (D-18: all rename operations above are
   completed before any new files are written.)

---

### Branch B: User skips rename (presses Enter)

If the user presses Enter without providing a suffix, enter the alias input loop (D-09):

Ask:
> `"Enter alias (e.g. 'cache' → ${SERVICE}-cache): _"`

For each alias input, apply the following validation sequence. **The loop repeats until
the user provides a valid free alias or types `cancel`.**

**1. Handle 'cancel':**
If the user types `cancel` → output `"Nothing written. Exiting."` and stop.

**2. Normalize the alias (D-04, D-05):**
- Lowercase the input.
- Replace any character that is not a letter (a–z), digit (0–9), or hyphen with a hyphen.
- Collapse consecutive hyphens into a single hyphen.
- Trim leading and trailing hyphens.
- Proceed silently with the normalized value — do **not** announce the normalization to the user.
  The normalized result will be visible in the done summary slug.

**3. Check for empty slug (D-06):**
If the normalized alias is empty (blank input, or input whose characters all normalize away):
- On the **first empty attempt**: re-prompt with:
  > `"Alias cannot be empty. Enter alias (e.g. 'cache' → ${SERVICE}-cache): _"`
  Increment an empty-attempt counter; return to step 1 of this loop.
- On the **second consecutive empty attempt**: output `"Nothing written. Exiting."` and stop.

**4. Check alias ≠ service name (D-07):**
If the normalized alias equals `${SERVICE}` (e.g. user typed `redis` for service `redis`):
- Re-prompt with:
  > `"Alias must differ from the service name. Enter alias (e.g. 'cache' → ${SERVICE}-cache): _"`
  Reset the empty-attempt counter; return to step 1 of this loop.

**5. Derive slug variables:**
- `ALIAS=<normalized alias>`
- `SERVICE_SLUG=${SERVICE}-${ALIAS}` (e.g. `redis-cache`)
- `SERVICE_SNAKE=${SERVICE}_${ALIAS}` (e.g. `redis_cache`) — used for YAML keys, volumes, networks.
- `ENV_PREFIX=<SERVICE_UPPER>_<ALIAS_UPPER>` (e.g. `REDIS_CACHE`) — used for env var names.

**6. Check if alias slug already installed (D-08, D-09):**
```bash
test -f .devtools/${SERVICE_SLUG}/${SERVICE_SLUG}.compose.yml
```
- If file **exists**: re-prompt with:
  > `"${SERVICE_SLUG} is already installed. Enter a different alias (or 'cancel'): _"`
  Reset the empty-attempt counter; return to step 1 of this loop.
- If file **does not exist**: set `MODE=alias` and continue to **Step 2**.

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
     **/.env
     ```
     This covers both the root `.devtools/.env` and all per-service `.devtools/<slug>/.env` files.
     Do NOT attempt to migrate any existing flat `.devtools/*.compose.yml` files — new installs
     always use the subfolder layout; old flat files continue to work as-is. (D-13: no migration)
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

1. **Expose this service on a host port? [y/N]** (default: N — internal Docker network only)
   - Store answer as `ANSWERS[host_port]=true/false`.
   - If **No (default)**: skip the Port question (question 2 below) entirely — proceed directly
     to question 3 (Image version). Do NOT ask for a port number. (D-04)
   - If **Yes**: continue to the Port question below. (D-06)

2. **Port?** `[default: <port from metadata>]` *(asked only if `ANSWERS[host_port]=true`)*
   - Store answer as `ANSWERS[port]`.

3. **Image version/tag?** `[default: <version from metadata>]`
   - Store answer as `ANSWERS[version]`.

---

### Per-service credential questions:

#### redis
*(Port default: `[default: 6379]`)*
4. `"Password? [optional — press enter to skip]"`
   - If user presses enter → `ANSWERS[password]=""` (empty string — Redis runs without auth).
   - Otherwise → `ANSWERS[password]=<user input>`.

#### rabbitmq
*(Port default: `[default: 5672]`)*
4. `"Username? [default: admin]"` → `ANSWERS[username]`
5. `"Password? (required — no default)"` → `ANSWERS[password]` (must be non-empty; re-ask if blank)
6. `"Management UI port? [default: 15672]"` → `ANSWERS[ui_port]`
   *(Note: RabbitMQ management UI is always-on — bundled in the rabbitmq:*-management image.
   Step 6 "Enable UI companion?" is SKIPPED for RabbitMQ.)*

#### postgres
*(Port default: `[default: 5432]`)*
4. `"Username? [default: postgres]"` → `ANSWERS[username]`
5. `"Password? (required — no default)"` → `ANSWERS[password]` (re-ask if blank)
6. `"Database name? [default: app]"` → `ANSWERS[db_name]`

#### mysql
*(Port default: `[default: 3306]`)*
4. `"Username? [default: app]"` → `ANSWERS[username]`
5. `"Password? (required — no default)"` → `ANSWERS[password]` (re-ask if blank)
6. `"Database name? [default: app]"` → `ANSWERS[db_name]`
7. `"Root password? (required — no default)"` → `ANSWERS[root_password]`
   *(MYSQL_ROOT_PASSWORD — required by MariaDB/MySQL even though it has token: null in metadata.json)*
8. *(MariaDB is the default image variant. MariaDB 11 ≈ MySQL 8 compatible. If user needs MySQL:
   they can override the version tag to use mysql:8 instead.)*

#### mongodb
*(Port default: `[default: 27017]`)*
4. `"Username? [default: admin]"` → `ANSWERS[username]`
5. `"Password? (required — no default)"` → `ANSWERS[password]` (re-ask if blank)

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

  #### redis (RedisInsight)
  *(RedisInsight authenticates via the Redis password already captured in Step 5.)*
  Ask: `"RedisInsight UI port? [default: 8001]"` → `ANSWERS[ui_port]`

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
  .devtools/.gitignore                     (first install only)
  .devtools/redis/redis.compose.yml        (or .devtools/redis-cache/redis-cache.compose.yml for alias)
  .devtools/redis/redis.Taskfile.yml       (if Taskfile=yes)
  .devtools/redis/.env                     (service credentials only)
  .devtools/compose.yml                    (created or appended)
  .devtools/Taskfile.yml                   (created on first install only)
  .devtools/.env                           (COMPOSE_PROJECT_NAME + COMPOSE_PROFILES, first install only)
  .devtools/.env.example                   (updated, first install only)
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
Apply these modifications to the in-memory content before writing (see write destination below).

**When `ANSWERS[host_port]=false` (user opted out of host port binding):**

Remove the main service port mapping from the in-memory compose content before writing.
Apply this modification AFTER any alias substitutions — so the env var names in the port
line are already in their final form (e.g. `${REDIS_CACHE_PORT}` for alias installs).

The main port line to remove for each service (match and delete the entire line including
leading whitespace and the trailing newline):

| Service  | Main port line to remove                                  |
|----------|-----------------------------------------------------------|
| redis    | `      - "127.0.0.1:${REDIS_PORT}:6379"`                 |
| rabbitmq | `      - "127.0.0.1:${RABBITMQ_PORT}:5672"`              |
| postgres | `      - "127.0.0.1:${POSTGRES_PORT}:5432"`              |
| mysql    | `      - "127.0.0.1:${MYSQL_PORT}:3306"`                 |
| mongodb  | `      - "127.0.0.1:${MONGODB_PORT}:27017"`              |

For alias installs, the env var prefix is `${ENV_PREFIX}` — match on the container port
number (`:6379`, `:5672`, `:5432`, `:3306`, `:27017`) in the main service container's
`ports:` block to identify the correct line regardless of prefix.

**After removing the main port line, check the remaining `ports:` block of the main
service container:**

- If the `ports:` block is now **empty** (no remaining lines under `ports:`):
  Remove the entire `ports:` block — including the `ports:` key line itself and any blank
  lines immediately following it. **Never write an empty `ports:` list.** (D-08)
  Services where this applies (only one port entry): redis, postgres, mysql, mongodb.

- If the `ports:` block still has **remaining entries**:
  Leave the remaining entries untouched. The `ports:` block stays with just the remaining
  lines. (D-09)
  Service where this applies: rabbitmq — after removing the AMQP port line
  (`127.0.0.1:${RABBITMQ_PORT}:5672`), the management UI port line
  (`127.0.0.1:${RABBITMQ_UI_PORT}:15672`) remains.

**Scope of this modification — do NOT touch:**
- UI companion container `ports:` blocks (redisinsight, pgadmin, phpmyadmin, mongo_express).
  These are ALWAYS host-mapped regardless of opt-out — the browser needs host access. (D-11)
- Monitoring exporter containers (`redis_exporter`, `rabbitmq_exporter`, etc.) — they already
  have no `ports:` block by design. (D-13)

Apply all modifications to the in-memory content before writing — never modify the fetched
template file. (D-10)

**For alias install (`MODE=alias`):** Before writing, perform all string substitutions in the
file content — see **Step 12** for the complete substitution map. Apply substitutions to the
in-memory copy before writing. Do NOT modify the original template file.

First, create the service subdirectory:
```bash
mkdir -p .devtools/${SERVICE_SLUG}
```
Where `${SERVICE_SLUG}` equals `${SERVICE}` for standard installs (e.g. `redis`) and
`${SERVICE}-${ALIAS}` for alias installs (e.g. `redis-cache`). (D-01/D-02)

Write to the target project:
```
.devtools/${SERVICE_SLUG}/<filename>.compose.yml
```
Where `<filename>` is `${SERVICE}` for standard, or `${SERVICE_SLUG}` for alias
(e.g. `.devtools/redis-cache/redis-cache.compose.yml`). (D-03/D-04)

**Note on `env_file: .env`:** The compose template contains `env_file: .env` — Docker Compose
`include:` resolves `env_file` paths relative to the included file's directory. Since the
compose file lives in `.devtools/${SERVICE_SLUG}/`, `env_file: .env` automatically resolves
to `.devtools/${SERVICE_SLUG}/.env`. No template changes are required. (D-07)

### 9b: Write Taskfile (if ANSWERS[taskfile]=true)

Fetch the per-service Taskfile template from the skill repo:
```bash
curl -fsSL "${SKILL_RAW_BASE}/taskfile-templates/${SERVICE}/${SERVICE}.Taskfile.yml"
```

**For alias install:** Perform string substitutions (see Step 12) on the in-memory copy.

Write to the target project (subfolder already created by Step 9a mkdir):
```
.devtools/${SERVICE_SLUG}/<filename>.Taskfile.yml
```
Where `<filename>` is `${SERVICE}` for standard, or `${SERVICE_SLUG}` for alias
(e.g. `.devtools/redis-cache/redis-cache.Taskfile.yml`). (D-03/D-04)

**Note:** Always write the per-service file (Step 9) BEFORE updating `compose.yml` (Step 11).
This ensures `compose.yml` never references a file that failed to write.

## Step 10: .env Management

### 10a: Write per-service .devtools/${SERVICE_SLUG}/.env  (D-05)

The service subdirectory was created by `mkdir -p` in Step 9a. Write service credentials
and config to `.devtools/${SERVICE_SLUG}/.env` (create if not exists, append if exists):

```
# ── <SERVICE> ───────────────────────────────────────
<ENV_VAR_NAME>=<value>
...
```

For **standard** install — env var names from metadata `env_var` field:

| Service | Env vars to write |
|---------|------------------|
| redis | REDIS_PORT, REDIS_VERSION, REDIS_PASSWORD (empty string if skipped), REDIS_UI_PORT (if `ANSWERS[ui_enabled]=true`) |
| rabbitmq | RABBITMQ_PORT, RABBITMQ_VERSION, RABBITMQ_USERNAME, RABBITMQ_PASSWORD, RABBITMQ_UI_PORT |
| postgres | POSTGRES_PORT, POSTGRES_VERSION, POSTGRES_USER, POSTGRES_PASSWORD, POSTGRES_DB, POSTGRES_UI_PORT (if UI enabled), PGADMIN_DEFAULT_EMAIL (if UI enabled), PGADMIN_DEFAULT_PASSWORD (if UI enabled) |
| mysql | MYSQL_PORT, MYSQL_VERSION, MYSQL_USER, MYSQL_PASSWORD, MYSQL_DATABASE, MYSQL_ROOT_PASSWORD, MYSQL_UI_PORT (if UI enabled) |
| mongodb | MONGODB_PORT, MONGODB_VERSION, MONGO_INITDB_ROOT_USERNAME, MONGO_INITDB_ROOT_PASSWORD, MONGODB_UI_PORT (if UI enabled), MONGO_EXPRESS_BASICAUTH (if UI enabled: `true` if auth, `false` if not), MONGO_EXPRESS_USER (if UI auth), MONGO_EXPRESS_PASSWORD (if UI auth) |

For **alias** install — prefix all env var names with `ENV_PREFIX_`:
e.g. `REDIS_CACHE_PORT`, `REDIS_CACHE_VERSION`, `REDIS_CACHE_PASSWORD`, etc.

**Conflict detection:** Before appending each key, check if the key already exists in
`.devtools/${SERVICE_SLUG}/.env`. If it does:
- Mark the old line: append ` ## REPLACED` as an inline comment on that line.
- Append the new value on the next line.
- Record the conflict in a warnings list for the done summary.

**Why per-service subfolder, not root .devtools/.env?** (D-05/D-06)
Credentials belong to their service. Deleting `.devtools/redis/` removes Redis completely —
including its credentials — without touching other services. The root `.devtools/.env` stays
minimal (project-level vars only), making it safe to inspect without risk of leaking service creds.

### 10a-root: Write root .devtools/.env (project-level vars only)  (D-06)

The root `.devtools/.env` holds ONLY project-level vars that Docker Compose needs when
running the root `compose.yml`. Write these on first install only (skip if already present):

```
COMPOSE_PROJECT_NAME=<value>
COMPOSE_PROFILES=services
```

- If `COMPOSE_PROJECT_NAME` is not yet in `.devtools/.env`, prepend it as the very first entry.
- Immediately after, if `COMPOSE_PROFILES` is not yet in `.devtools/.env`, add it.
- `COMPOSE_PROFILES=services` ensures `docker compose up` starts core service containers.
  Without it the default profile is empty and all containers are skipped.
- Never overwrite an existing `COMPOSE_PROFILES` entry — the user may have added `ui`
  or `monitoring` to it.
- Do NOT write any service credentials or ports to root `.devtools/.env`.

### 10b: Write .devtools/.env.example  (D-06, D-08)

The root `.devtools/.env.example` documents the project-level structure only. Update it on
first install (skip if `COMPOSE_PROJECT_NAME` is already present):

```
# .devtools/.env — project-level vars (safe to inspect; do NOT commit per-service .env files)
COMPOSE_PROJECT_NAME=myapp
COMPOSE_PROFILES=services

# Per-service credentials live in their service subfolders:
#   .devtools/redis/.env       — Redis credentials
#   .devtools/postgres/.env    — PostgreSQL credentials
#   .devtools/rabbitmq/.env    — RabbitMQ credentials
#   .devtools/mysql/.env       — MySQL credentials
#   .devtools/mongodb/.env     — MongoDB credentials
# All subfolder .env files are gitignored via **/.env in .devtools/.gitignore
```

Do NOT append per-service credential keys to `.devtools/.env.example`. Per-service vars are
self-documenting in `.devtools/${SERVICE_SLUG}/.env` which the skill creates.

### 10c: Monitoring .env (if ANSWERS[monitoring]=true)

Write monitoring credentials to `.devtools/monitoring/.env`
(the `mkdir -p .devtools/monitoring` will be run in Step 10d before writing these):

```
# ── Monitoring ──────────────────────────────────────
GRAFANA_ADMIN_USER=admin
GRAFANA_ADMIN_PASSWORD=<ANSWERS[grafana_password]>
```

Apply conflict detection for `GRAFANA_ADMIN_USER` and `GRAFANA_ADMIN_PASSWORD` within
`.devtools/monitoring/.env`. Do NOT write monitoring vars to root `.devtools/.env`.

### 10d: Monitoring files (if ANSWERS[monitoring]=true)

Copy monitoring templates verbatim (no token substitution needed):

1. Create monitoring subfolder and write the compose template:
   ```bash
   mkdir -p .devtools/monitoring
   curl -fsSL "${SKILL_RAW_BASE}/compose-templates/monitoring/monitoring.compose.yml"
   ```
   Write fetched content to `.devtools/monitoring/monitoring.compose.yml`.
   Then write `.devtools/monitoring/.env` as specified in Step 10c.

2. If `ANSWERS[taskfile]=true`, fetch and write the Taskfile:
   ```bash
   curl -fsSL "${SKILL_RAW_BASE}/taskfile-templates/monitoring/monitoring.Taskfile.yml"
   ```
   Write fetched content to `.devtools/monitoring/monitoring.Taskfile.yml`.

3. After writing `monitoring/monitoring.compose.yml` (step 1 above), update `.devtools/compose.yml`
   to add the monitoring include (using the same create-or-append logic as Step 11a):
   ```yaml
     - monitoring/monitoring.compose.yml
   ```

**Important:** Write `monitoring/monitoring.compose.yml` before updating `compose.yml` include —
same write-order rule as Step 9.

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
  - path: <slug>/<filename>.compose.yml
```
Where `<slug>/<filename>` is `redis/redis` (standard) or `redis-cache/redis-cache` (alias).
The `path:` key is used for explicitness; bare list entries are also valid Docker Compose v2.
(D-09)

**If it DOES exist (subsequent service install):**
Read the file, locate the `include:` block, and append a new line:
```yaml
  - path: <slug>/<filename>.compose.yml
```
Where `<slug>/<filename>` is `redis/redis` (standard) or `redis-cache/redis-cache` (alias).
Do NOT rewrite the entire file — only add the new entry. (D-09)

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

**If it DOES exist:** Do NOT overwrite it. The root Taskfile template contains
`optional: true` for all 6 service includes using subfolder-relative paths (e.g.
`redis/redis.Taskfile.yml`) — missing service files are silently skipped at runtime.
No append is needed for the Taskfile. (D-10)

**Exception — alias installs:** When `MODE=alias`, the `optional: true` pattern does NOT
cover the alias name (only the 6 base service names are pre-included). After writing the
alias Taskfile to `.devtools/${SERVICE_SLUG}/${SERVICE_SLUG}.Taskfile.yml`, append a new
`includes:` entry to `.devtools/Taskfile.yml`:

Locate the `includes:` block in `.devtools/Taskfile.yml` and append (using 2-space indent
to match the existing entries):

```yaml
  ${SERVICE_SLUG}:
    taskfile: ${SERVICE_SLUG}/${SERVICE_SLUG}.Taskfile.yml
    optional: true
```

Where `${SERVICE_SLUG}` is the alias slug (e.g. `redis-cache`). This makes
`task redis-cache:up` reachable. (D-10 alias extension)

**Important:** Never overwrite an existing `.devtools/Taskfile.yml`. Users may have
customized it. The `optional: true` pattern makes per-install updates unnecessary.

**Root file locations (D-08):** `.devtools/compose.yml`, `.devtools/Taskfile.yml`,
`.devtools/.gitignore`, and `.devtools/.env` all remain at the `.devtools/` root —
their locations do not change with the subfolder layout. Only per-service files
move into subfolders.

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
- `.devtools/${SERVICE}/${SERVICE}.compose.yml` → `.devtools/${SERVICE_SLUG}/${SERVICE_SLUG}.compose.yml`
- `logs -f ${SERVICE}` → `logs -f ${SERVICE_SNAKE}`
- `restart ${SERVICE}` → `restart ${SERVICE_SNAKE}`

## Step 13: Done Summary

Output a completion summary:

```
✓ Done! Files written:
```

List every file actually written (skip files that were skipped):
```
  .devtools/.gitignore                           (first install)
  .devtools/redis/redis.compose.yml              (or .devtools/redis-cache/redis-cache.compose.yml for alias)
  .devtools/redis/redis.Taskfile.yml             (if Taskfile=yes)
  .devtools/redis/.env                           (service credentials)
  .devtools/compose.yml                          (created or updated include entry)
  .devtools/Taskfile.yml                         (if first install)
  .devtools/.env                                 (COMPOSE_PROJECT_NAME + COMPOSE_PROFILES, first install only)
  .devtools/.env.example                         (updated, first install only)
```

If the install was preceded by a rename (Branch A of Step 1a), also list:
```
  .devtools/${RENAME_SLUG}/ (folder renamed from .devtools/${SERVICE}/)
  .devtools/${RENAME_SLUG}/${RENAME_SLUG}.compose.yml  (renamed + substituted)
  .devtools/${RENAME_SLUG}/${RENAME_SLUG}.Taskfile.yml (renamed + substituted, if existed)
  .devtools/${RENAME_SLUG}/.env                        (env keys renamed with ## REPLACED markers)
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

If any `## REPLACED` conflicts were found in `.devtools/${SERVICE_SLUG}/.env`, add a warning:
```
⚠ Warning: The following keys were already present in .devtools/${SERVICE_SLUG}/.env and have been replaced:
  - REDIS_PORT  (old value marked ## REPLACED)
Review .devtools/${SERVICE_SLUG}/.env if this is unexpected.
```

Always add the reminder:
```
⚠ Do not commit .devtools/<service>/.env (or any .devtools/*/.env) — they contain real credentials.
   .devtools/.gitignore already excludes them.
```

</process>

<success_criteria>
- Service compose file exists at `.devtools/<service>/<service>.compose.yml`
  (or `.devtools/<service>-<alias>/<service>-<alias>.compose.yml` for alias installs)
- Service Taskfile exists at `.devtools/<service>/<service>.Taskfile.yml` (if Taskfile=yes)
- `.devtools/<service>/.env` contains service credentials and config vars with real values
- `.devtools/.env` contains only `COMPOSE_PROJECT_NAME` and `COMPOSE_PROFILES` (no service vars)
- `.devtools/.gitignore` exists and contains both `.env` and `**/.env`
- `.devtools/compose.yml` exists and includes the new service's compose file as
  `redis/redis.compose.yml` (subfolder-relative path, no leading `./`)
- `.devtools/Taskfile.yml` exists (created or pre-existing)
- `task <service>:up` starts the service successfully
- Re-running the skill for the same service offers rename-first, then alias-or-cancel (no silent overwrite)
- Re-running with an identical alias produces "already installed — nothing to do"
- After a rename, `.devtools/${RENAME_SLUG}/` exists with renamed files and updated env keys;
  `.devtools/${SERVICE}/` no longer exists; `.devtools/compose.yml` and `Taskfile.yml` reference the renamed paths
</success_criteria>
