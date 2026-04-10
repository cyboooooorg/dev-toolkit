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
