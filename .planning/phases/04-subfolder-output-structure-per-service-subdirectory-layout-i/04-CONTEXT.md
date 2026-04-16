# Phase 4: Subfolder Output Structure — Context

**Gathered:** 2026-04-16
**Status:** Ready for planning

<domain>
## Phase Boundary

Reorganize how the `add-service` skill writes output files — from a flat `.devtools/` layout
(all files at root) to a **per-instance subfolder layout** where each installed service gets
its own dedicated subdirectory containing its compose file, Taskfile, and `.env`.

**In scope:** Updating `SKILL.md` file-path logic (detection, writes, merge detection), updating
root `compose.yml` include paths, updating root `Taskfile.yml` include paths, updating
`.gitignore` to cover subfolder `.env` files, updating the skill's frontmatter description
and done-summary to reflect the new layout.

**Out of scope:** Changes to template file contents, changes to Q&A flow, changes to runtime
detection (Copilot CLI vs Claude/MCP), migration of existing `.devtools/` directories.

</domain>

<decisions>
## Implementation Decisions

### Subfolder Naming Strategy

- **D-01:** Each installed service gets its **own subdirectory** named after the service-instance
  slug: `.devtools/<slug>/` where `<slug>` is `${SERVICE}` for standard installs and
  `${SERVICE}-${ALIAS}` for alias (multi-instance) installs.
  - Standard: `.devtools/redis/`
  - Alias: `.devtools/redis-cache/`
- **D-02:** The folder name is the **same as the current file prefix** — no renaming of the
  slug convention, just lifting it into a directory.

### Per-Instance File Layout

- **D-03:** Each subfolder contains exactly three files:
  - `<slug>.compose.yml` — the Docker Compose service fragment
  - `<slug>.Taskfile.yml` — the per-service Taskfile (if Taskfile=yes)
  - `.env` — credentials and config for this service only
- **D-04:** Service files inside the subfolder keep their existing `<slug>.compose.yml` and
  `<slug>.Taskfile.yml` naming (not renamed to generic `compose.yml`/`Taskfile.yml`) —
  makes the file unambiguous when opened in an editor outside the folder context.

### Per-Service .env

- **D-05:** Each subfolder gets its own `.env` file containing **only that service's credentials
  and config vars** (e.g., `.devtools/redis/.env` has `REDIS_PORT`, `REDIS_PASSWORD`, etc.).
- **D-06:** `COMPOSE_PROJECT_NAME` and `COMPOSE_PROFILES` remain in a **minimal `.devtools/.env`**
  at the root level — Docker Compose auto-reads this file when the main compose command runs
  from `.devtools/compose.yml`. These are project-level vars, not service-level.
- **D-07:** The service compose template's `env_file: .env` directive resolves to the
  subfolder's `.env` (Docker Compose `include:` resolves `env_file` paths relative to the
  included file's directory). No template changes required — path resolution handles this.

### Root Files (Unchanged Locations)

- **D-08:** These files **stay at `.devtools/` root** — their location does not change:
  - `.devtools/compose.yml` — root orchestrator with `include:` entries
  - `.devtools/Taskfile.yml` — root with `includes:` entries
  - `.devtools/.gitignore` — updated to cover subfolder `.env` files
  - `.devtools/.env` — minimal, project-level vars only (COMPOSE_PROJECT_NAME, COMPOSE_PROFILES)
  - `.devtools/.env.example` — updated to reflect new structure

### Root compose.yml Include Paths

- **D-09:** When a service is installed, the `include:` entry added to `.devtools/compose.yml`
  uses the **subfolder-relative path**:
  ```yaml
  include:
    - path: redis/redis.compose.yml
  ```
  Previously was `redis.compose.yml`.

### Root Taskfile.yml Include Paths

- **D-10:** When a service is installed, the `includes:` entry in `.devtools/Taskfile.yml`
  uses the **subfolder-relative path**:
  ```yaml
  includes:
    redis:
      taskfile: redis/redis.Taskfile.yml
      optional: true
  ```
  Previously was `redis/redis.Taskfile.yml` → now just confirms this is the new canonical form.

### Merge Detection

- **D-11:** The existence check for detecting an already-installed service changes from:
  ```bash
  test -f .devtools/${SERVICE}.compose.yml
  ```
  To:
  ```bash
  test -f .devtools/${SERVICE}/${SERVICE}.compose.yml
  ```
  (and for alias: `test -f .devtools/${SERVICE_SLUG}/${SERVICE_SLUG}.compose.yml`)

### .gitignore Update

- **D-12:** `.devtools/.gitignore` changes from just `.env` to a pattern that covers all
  subfolder `.env` files:
  ```
  .env
  **/.env
  ```
  This ensures both the root `.devtools/.env` and any `services/<slug>/.env` are gitignored.

