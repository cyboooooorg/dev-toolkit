# Pitfalls Research

**Domain:** AI skill toolkit — Docker Compose v2 + Taskfile v3 dev environment generator
**Researched:** 2025-07-14
**Confidence:** HIGH (Compose spec + Taskfile docs verified from official sources; skill format from live GitHub inspection)

---

## Critical Pitfalls

### Pitfall 1: Taskfile Namespace Collision on Include Injection

**What goes wrong:**
The skill tries to patch the user's `Taskfile.yml` by adding an `includes:` block for `.devtools/Taskfile.yml`. If the user already has an `includes:` entry using the same namespace key (e.g., `redis:`, `db:`), Task will parse the file with a duplicate key. YAML silently drops one of the duplicate keys — meaning either the user's include or the skill's include disappears with no error.

**Why it happens:**
YAML parsers treat duplicate mapping keys as last-wins (or first-wins, depending on implementation). Developers assume that appending `includes: redis: ...` to a Taskfile is safe without checking if `redis:` already exists in the `includes:` block.

**How to avoid:**
Before patching any Taskfile:
1. Parse `includes:` section and enumerate existing namespace keys
2. If `redis:` already exists, abort with a clear error: "Namespace 'redis' already used in your Taskfile.yml at line X"
3. Never generate task namespace names that match common generic words the user likely already has (`build`, `test`, `up`, `down`, `start`)
4. Scope namespaces under a prefix — e.g., `devtools-redis:` — or nest them: `devtools: .devtools/Taskfile.yml` with all services as sub-tasks

