# Phase 2: Skill Core — Interactive Flow & Merge Logic — Discussion Log

> **Audit trail only.** Do not use as input to planning, research, or execution agents.
> Decisions are captured in CONTEXT.md — this log preserves the alternatives considered.

**Date:** 2026-04-10
**Phase:** 02-skill-core-interactive-flow-merge-logic
**Areas discussed:** SKILL.md instruction style, Template materialization, Root compose file strategy, First-run setup

---

## SKILL.md Instruction Style

| Option | Description | Selected |
|--------|-------------|----------|
| Scripted steps | Numbered instructions spell out each question, answer handling, and file operations | ✓ |
| Guided narrative | Describe goal and constraints, let AI figure out question order | |
| Hybrid | Numbered phases with each phase described at high level | |

**User's choice:** Scripted steps

---

| Option | Description | Selected |
|--------|-------------|----------|
| Yes — summary table | Show config summary and ask 'Write these files?' before proceeding | ✓ |
| No — write immediately | Write files as soon as questions are answered | |
| Optional — only if >3 params | Show summary only for complex installs | |

**User's choice:** Yes — show summary table and confirm before writing

---

| Option | Description | Selected |
|--------|-------------|----------|
| Done summary | List files written, next steps, warnings | ✓ |
| Silent success | Just write files, no output | |
| Minimal | Just say 'Done. Files written to .devtools/' | |

**User's choice:** Done summary with files, next steps, and warnings

---

| Option | Description | Selected |
|--------|-------------|----------|
| Inline defaults | 'Port? [default: 6379]' | ✓ |
| Accept on empty | Ask question, use default if enter pressed | |
| Show on request | Defaults only shown if user types '?' | |

**User's choice:** Inline defaults in the question text

---

| Option | Description | Selected |
|--------|-------------|----------|
| One service per invocation | Install one service per skill run | ✓ |
| Allow multi-service | 'add Redis and RabbitMQ' installs both | |
| Ask at end | 'Want to add another service?' after installing | |

**User's choice:** One service per invocation

---

## Template Materialization

| Option | Description | Selected |
|--------|-------------|----------|
| Read & substitute | AI reads template, substitutes {{PLACEHOLDER}} tokens | ✓ |
| Generate from metadata | AI reads metadata.json and generates from scratch | |
| Hybrid | Read metadata for Q&A, read template only at write time | |

**User's choice:** Read & substitute — AI reads compose template and substitutes tokens

---

| Option | Description | Selected |
|--------|-------------|----------|
| Skill repo path | SKILL.md states repo root path, AI reads templates relatively | ✓ |
| Hardcoded relative paths | SKILL.md lists exact paths per service | |
| Dynamic lookup | AI globs for templates at runtime | |

**User's choice:** Skill repo path (relative to SKILL.md location)

---

| Option | Description | Selected |
|--------|-------------|----------|
| Q&A/defaults only | Read metadata.json during Q&A phase, not at write time | ✓ |
| Both Q&A and write | Use metadata to determine tokens before substituting | |
| Skip metadata | SKILL.md hardcodes questions and token names | |

**User's choice:** Read metadata.json for Q&A and defaults only

---

| Option | Description | Selected |
|--------|-------------|----------|
| Replace token + env var | Compose gets ${REDIS_PORT}, .env gets REDIS_PORT=6379 | ✓ |
| Replace exactly | {{PORT}} → 6379 literal in compose file | |
| Two-pass | Template already has ${REDIS_PORT}, skill just writes .env | |

**User's choice:** Token resolves to env var reference in compose + actual value in .env

---

| Option | Description | Selected |
|--------|-------------|----------|
| Ask user | 'Also set up Taskfile tasks? [Y/n]' | ✓ |
| Always write both | Compose + Taskfile always written together | |
| Compose only | Taskfile optional, only on explicit request | |

**User's choice:** Ask user (default yes)

---

| Option | Description | Selected |
|--------|-------------|----------|
| Ask user | 'Also install monitoring? [y/N]' | ✓ |
| Install automatically | Monitoring added whenever a service is added | |
| Skip | Monitoring left for a future explicit invocation | |

**User's choice:** Ask user (default no)

---

| Option | Description | Selected |
|--------|-------------|----------|
| Append + ## REPLACED on conflict | Append new entries; comment conflicting with ## REPLACED; update .env.example | ✓ |
| Merge entries | Check key exists first; only write if missing | |
| Rewrite section | Manage per-service block inside .env | |

