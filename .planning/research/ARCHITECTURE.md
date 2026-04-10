# Architecture Research

**Domain:** AI skill toolkit / dev-tooling generator
**Researched:** 2025-01-10
**Confidence:** HIGH (based on direct inspection of live skill ecosystem)

## Standard Architecture

### System Overview

```
┌─────────────────────────────────────────────────────────────────┐
│                      AI Runtime Layer                            │
│  ┌──────────────────┐          ┌──────────────────┐             │
│  │ GitHub Copilot   │          │  Claude / MCP    │             │
│  │ CLI              │          │  Agent Runtime   │             │
│  │ .github/skills/  │          │  .agents/skills/ │             │
│  └────────┬─────────┘          └────────┬─────────┘             │
└───────────┼────────────────────────────┼─────────────────────────┘
            │  reads same SKILL.md       │
            └──────────────┬─────────────┘
                           ↓
┌─────────────────────────────────────────────────────────────────┐
│                    Skill Entry Point                             │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │  SKILL.md                                                  │  │
│  │  ─ YAML frontmatter (name, description, allowed-tools)    │  │
│  │  ─ Interactive flow instructions                           │  │
│  │  ─ References to service registry + templates             │  │
│  └─────────────────────────────────────────────┬─────────────┘  │
└────────────────────────────────────────────────┼────────────────┘
                                                 ↓
┌─────────────────────────────────────────────────────────────────┐
│                    Service Knowledge Layer                       │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │  Service Registry (embedded in SKILL.md or ref file)    │    │
│  │  ─ Redis: port 6379, no auth default, image redis:7     │    │
│  │  ─ RabbitMQ: port 5672, user/pass required              │    │
│  │  ─ PostgreSQL: port 5432, user/pass/db required         │    │
│  │  ─ MySQL: port 3306, root pass + db required            │    │
│  │  ─ MongoDB: port 27017, no auth default                 │    │
│  └─────────────────────────────────────────────────────────┘    │
└────────────────────────────────────────────────┬────────────────┘
                                                 ↓
┌─────────────────────────────────────────────────────────────────┐
│                    Template Library                              │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────────────┐   │
│  │ Compose      │  │ Per-service  │  │ Root Taskfile        │   │
│  │ fragments    │  │ Taskfiles    │  │ (includes: block)    │   │
│  │ per service  │  │ per service  │  │                      │   │
│  └──────┬───────┘  └──────┬───────┘  └──────────┬───────────┘   │
└─────────┼─────────────────┼───────────────────────┼─────────────┘
          │                 │                       │
          └─────────────────┴───────────────────────┘
                                    ↓
┌─────────────────────────────────────────────────────────────────┐
│                    Output: .devtools/ directory                  │
│  ┌──────────────────────┐   ┌──────────────────────────────┐    │
│  │  docker-compose.yml  │   │  Taskfile.yml (root)         │    │
│  │  (merged services)   │   │  includes: taskfiles/*.yml   │    │
│  └──────────────────────┘   └──────────────────────────────┘    │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  taskfiles/                                               │   │
│  │  ├── redis.yml  ├── postgres.yml  ├── rabbitmq.yml  ...  │   │
│  └──────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────┘
```

### Component Responsibilities

| Component | Responsibility | Implementation |
|-----------|----------------|----------------|
| **Skill Entry Point** (SKILL.md) | Frontmatter declares invocation contract; body contains all interactive logic, questions, and references | Single SKILL.md per runtime format |
| **Service Registry** | Knows default port, default image tag, required credentials, and optional extras per service | Embedded section inside SKILL.md (no separate runtime needed) |
| **Interactive Flow** | Sequence of questions (service?, port?, version?, credentials?), validation, and confirmation before writing | Written as instructions in SKILL.md body; AI executes |
| **Template Library** | Static Docker Compose and Taskfile content per service | Files in `templates/` directory; AI reads and copies |
| **Merge Logic** | If `.devtools/` exists, add service without clobbering existing services | Rules described in SKILL.md; AI uses its file tools |
| **Output Writer** | Creates `.devtools/` directory structure and writes final files | AI's Write/Edit tools; guided by SKILL.md instructions |

---

## Recommended Project Structure