**Warning signs:**
- Generated `task redis:up` silently does nothing (user's own `redis:` namespace was overwritten)
- Taskfile schema validation passes but behaviour is wrong
- User reports "my existing redis tasks disappeared"

**Phase to address:** Phase implementing Taskfile patching / `.devtools/` directory setup

---

### Pitfall 2: Taskfile Included Files Must Use the Same Schema Version

**What goes wrong:**
Task enforces that all included Taskfiles use the **same `version:` value** as the root Taskfile. If the skill generates `version: '3'` in `.devtools/redis.yml` but the user's root `Taskfile.yml` uses `version: '3.17'` (or vice versa), Task exits with a version mismatch error when the user runs any task.

**Why it happens:**
The `version:` field in a Taskfile is a minimum Task CLI version requirement, not a compatibility range. Mismatched versions between included files and root are caught at parse time.
Source: https://taskfile.dev/taskfile-versions/ — "The included Taskfiles must be using the same schema version as the main Taskfile uses."

**How to avoid:**
During the interactive setup phase, read the user's existing `Taskfile.yml` `version:` field and mirror it exactly in all generated `.devtools/*.yml` files. Default to `version: '3'` only if no root Taskfile exists.

```yaml
# Generated .devtools/redis.yml - version copied from root
version: '3'   # <- mirrored from user's root Taskfile.yml
```

**Warning signs:**
- `task redis:up` fails immediately with "included Taskfile version mismatch"
- Works on one developer's machine (has newer Task) but not another's

**Phase to address:** Phase generating per-service Taskfiles

---

### Pitfall 3: YAML Anchors/Aliases Destroyed on Compose File Modification

**What goes wrong:**
If the skill reads and rewrites an existing Docker Compose file (to merge a new service into it), all YAML anchors (`&name`) and aliases (`*name`) are lost. `js-yaml`, Python `yaml`, and most YAML parsers resolve anchors during load; re-serializing the parsed object drops the anchor notation entirely. Users with DRY Compose files using `x-` extension fields and `<<: *merge` patterns lose their abstractions.

**Why it happens:**
YAML anchors are a serialization convenience — the in-memory object graph has no concept of them. Round-tripping through any standard YAML library expands all references.

**How to avoid:**
**Never modify the user's existing `docker-compose.yml` or `compose.yaml`.** Instead:
- Generate a self-contained `.devtools/docker-compose.yml` with only the devtools services
- Instruct the user (or their root Compose file) to `include:` the `.devtools/` compose file
- If patching is unavoidable, use line-level text manipulation instead of parse-and-rewrite

The safe merge strategy: write `.devtools/docker-compose.yml` as standalone, then add to the user's root Compose:
```yaml
# User's compose.yaml — add this block (skill appends, never rewrites)
include:
  - path: .devtools/docker-compose.yml
```

**Warning signs:**
- User reports their `<<: *common` patterns are gone after running the skill
- Compose file grew from 50 lines to 200 lines with all shared config duplicated inline

**Phase to address:** Phase implementing the merge/include strategy for existing Compose files

---

### Pitfall 4: Docker Compose `version:` Key Written Into Output Files

**What goes wrong:**
If the skill includes a `version: '3.9'` (or any version) at the top of generated Compose files, Docker Compose v2 CLI (current standard) emits a warning: "version is obsolete". Worse, if the skill generates `version: '2'` syntax, it may use deprecated fields that Docker Compose v2 rejects. 

This project explicitly requires Docker Compose v2 syntax.

**Why it happens:**
Training data and tutorials are full of `version: '3.9'` examples — it was required in Docker Compose v1. Generating it out of habit produces backwards-looking output.

**How to avoid:**
**Never include `version:` in generated Compose files.** The top-level `services:`, `volumes:`, and `networks:` blocks without a version field are the correct Docker Compose v2 format. Validate this in templates with a linter check.

```yaml
# WRONG — generates deprecation warnings
version: '3.9'
services:
  redis:
    image: redis:7

# CORRECT — Docker Compose v2 format
services:
  redis:
    image: redis:7
```

**Warning signs:**
- Generated files have `version:` in them
- `docker compose up` prints "version is obsolete" on every run

**Phase to address:** Template creation phase (day one — bake into template design)

---

### Pitfall 5: Hardcoded Credentials Committed to Version Control

**What goes wrong:**
The skill writes `POSTGRES_PASSWORD: devpassword` (or similar) directly into `.devtools/docker-compose.yml`. That file gets committed to git. Developers cargo-cult the "devpassword" pattern into staging or production because "it's just the password from the compose file."

**Why it happens:**
For local dev, hardcoded passwords seem harmless. The skill defaults to a simple value to avoid making the user set up credentials before running anything. The `.devtools/` directory gets committed because it contains the Taskfile and compose config the team shares.

**How to avoid:**
- Credential values go in `.devtools/.env` (gitignored by default, created by skill)
- Compose file references them via `${POSTGRES_PASSWORD}` interpolation
- Skill generates `.devtools/.env.example` with placeholder values — this IS committed
- Skill generates `.devtools/.env` with random dev values — this is NOT committed (auto-gitignore)

```yaml
# .devtools/docker-compose.yml — CORRECT pattern
services:
  postgres:
    image: postgres:16
    environment:
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}
      POSTGRES_USER: ${POSTGRES_USER:-devuser}
      POSTGRES_DB: ${POSTGRES_DB:-devdb}
```

```bash
# .devtools/.env (gitignored) — generated with random dev values
POSTGRES_PASSWORD=dev_f8a2c19e
POSTGRES_USER=devuser
POSTGRES_DB=devdb
```

**Warning signs:**
- `docker-compose.yml` contains literal password strings
- No `.env.example` file exists
- `git log --all --full-history -- .devtools/.env` shows `.env` was committed

**Phase to address:** Phase defining credential/secrets handling strategy (before first service template is written)

---

### Pitfall 6: Port Collisions With Locally-Running Services

**What goes wrong:**
The skill generates Compose configs with default ports (Redis: 6379, PostgreSQL: 5432, MySQL: 3306, MongoDB: 27017, RabbitMQ: 5672/15672). Many developers already have these services running locally on identical ports. When they run `task devtools-redis:up`, Docker fails to bind the port and the container exits immediately — confusing because the task "succeeded" (exit 0) but the service isn't running.

**Why it happens:**
Defaults are assumed to be safe. The skill is designed to be zero-config, which biases toward not asking the user anything. Port conflicts are invisible until runtime.

**How to avoid:**
- During the interactive configuration flow, **default to the standard port but explicitly ask**: "Redis port [6379]:"
- Warn in generated docs if the port matches a well-known default: "Note: port 6379 is the Redis default. If you have Redis running locally, use a different port."
- Consider prefixing default ports with a non-standard offset (e.g., 16379 for Redis dev instance) to avoid conflicts by default
- For the `docker compose up` Taskfile task, add `||` fallback that prints "Port already in use — check if service is already running"

**Warning signs:**
- `task redis:up` exits cleanly but `redis-cli -p 6379 ping` fails
- `docker ps` shows container exited immediately after start
- `docker compose logs redis` shows "bind: address already in use"

**Phase to address:** Interactive configuration phase (port question is required input, not optional)

---

### Pitfall 7: Taskfile `dir:` Mismatch — Included Tasks Run From Wrong Directory

**What goes wrong:**
Task's default behaviour is that tasks in included files run from the **root Taskfile's directory**, not from the included file's directory. If `.devtools/redis.yml` uses relative paths (e.g., `./docker-compose.yml`), they resolve against the project root — not against `.devtools/`. The compose file is not found.

**Why it happens:**
Developers assume tasks in a file run from that file's directory (like `make` or shell scripts). Taskfile's include mechanism doesn't work that way by default.

**How to avoid:**
Set `dir:` in the include declaration or use the `{{.TASKFILE_DIR}}` built-in variable in all relative paths within generated Taskfiles:

```yaml
# .devtools/Taskfile.yml — use TASKFILE_DIR for all relative paths
version: '3'

tasks:
  up:
    dir: '{{.TASKFILE_DIR}}'
    cmds:
      - docker compose -f docker-compose.yml up -d
```

Or in the user's root include:
```yaml
includes:
  devtools:
    taskfile: .devtools/Taskfile.yml
    dir: .devtools    # Forces all tasks in this include to run from .devtools/
```

**Warning signs:**
- `task devtools:up` fails with "no such file or directory: docker-compose.yml"
- Works when user `cd .devtools && task up` but not from project root

**Phase to address:** Taskfile template design phase

---

### Pitfall 8: AI Skill `description` Too Generic — Triggers on Wrong Prompts

**What goes wrong:**
If the skill description is "help with Docker services" or "manage developer tools", the AI invokes it for requests that have nothing to do with adding devtools services — like "how do I optimize my Docker image?" or "help me debug my docker-compose networking." This wastes the user's time and produces confusing irrelevant output.

Conversely, if the description is too narrow ("add Redis to this project"), the skill is never invoked for "set up a message queue" (RabbitMQ) or "I need a database" (PostgreSQL).

**Why it happens:**
Skill descriptions are used by the AI runtime as the primary signal for tool invocation. Bad descriptions are the skill equivalent of a function with the wrong name — technically present but never called correctly.

**How to avoid:**
- Description must enumerate the key trigger phrases AND the services covered:
  ```yaml
  description: >
    Adds development service infrastructure to this project. Use when the user 
    asks to add Redis, RabbitMQ, PostgreSQL, MySQL, MongoDB, or similar services 
    to their dev environment. Writes Docker Compose and Taskfile configurations 
    into .devtools/.
  ```
- Test the description against 10+ realistic prompts: "add a cache", "I need postgres", "set up a message broker", "add a database for local dev", "configure redis for development"
- Do NOT include descriptions that trigger for production infrastructure requests (Kubernetes, cloud providers)

**Warning signs:**
- Skill never triggers when users say "I need a cache"
- Skill triggers for production infrastructure questions
- AI uses wrong skill when user mentions "database"

**Phase to address:** Skill definition design phase (first milestone)

---

### Pitfall 9: Skill Not Idempotent — Running Twice Corrupts State

**What goes wrong:**
User runs "add Redis" → skill creates `.devtools/redis.yml` and `.devtools/docker-compose.yml` with Redis service. User runs "add Redis" again (accidentally or testing). Skill appends a duplicate `redis:` service block to `docker-compose.yml`. Docker Compose throws a parse error: "service 'redis' already defined."

**Why it happens:**
The skill writes files without first checking if the service already exists. Write operations are assumed to be safe to repeat.

**How to avoid:**
Before writing any service config:
1. Check if `.devtools/docker-compose.yml` exists and contains a service named `redis`
2. If it does, output: "Redis is already configured in .devtools/docker-compose.yml. Use `task redis:up` to start it."
3. If user explicitly wants to reconfigure, offer `--force` / "replace existing" flow

**Warning signs:**
- `docker compose up` fails with "service X already defined"
- `.devtools/docker-compose.yml` has duplicate service blocks
- Running the skill twice produces different file sizes

**Phase to address:** All phases that write files — establish "check before write" as a design rule

---

### Pitfall 10: Volume Names Collide Across Multiple Projects

**What goes wrong:**
The skill generates generic volume names: `redis_data`, `postgres_data`. These are namespaced under Docker's project name (defaults to the directory name). If a developer has two projects named `app` or both in a `~/dev/project` dir, volumes from one project contaminate another: `app_postgres_data` is shared, corrupting database state.

**Why it happens:**
Docker Compose prefixes named volumes with the project name automatically — but the project name is often the same across similar projects ("app", "backend", "api"). Developers don't notice until they see stale data or schema mismatches.

**How to avoid:**
- In the interactive config, ask for or auto-derive a project-specific prefix
- Use the `.devtools/docker-compose.yml` `name:` top-level field to set an explicit, unique project name based on the repo name
- Name volumes with a service-specific prefix: `${COMPOSE_PROJECT_NAME:-myapp}_redis_data`

```yaml
# .devtools/docker-compose.yml
name: ${DEVTOOLS_PROJECT_NAME:-myapp}-devtools   # explicit, unique

volumes:
  redis_data:    # will become: myapp-devtools_redis_data
```

**Warning signs:**
- `docker volume ls` shows volumes shared across projects
- Postgres schema from project A appears in project B's database
- `docker compose down -v` in one project destroys another project's volumes

**Phase to address:** Volume and network naming strategy phase

---

## Technical Debt Patterns

| Shortcut | Immediate Benefit | Long-term Cost | When Acceptable |
|----------|-------------------|----------------|-----------------|
| Hardcode `version: '3'` in all generated Taskfiles | Simpler code | Breaks if user has mismatched version; cryptic error messages | Never — always read and mirror the user's version |
| Write passwords directly in compose YAML | Zero-friction setup | Passwords committed to git; habit transfer to production | Never |
| Assume `.devtools/` doesn't exist and create fresh | Simpler write logic | Overwrites user's custom modifications | Never — always check, merge or abort |
| Use `flatten: true` in Taskfile includes | Shorter task names (`task up` vs `task devtools:up`) | Silently overrides user's own `up`, `down`, `build` tasks | Never in user-facing includes |
| Generic volume names (`redis_data`) | Simple, memorable | Cross-project volume collisions for developers with many projects | Only if project name is always explicitly set |
| Single monolithic `.devtools/docker-compose.yml` | Simpler to reason about | Adding one service requires modifying the file (merge conflicts) | Acceptable for v1; consider per-service compose files for v2 |

---

## Integration Gotchas

| Integration | Common Mistake | Correct Approach |
|-------------|----------------|------------------|
| RabbitMQ | Only exposing AMQP port 5672, forgetting management UI port 15672 | Expose both 5672 and 15672; management UI is essential for local dev |
| PostgreSQL | Using `postgres:latest` image | Pin to specific major version (`postgres:16`); `latest` breaks when major version increments |
| MongoDB | No `--auth` flag for local dev | Add `MONGO_INITDB_ROOT_USERNAME/PASSWORD`; auth-free MongoDB is a security liability even locally |
| Redis | No `--requirepass` for local dev | Add `requirepass devpassword` via `command:` override; builds production-auth muscle memory |
| MySQL | Using `mysql:latest` with arm64 (Apple Silicon) | Use `mysql:8` or `platform: linux/amd64`; MySQL images have inconsistent ARM support |
| Taskfile `{{.TASKFILE_DIR}}` | Using bare relative paths in included Taskfile cmds | Always prefix Docker Compose `-f` flags with `{{.TASKFILE_DIR}}/docker-compose.yml` |
| Docker Compose `include:` | Assuming `.env` from parent flows into included file's project | Each `include:` gets its own project context; pass vars explicitly via `env_file:` |

---

## Performance Traps

| Trap | Symptoms | Prevention | When It Breaks |
|------|----------|------------|----------------|
| All services in one Docker network | Services can reach each other unintentionally across projects | Use isolated networks per `.devtools/` deployment; prefix network name | When 2+ projects are up simultaneously |
| No resource limits on dev containers | MongoDB or PostgreSQL consumes 4GB RAM on laptop | Add `deploy.resources.limits.memory` to each service | Single container scale; hits immediately on laptops |
| `restart: always` instead of `unless-stopped` | Containers restart after Docker daemon restart, even when dev doesn't want them | Use `restart: unless-stopped` as default | On every machine restart; unexpected service startup |
| Taskfile `--watch` on file-generating tasks | Task re-triggers itself in a loop when it generates files it watches | Never use watch mode on file-writing tasks | First use of watch mode |

---

## Security Mistakes

| Mistake | Risk | Prevention |
|---------|------|------------|
| `.devtools/.env` not gitignored | Credentials pushed to remote; team members expose dev secrets | Skill must write `.gitignore` entry for `.devtools/.env` as first operation |
| Redis with no password exposed on `0.0.0.0` | Any process on the machine can write to Redis; malware persistence | Always bind to `127.0.0.1` for port mappings: `"127.0.0.1:6379:6379"` |
| PostgreSQL/MySQL ports exposed on all interfaces | Database accessible network-wide on shared networks (coffee shop, office) | Use `127.0.0.1:5432:5432` host binding, never just `5432:5432` |
| Committing `.devtools/.env` in initial "just testing" commit | Secrets in git history even after `.gitignore` added | Pre-flight check: verify `.devtools/.env` is in `.gitignore` before writing it |
| Using `POSTGRES_HOST_AUTH_METHOD: trust` | No password required; anyone on Docker network can connect | Never generate `trust` auth; always use password-based auth |

---

## UX Pitfalls

| Pitfall | User Impact | Better Approach |
|---------|-------------|-----------------|
| Interactive questions with no defaults shown | User doesn't know what's acceptable; types wrong format | Always show default in brackets: `Redis port [6379]:` |
| Silent success on file write | User doesn't know what was created; can't verify | Print a summary of all files written with their paths after each operation |
| Skill fails without explaining how to retry | User left in partial state; confused | On any failure, print exact state of what was and wasn't written, and the command to clean up |
| Task names not predictable | User can't guess `task redis:up` vs `task devtools:redis:up` | Document all generated task names in a `README.md` written to `.devtools/` |
| No "what do I have?" command | User forgets what services are configured months later | Generate a `task devtools:status` task that lists all running devtools containers |

---

## "Looks Done But Isn't" Checklist

- [ ] **Taskfile version parity:** Generated `.devtools/*.yml` files actually use the same `version:` as the user's root `Taskfile.yml` — verify with `grep "^version:" .devtools/*.yml` vs `grep "^version:" Taskfile.yml`
- [ ] **Gitignore coverage:** `.devtools/.env` is in `.gitignore` (check root `.gitignore` AND `.devtools/.gitignore`) before `.env` is written
- [ ] **Credential externalization:** Zero literal passwords in `.devtools/docker-compose.yml` — all values use `${VAR}` interpolation, verify with `grep -E "PASSWORD|SECRET|KEY" .devtools/docker-compose.yml`
- [ ] **Idempotency check:** Running the skill twice on the same service produces the same files, not duplicates — verify by running twice and diffing
- [ ] **Port binding safety:** All port mappings use `127.0.0.1:HOST:CONTAINER` format, not bare `HOST:CONTAINER` — verify with `grep ports .devtools/docker-compose.yml`
- [ ] **No `version:` key in Compose files:** `grep "^version:" .devtools/docker-compose.yml` returns nothing
- [ ] **Taskfile `TASKFILE_DIR` used:** All compose `-f` paths in generated Taskfiles use `{{.TASKFILE_DIR}}` prefix
- [ ] **`.devtools/.env.example` exists and is committed:** Contains all variables referenced in `docker-compose.yml` with placeholder values

---

## Recovery Strategies

| Pitfall | Recovery Cost | Recovery Steps |
|---------|---------------|----------------|
| YAML anchors lost from user's compose file | HIGH | Restore from git: `git checkout -- docker-compose.yml`. Enforce "never modify user files" policy retroactively in skill. |
| Credentials committed to git | HIGH | `git filter-repo` or BFG Repo Cleaner to purge commit history; rotate all affected secrets; add `.devtools/.env` to `.gitignore` |
| Taskfile namespace collision broke user's tasks | MEDIUM | User manually removes the conflicting `includes:` entry; skill needs to re-run with a different namespace |
| Duplicate services in `docker-compose.yml` | LOW | Remove duplicate service blocks manually; add idempotency check to skill |
| Volume name collision between projects | MEDIUM | `docker compose down -v` in affected project; set unique `DEVTOOLS_PROJECT_NAME` in `.devtools/.env`; `docker compose up` to recreate with new name |
| Port conflict on startup | LOW | Update `.devtools/.env` with a free port; `task redis:down && task redis:up` |

---

## Pitfall-to-Phase Mapping

| Pitfall | Prevention Phase | Verification |
|---------|------------------|--------------|
| Taskfile namespace collision | Phase: Taskfile patching implementation | Automated test: inject into Taskfile with pre-existing `redis:` namespace; assert error, not silent overwrite |
| Taskfile version mismatch | Phase: Per-service Taskfile template design | Test with root Taskfile at version `3`, `3.17`, `3.27`; verify all includes use matching version |
| YAML anchor destruction | Phase: Compose merge strategy design | Test: create compose file with `<<: *merge` pattern, run skill, verify file is unchanged |
| `version:` key in Compose output | Phase: Template creation (Day 1) | CI lint: `grep "^version:" .devtools/*.yml` must return empty |
| Hardcoded credentials | Phase: Credential strategy (before first template) | CI check: `grep -E "PASSWORD|SECRET|TOKEN" .devtools/docker-compose.yml` returns empty |
| Port collisions | Phase: Interactive config flow design | Interactive test: verify port question is always asked, not defaulted silently |
| Taskfile `dir:` mismatch | Phase: Taskfile template design | Integration test: run `task devtools:up` from project root; verify compose file is found |
| Skill description too generic/narrow | Phase: Skill definition authoring | Manual test: 10 natural language prompts; verify correct skill invocation rate |
| Non-idempotent writes | Phase: File write logic (any service) | Automated test: run skill twice for same service; diff output files |
| Volume name collisions | Phase: Compose template design | Test: two projects with same name; verify volumes don't share data |

---

## Sources

- **Compose spec merge rules:** https://github.com/compose-spec/compose-spec/blob/main/13-merge.md
- **Compose spec include directive:** https://github.com/compose-spec/compose-spec/blob/main/14-include.md
- **Compose spec volumes:** https://github.com/compose-spec/compose-spec/blob/main/07-volumes.md
- **Compose spec interpolation:** https://github.com/compose-spec/compose-spec/blob/main/12-interpolation.md
- **Taskfile includes documentation:** https://taskfile.dev/guide/ (§ Including other Taskfiles)
- **Taskfile version semantics:** https://taskfile.dev/taskfile-versions/
- **Taskfile schema reference:** https://taskfile.dev/reference/schema/ (§ includes, flatten, dir)
- **MCP Tool schema:** https://github.com/modelcontextprotocol/specification/blob/main/schema/2025-03-26/schema.json
- **Agent skill format (live inspection):** https://github.com/hatch3r/hatch3r — `.github/skills/` and `skills/` directories
- **Compose spec `version:` obsolescence:** compose-spec/spec.md — "The top-level `version` property is defined by the Compose Specification for backward compatibility. It is only informative."
- **YAML anchor round-trip loss:** Verified via js-yaml `yaml.load()` + `yaml.dump()` round-trip test

---
*Pitfalls research for: AI skill toolkit generating Docker Compose v2 + Taskfile v3 dev environments*
*Researched: 2025-07-14*