### Rename-Before-Add Flow (Multi-Instance UX)

- **D-15:** When merge detection finds an existing service (`.devtools/${SERVICE}/` exists), the
  skill **first offers to rename the existing instance** before installing the new one, instead
  of jumping straight to "add another alias". This turns a conflict into a clean multi-instance
  setup.
- **D-16:** The rename prompt flow:
  1. Detect: `.devtools/redis/` exists.
  2. Offer: `"Redis is already installed as 'redis'. Before adding a new instance, rename the
     existing one first? (e.g., rename 'redis' → 'redis-cache')"`
  3. If user provides a rename alias (e.g., `cache`):
     - Rename the folder: `.devtools/redis/` → `.devtools/redis-cache/`
     - Rename files inside: `redis.compose.yml` → `redis-cache.compose.yml`,
       `redis.Taskfile.yml` → `redis-cache.Taskfile.yml`
     - Apply string substitution in the renamed files (container name, volume name, env var
       prefixes) — same as an alias install would have produced
     - Update `.devtools/compose.yml`: replace `include: redis/redis.compose.yml` with
       `include: redis-cache/redis-cache.compose.yml`
     - Update `.devtools/Taskfile.yml`: replace `redis: taskfile: redis/redis.Taskfile.yml`
       with `redis-cache: taskfile: redis-cache/redis-cache.Taskfile.yml`
     - Update `.devtools/.env` and `.devtools/redis-cache/.env`: rename env var keys from
       `REDIS_*` to `REDIS_CACHE_*` (with `## REPLACED` markers on old keys per D-13)
     - Then continue to install the new instance (now the `redis/` slot is free)
  4. If user declines rename:
     - Fall back to existing alias-only prompt: `"Add another instance with an alias? [alias/cancel]"`
- **D-17:** After renaming completes, the skill confirms: `"Existing redis renamed to redis-cache.
  Now installing new redis instance..."` and proceeds through the normal Q&A flow for the
  new instance (which will land in `.devtools/redis/`).
- **D-18:** The rename operation is **atomic in output** — all file moves and reference updates
  are done before any new files are written, so there is no intermediate broken state in the
  summary shown to the user.

### Backward Compatibility

- **D-13:** **No migration.** If `.devtools/` already exists with the old flat-file layout
  (e.g., `redis.compose.yml` at root), the skill does **not** attempt to move or transform
  those files. Old flat files continue to work as-is. New service installs always use the
  subfolder layout. The skill does not mix detection strategies — it only checks for the
  new subfolder path.

### Skill Description Update

- **D-14:** The skill frontmatter description and `<objective>` block should be updated to
  mention the subfolder layout (e.g., "writes to `.devtools/<service>/`") so users and
  AI runtimes have accurate expectations.

### the agent's Discretion

- Exact `.env.example` structure for the new layout (whether to document subfolder layout
  in comments) — agent decides what's clearest.
- Whether to mkdir the subfolder with `mkdir -p .devtools/${SERVICE_SLUG}/` or let the write
  step create it — agent picks the most reliable approach.
- Whether the done-summary file list needs visual grouping by subfolder — agent decides
  based on readability.

</decisions>

<specifics>
## Specific Ideas

- User's primary motivation: "simpler to manage multiple services of same type" — the subfolder
  structure makes it trivial to see, delete, or inspect a single service instance without
  touching others.
- Each subfolder should be fully self-contained: delete the folder = fully remove that service
  from the project (minus the include line in root compose/Taskfile).

</specifics>

<canonical_refs>
## Canonical References

**Downstream agents MUST read these before planning or implementing.**

- `skills/add-service/SKILL.md` — the full current skill implementation (582 lines); this is
  the primary file to update. Read completely before planning.
- `compose-templates/redis/redis.compose.yml` — example of `env_file: .env` usage; confirms
  path resolution behavior under Docker Compose `include:`.
- `taskfile-templates/redis/redis.Taskfile.yml` — example of `{{.TASKFILE_DIR}}` usage;
  confirms portability of path resolution in Taskfile includes.
- `taskfile-templates/root/Taskfile.yml` — root Taskfile template showing `includes:` format.
- `.planning/phases/02-skill-core-interactive-flow-merge-logic/02-CONTEXT.md` — Phase 2
  decisions (D-01 through D-27) — especially D-15/D-16 (root compose strategy), D-25/D-27
  (alias naming), D-12/D-13 (env append strategy). Phase 4 extends these, not replaces them.

</canonical_refs>

<deferred>
## Deferred Ideas

- **Auto-migration:** Moving existing flat-file `.devtools/` to the new subfolder layout —
  deferred; user chose no migration for now.
- **`docker compose include:` env_file scoping validation:** A future phase could add a
  validation step that confirms `env_file` resolution works correctly across Docker Compose
  versions. Deferred to post-v1.

</deferred>