```
dev-tools/
├── README.md                       # Human-facing documentation
│
├── skills/                         # Canonical skill source (distributable)
│   └── add-service/
│       └── SKILL.md                # Primary skill: one file, all logic + templates
│
├── .github/
│   └── skills/                     # GitHub Copilot CLI skill path
│       └── add-service/
│           └── SKILL.md            # Symlink or copy of skills/add-service/SKILL.md
│
├── .agents/
│   └── skills/                     # Claude/MCP .agents runtime path
│       └── add-service/
│           └── SKILL.md            # Symlink or copy of skills/add-service/SKILL.md
│
└── templates/                      # Template files (referenced from SKILL.md)
    ├── services/                   # Docker Compose service fragments
    │   ├── redis.compose.yml
    │   ├── rabbitmq.compose.yml
    │   ├── postgres.compose.yml
    │   ├── mysql.compose.yml
    │   └── mongodb.compose.yml
    ├── taskfiles/                  # Per-service Taskfile fragments
    │   ├── redis.Taskfile.yml
    │   ├── rabbitmq.Taskfile.yml
    │   ├── postgres.Taskfile.yml
    │   ├── mysql.Taskfile.yml
    │   └── mongodb.Taskfile.yml
    └── root.Taskfile.yml           # Root Taskfile template (includes: block)
```

### Structure Rationale

- **`skills/add-service/SKILL.md`:** Single source of truth for the skill. Runtime-specific paths (`.github/skills/`, `.agents/skills/`) are symlinks or copies of this file — no logic duplication.
- **`.github/skills/`:** Required path for GitHub Copilot CLI skill discovery (confirmed by copilot-toolkit patterns). Contains either a symlink or install-time copy.
- **`.agents/skills/`:** Required path for Claude/MCP skills.sh ecosystem (confirmed by `copilot-toolkit/.agents/skills/` convention). Same SKILL.md format.
- **`templates/`:** Separating templates from the skill definition makes each template editable independently. The AI reads them at write-time. Keeps SKILL.md from becoming unmaintainably large.
- **No `src/`, no build step, no `package.json`:** This project is static files all the way down. There is no compilation step. The constraint "no runtime dependencies" means the templates must be readable text, not generated code.

---

## Architectural Patterns

### Pattern 1: Self-Contained Skill with External Template References

**What:** SKILL.md contains all interactive logic (questions, validation, merge rules, defaults) but loads actual template content from files in `templates/` at write-time via the AI's Read tool.

**When to use:** When template content is large enough that embedding it inline would make SKILL.md hard to maintain, but you still want a single skill file as the entry point.

**Trade-offs:**
- Pro: Each template file is independently editable and testable
- Pro: SKILL.md stays focused on flow logic, not YAML content
- Con: Skill is only fully self-contained when the whole repo is present — but that's fine for a cloned/installed toolkit

**How it looks in SKILL.md:**
```markdown
<execution_context>
@./templates/services/redis.compose.yml
@./templates/taskfiles/redis.Taskfile.yml
@./templates/root.Taskfile.yml
</execution_context>
```

### Pattern 2: Per-Service Taskfile with Root Include

**What:** One Taskfile per service (e.g. `redis.yml`) that owns `up`, `down`, `logs`, `restart` for that service. A root `Taskfile.yml` uses `includes:` to stitch them together.

**When to use:** Always, for this project. This is the modular pattern that lets users add services without editing existing files.

**Trade-offs:**
- Pro: Adding a service = adding one file + one `includes:` entry. Nothing else changes.
- Pro: Users can delete a per-service file without touching root Taskfile
- Con: Task names become namespaced (`task redis:up` not `task up-redis`)

**Taskfile v3 `includes:` syntax:**
```yaml
# .devtools/Taskfile.yml
version: '3'

includes:
  redis:
    taskfile: ./taskfiles/redis.yml
    optional: true
  postgres:
    taskfile: ./taskfiles/postgres.yml
    optional: true
```
Using `optional: true` means removing a service's taskfile won't break the root.

### Pattern 3: Merge-Safe Service Addition

**What:** When `.devtools/` already exists, the skill checks existing files and adds only what's missing — never overwrites.

**When to use:** Every invocation where `.devtools/` is already present.

**Detection logic (in SKILL.md instructions):**
```
1. Check if .devtools/docker-compose.yml exists
   - If yes: read it, check if service name already in `services:` block
   - If service exists: inform user, skip writing
   - If service missing: append new service block

2. Check if .devtools/Taskfile.yml exists
   - If yes: read it, check if service already in `includes:` block
   - If missing: add new includes entry

3. Check if .devtools/taskfiles/<service>.yml exists
   - If yes: skip (never overwrite)
   - If no: write fresh
```

---

## Data Flow

### "Add Redis to this project" — Full Request Flow

