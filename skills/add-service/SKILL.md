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