**User's choice:** Append entries; comment conflicting ones with ## REPLACED; also update .env.example with dummy values; notify user in done summary

---

## Root Compose File Strategy

| Option | Description | Selected |
|--------|-------------|----------|
| Per-service files + root compose with include: | Root compose.yml uses Docker Compose include: | ✓ |
| Single merged docker-compose.yml | All services in one unified file | |
| No root compose | Users interact only via Taskfile tasks | |

**User's choice:** Per-service files + root compose.yml with include:

---

| Option | Description | Selected |
|--------|-------------|----------|
| On first service install | Create compose.yml when first service is added | ✓ |
| Always | Create even if empty on initial run | |
| On demand | Create only when explicitly requested | |

**User's choice:** Create on first service install

---

| Option | Description | Selected |
|--------|-------------|----------|
| Append include: entry | Add new include: block to existing compose.yml | ✓ |
| Regenerate fully | Rewrite compose.yml by scanning *.compose.yml | |
| AI agent decides | Let agent pick safer approach | |

**User's choice:** Append include: entry

---

| Option | Description | Selected |
|--------|-------------|----------|
| compose.yml | Newer preferred Docker Compose filename | ✓ |
| docker-compose.yml | Legacy but widely supported | |
| devtools.compose.yml | Custom name to avoid conflicts | |

**User's choice:** compose.yml

---

| Option | Description | Selected |
|--------|-------------|----------|
| Also update Taskfile.yml includes: | Update both compose.yml and Taskfile.yml in same step | ✓ |
| Taskfile includes separate | Update Taskfile.yml independently | |
| Root Taskfile is fixed | Uses optional: true glob, no update needed | |

**User's choice:** Update both files atomically in same step

---

| Option | Description | Selected |
|--------|-------------|----------|
| .devtools/.gitignore scoped | Write .devtools/.gitignore that ignores .env | ✓ |
| Root .gitignore entry | Add .devtools/.env entry to project root .gitignore | |
| Mention in summary only | Don't write any .gitignore | |

**User's choice:** .devtools/.gitignore scoped to .devtools/

---

## First-Run Setup

| Option | Description | Selected |
|--------|-------------|----------|
| Ask user | 'Project name for Docker namespacing? [default: git repo name]' | ✓ |
| Infer from git | Silently use repo name | |
| Infer + confirm | 'I'll use myapp — OK?' | |

**User's choice:** Ask user with git repo name as default

---

| Option | Description | Selected |
|--------|-------------|----------|
| Only on first install | Skip if COMPOSE_PROJECT_NAME already in .devtools/.env | ✓ |
| Always ask | Confirm on every install | |
| Ask if missing | Ask if COMPOSE_PROJECT_NAME absent from .env | |

**User's choice:** Only on first install

---

| Option | Description | Selected |
|--------|-------------|----------|
| Announce it | 'Creating .devtools/ directory...' then proceed | ✓ |
| Silent creation | Just mkdir and proceed | |
| Ask first | 'No .devtools/ found. Create it? [Y/n]' | |

**User's choice:** Announce directory creation

---

| Option | Description | Selected |
|--------|-------------|----------|
| General pattern for all services with ui_companion | 'Enable UI? [y/N]' then 'Enable auth? [y/N] [default: N]' | ✓ |
| CONF-02 RabbitMQ only | Only RabbitMQ gets the extra UI port question | |
| No auth question | Just offer UI enable/disable | |

**User's choice:** Ask 'Enable UI?' for all services with ui_companion in metadata.json; if yes, also ask 'Enable auth on UI? [y/N] [default: N]'

---

| Option | Description | Selected |
|--------|-------------|----------|
| Keep as-is | Each service in its own named file (redis.compose.yml, redis.Taskfile.yml) | ✓ |
| Flatten to single files | All services merged into .devtools/docker-compose.yml | |
| User's choice | Ask at install time | |

**User's choice:** Per-service files — redis.compose.yml, redis.Taskfile.yml, etc.

---

## the agent's Discretion

- Token substitution for edge cases (null/optional params in metadata.json)
- Exact `include:` YAML format in compose.yml
- Exact `includes:` format in Taskfile.yml

## Deferred Ideas

- Prometheus scrape config generation per-project
- docker compose include: integration for projects with existing root compose.yaml
- Service-specific CLI tasks (redis:cli, mongo:shell)
- Version pinning recommendations / latest tag warnings
- Auto-detect existing stack from project files