```
User: "add Redis to this project"
    ↓
AI Runtime detects skill trigger (keyword match on service names / intent)
    ↓
SKILL.md interactive flow activates
    ↓
[Q1] Which service? → "Redis" (already specified in user message)
[Q2] Port? → default 6379 offered, user confirms or changes
[Q3] Version/image tag? → default "redis:7-alpine" offered
[Q4] Auth required? → default no, but configurable
    ↓
Configuration assembled in memory:
  { service: "redis", port: 6379, image: "redis:7-alpine", auth: false }
    ↓
Merge check: does .devtools/ exist?
    ├── No  → Create fresh .devtools/ structure
    └── Yes → Check if redis already present
                  ├── Already present → Inform user, no writes
                  └── Not present    → Add redis to existing files
    ↓
Template reads:
  templates/services/redis.compose.yml → fill in port/image variables
  templates/taskfiles/redis.Taskfile.yml → fill in service name/port
  templates/root.Taskfile.yml → use as base or update includes:
    ↓
File writes:
  .devtools/docker-compose.yml    (created or updated)
  .devtools/Taskfile.yml          (created or updated)
  .devtools/taskfiles/redis.yml   (created, never overwritten)
    ↓
Confirmation message to user:
  "✓ Added Redis to .devtools/
   Run: task redis:up   → start Redis
        task redis:logs → view logs
        task redis:down → stop Redis"
```

### State: What Exists Between Invocations

There is no persistent skill state — the `.devtools/` directory IS the state. Each invocation reads the current `.devtools/` contents to determine what's already there.

```
.devtools/ directory
    = source of truth for "what services are installed"
    = read at the start of each invocation
    = written to at the end of each invocation
```

---

## Output: The `.devtools/` Directory

The generated output follows this layout:

```
.devtools/
├── docker-compose.yml          # All services in one compose file
├── Taskfile.yml                # Root — only includes: entries
└── taskfiles/
    ├── redis.yml               # redis: up, down, logs, restart, flush
    ├── postgres.yml            # postgres: up, down, logs, restart, psql
    ├── rabbitmq.yml            # rabbitmq: up, down, logs, restart, ui
    ├── mysql.yml               # mysql: up, down, logs, restart, cli
    └── mongodb.yml             # mongodb: up, down, logs, restart, cli
```

**docker-compose.yml:** Uses Docker Compose v2 syntax (no `version:` key). Each service gets its own named block. Port, image, environment variables, and named volume are service-specific.

**Taskfile.yml (root):** Only contains `version: '3'` and `includes:`. No tasks defined directly at root — all tasks are in per-service files.

**taskfiles/<service>.yml:** Defines tasks:
- `up` — `docker compose -f .devtools/docker-compose.yml up -d <service>`
- `down` — `docker compose -f .devtools/docker-compose.yml stop <service>`
- `logs` — `docker compose -f .devtools/docker-compose.yml logs -f <service>`
- `restart` — `down` then `up`
- Service-specific: `flush` (redis), `psql`/`cli` (databases), `ui` (rabbitmq management)

---

## Build Order (Phase Dependencies)

```
Phase 1: Templates
    ↓ (no dependencies — pure content authoring)
    templates/services/*.compose.yml
    templates/taskfiles/*.Taskfile.yml
    templates/root.Taskfile.yml

Phase 2: Service Registry + Interactive Flow
    ↓ (depends on Phase 1 existing — flow references templates)
    Service defaults per service (port, image, auth)
    Question sequence
    Merge detection logic
    Validation rules

Phase 3: SKILL.md Assembly
    ↓ (depends on Phase 1 + Phase 2 — references both)
    skills/add-service/SKILL.md
    Frontmatter (name, description, allowed-tools)
    All interactive instructions embedded
    Template references via execution_context

Phase 4: Cross-Runtime Wiring
    ↓ (depends on Phase 3 — copies/symlinks the canonical skill)
    .github/skills/add-service/SKILL.md (GitHub Copilot CLI path)
    .agents/skills/add-service/SKILL.md (Claude/MCP path)
    Optional: install.sh for repo users
```

**Critical dependency:** SKILL.md must reference templates by path. If templates are reorganized after SKILL.md references them, paths break. Lock the `templates/` layout before writing the final SKILL.md.

---

## Anti-Patterns

### Anti-Pattern 1: All Logic in SKILL.md as Inline YAML

**What people do:** Embed full Docker Compose and Taskfile content as code blocks directly inside SKILL.md.

**Why it's wrong:** SKILL.md becomes 500+ lines. Adding a service means editing the skill itself — mixing content with logic. Template errors are harder to isolate.

