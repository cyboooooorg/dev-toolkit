# dev-tools

> AI skills for adding Docker dev services to any project.

One command adds a fully configured Docker Compose service and Taskfile to your project.
Supports Redis, RabbitMQ, PostgreSQL, MySQL/MariaDB, and MongoDB.

## Installation

Run this once in your project root:

```bash
npx skills add Cyboooooorg/dev-tools
```

This installs the skill to all detected AI agents in your project
(GitHub Copilot, Claude Code, Cursor, and [more](https://agentskills.io)).

## Usage

Trigger the skill with natural language or a slash command:

```
add Redis to this project
```

```
/add-service postgres
```

The skill will ask for port, image version, and credentials before writing anything.
A confirmation summary is shown before any file is created.

## Supported Services

| Service | Description |
|---------|-------------|
| **Redis** | In-memory key-value store. Ideal for caching and session storage. |
| **RabbitMQ** | Message broker with optional Management UI (port 15672). |
| **PostgreSQL** | Relational database with named volume and health check. |
| **MySQL / MariaDB** | MySQL-compatible database (defaults to MariaDB on ARM64). |
| **MongoDB** | Document database with named volume and health check. |

## What Gets Written

Files are created in a `.devtools/` directory in your project root:

```
.devtools/
  compose.yml              # Root Docker Compose file (run with docker compose up)
  Taskfile.yml             # Root Taskfile with includes for all services
  <service>.compose.yml    # Docker Compose service fragment
  <service>.Taskfile.yml   # Tasks: up, down, logs, restart
  .env                     # Credentials (gitignored)
```

A `.devtools/Taskfile.yml` with `includes:` entries for all installed services is
created on first install. To use tasks from your project root, manually add
`includes: { devtools: .devtools/Taskfile.yml }` to your root `Taskfile.yml`.
Each run appends to `.devtools/.env` — existing services are never overwritten.

> **Note:** `.devtools/.env` is gitignored. Credentials stay on your machine.

## Multiple Instances

Running the skill for a service that already exists prompts you to add a named second
instance (e.g., `redis-cache`, `redis-session`). All files and env vars are namespaced
by alias. Re-running with the same alias is a no-op — safe to invoke twice.