**Do this instead:** Keep templates as separate files in `templates/`. SKILL.md stays focused on flow and references. Templates are independently diffable and testable.

---

### Anti-Pattern 2: One Taskfile For All Services

**What people do:** Write a single `Taskfile.yml` with tasks like `redis-up`, `redis-down`, `postgres-up` etc. all in one file.

**Why it's wrong:** Adding a new service requires editing the existing Taskfile — violates the "adding a service doesn't require editing existing files" requirement. Also creates merge conflicts when multiple developers add services.

**Do this instead:** Per-service Taskfiles using `includes:`. Each service is its own isolated file. Root Taskfile only adds a new `includes:` entry.

---

### Anti-Pattern 3: Separate SKILL.md Per Runtime

**What people do:** Write different SKILL.md files for Copilot CLI vs Claude/MCP because they "feel different".

**Why it's wrong:** Maintenance burden doubles. Bugs get fixed in one but not the other. The two formats are actually identical (same SKILL.md schema) — only the path differs.

**Do this instead:** One canonical `skills/add-service/SKILL.md`. The `.github/skills/` and `.agents/skills/` paths are symlinks or install-time copies.

---

### Anti-Pattern 4: Overwriting Existing `.devtools/` Files

**What people do:** Always write fresh files, clobbering whatever was in `.devtools/` before.

**Why it's wrong:** Destroys user customizations (changed ports, added healthchecks, custom task logic). Creates a frustrating "why did my changes disappear" experience.

**Do this instead:** Always read before writing. Append new service blocks to existing compose/Taskfile. Never write a per-service taskfile if one already exists for that service.

---

## Integration Points

### External Services

| Service | Integration Pattern | Notes |
|---------|---------------------|-------|
| **Docker Compose v2** | Written output (`.devtools/docker-compose.yml`) | Target is `docker compose` CLI, not `docker-compose`. No `version:` key in output. |
| **Taskfile v3** | Written output (`.devtools/Taskfile.yml`, `taskfiles/*.yml`) | Must use `version: '3'` key. `includes:` with `optional: true` for resilience. |
| **GitHub Copilot CLI** | `.github/skills/<name>/SKILL.md` discovery path | Requires `name` and `description` in frontmatter. `allowed-tools: Read, Write, Edit` needed. |
| **Claude/MCP (skills.sh)** | `.agents/skills/<name>/SKILL.md` discovery path | Same format as Copilot CLI. Discoverable via `npx skills` ecosystem. |

### Internal Boundaries

| Boundary | Communication | Notes |
|----------|---------------|-------|
| SKILL.md ↔ templates/ | AI reads template files via Read tool | SKILL.md references paths; AI resolves at runtime |
| SKILL.md ↔ .devtools/ | AI reads existing files for merge check, then writes | Path is always relative to the invoking project's root |
| .github/skills/ ↔ skills/ | File copy or symlink (install-time) | No runtime communication — same file content |
| .agents/skills/ ↔ skills/ | File copy or symlink (install-time) | Same as above |

---

## Scaling Considerations

This is a file-writing tool invoked per developer request. "Scale" here means service count and project variety, not traffic.

| Scale | Architecture Adjustments |
|-------|--------------------------|
| 5 services (current scope) | Single SKILL.md, flat `templates/` layout — no special handling needed |
| 10–20 services | Consider a `services/` registry file listing all supported services with metadata; SKILL.md reads it at runtime |
| 30+ services | Split into service-category skills (databases, queues, caches) or add a service selector step |

### Scaling Priorities

1. **First bottleneck:** SKILL.md length — as more services are added, the interactive flow branches grow. Mitigation: extract service registry to a separate reference file early.
2. **Second bottleneck:** Template drift — if compose/Taskfile syntax evolves, all service templates need updating. Mitigation: shared base fragments that service templates extend.

---

## Sources

- Live inspection of `github.com/Cyboooooorg/copilot-toolkit` skill structure (HIGH confidence)
- Live inspection of `~/.copilot/skills/gsd-*` skill format (HIGH confidence)
- Live inspection of `.agents/skills/` convention in Cyboooooorg/ai-playground (HIGH confidence)
- `skills.sh` ecosystem conventions observed in copilot-toolkit (MEDIUM confidence — pattern matches multiple sources)
- Taskfile v3 `includes:` syntax with `optional: true` (MEDIUM confidence — based on taskfile.dev documentation patterns)

---
*Architecture research for: AI skill toolkit / dev-tooling generator*
*Researched: 2025-01-10*
