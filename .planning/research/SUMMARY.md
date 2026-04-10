# Project Research Summary

**Project:** dev-tools — AI skill toolkit (Docker Compose + Taskfile generator)
**Domain:** AI agent skill / developer infrastructure scaffolding
**Researched:** 2026-04-10
**Confidence:** HIGH

## Executive Summary

This project is a zero-dependency AI agent skill that scaffolds Docker Compose v2 service configurations and Taskfile v3 task runners into a `.devtools/` directory via a natural-language conversation. The expert approach is to write a single `SKILL.md` file (identical format for both GitHub Copilot CLI and Claude) that guides the AI through an interactive Q&A before writing any files. The key architectural insight is that there is no runtime, no build step, and no package manager — the skill is pure Markdown + YAML that the AI agent executes using its own file tools. Templates for each of the 5 target services (Redis, RabbitMQ, PostgreSQL, MySQL, MongoDB) live as separate files the skill reads at write-time, keeping the entry point focused on flow logic.

The recommended approach builds in four ordered phases: (1) author all service templates first — locking the file layout before SKILL.md references it, (2) write the SKILL.md interactive flow with embedded service registry and merge-detection logic, (3) wire the canonical skill to both runtime discovery paths (`.github/skills/` and `.agents/skills/`), and (4) add polish features (service CLI tasks, `.env` generation) after the core is validated. The differentiating value over Lando/DDEV/Devbox is that users install nothing — the AI is the CLI, and the generated output is plain files any developer can read and modify.

The primary risks are correctness risks, not technical complexity risks: generating a `version:` key in Compose output (deprecated), mismatching Taskfile schema versions across included files, corrupting user files during merge, and hardcoding credentials. All of these can be prevented by following established rules from day one — they are not hard problems, but they cause severe UX damage if they slip through. The merge/idempotency logic (never overwriting existing files, always checking before writing) is the most important behavioral contract in the entire skill.

---

## Key Findings

### Recommended Stack

The stack is intentionally minimal. `SKILL.md` is the only file format that matters — it is identical for GitHub Copilot CLI and Claude Code, so one canonical file covers both runtimes. Generated output targets Docker Compose v2 (no `version:` key — the field is officially obsolete in the Compose Spec) and Taskfile v3 (`version: '3'`, with `includes:` for per-service modularity). No runtime dependencies exist: the skill is static files that an AI writes using its own tools. Ship the skill in all three discovery paths (`.github/skills/`, `.agents/skills/`, `.claude/skills/`) via symlinks for zero-config pickup by any runtime.

**Core technologies:**
- `SKILL.md` format: AI skill entry point — one file, both Copilot CLI and Claude, no divergence
- Docker Compose v2 (`compose.yaml`, no `version:` key): Generated service configs — current spec, eliminates deprecation warnings
- Taskfile v3 (`version: '3'`, `includes:` + `optional: true`): Generated task runner — modular per-service files, additive without editing existing files
- YAML anchors + `x-*` extension fields: DRY template patterns in Compose — use `x-common-service: &anchor` for shared restart/network config
- `{{.TASKFILE_DIR}}`: Essential Taskfile built-in — makes all relative paths in included Taskfiles resolve correctly regardless of invocation location

### Expected Features

**Must have (table stakes — v1):**
- Docker Compose service definitions for all 5 services (Redis, RabbitMQ, PostgreSQL, MySQL, MongoDB)
- Interactive config prompts before any file write (port, version, credentials) — never assume defaults
- Named volumes per service — data loss on first `docker compose down` is a fatal first impression
- Health checks per service — enables `depends_on: condition: service_healthy`
- Per-service Taskfile with `up`, `down`, `logs`, `restart` tasks
- Root `Taskfile.yml` with `includes:` for each added service (using `optional: true`)
- Merge / no-clobber behavior — adding service #2 must not disturb service #1's files
- Post-generation output with file list + connection string — eliminates "now what?"
- AI skill definition for both Copilot CLI + Claude — the entire value proposition

**Should have (competitive — v1.x):**
- Service-specific shell access tasks (`redis:cli`, `postgres:psql`, `mysql:cli`, `rabbitmq:admin`)
- `.env` / `.env.example` generation — credentials via env var interpolation, never hardcoded
- Version pinning prompts ("Which Redis version? [7-alpine]:")
- RabbitMQ Management UI port 15672 exposed by default

**Defer (v2+):**
- Additional services (MinIO, Kafka, Elasticsearch, NATS, Valkey)
- `devtools remove <service>` — inverse merge logic; wait for user demand
- Snapshot / backup tasks — DB dump per service
- `task devtools:status` — requires Docker CLI integration at runtime

### Architecture Approach

The architecture has four components with a clean separation of concerns: SKILL.md (entry point and interactive logic), Service Registry (embedded in SKILL.md — defaults per service), Template Library (separate files in `templates/` read by the AI at write-time), and the `.devtools/` directory (which *is* the persistent state — no database, no daemon). The key patterns are: per-service Taskfile with root include (one file per service, additive), merge-safe service addition (check before every write), and self-contained skill with external template references (SKILL.md stays focused on flow).

**Major components:**
1. **SKILL.md** — frontmatter invocation contract + interactive Q&A flow + merge detection rules
2. **Service Registry** (embedded in SKILL.md) — port, image, auth defaults for each of the 5 services
3. **Template Library** (`templates/services/*.compose.yml`, `templates/taskfiles/*.Taskfile.yml`) — static content read at write-time
4. **`.devtools/` directory** — persistent state: `docker-compose.yml`, `Taskfile.yml` (root, includes-only), `taskfiles/<service>.yml`

**Build order constraint:** Templates must be finalized before SKILL.md references them. Reorganizing `templates/` after SKILL.md is written breaks path references.

### Critical Pitfalls

1. **`version:` key in generated Compose files** — never write it; Docker Compose v2 treats it as obsolete and emits warnings. Validate templates with `grep "^version:" .devtools/docker-compose.yml` returning empty.

2. **Hardcoded credentials in compose files** — all passwords go in `.devtools/.env` (gitignored) referenced as `${POSTGRES_PASSWORD}`; commit only `.env.example` with placeholder values. This must be a day-one design decision, not retrofitted.

3. **YAML anchor destruction during merge** — never parse-and-rewrite a user's existing `compose.yaml`. Always write to isolated `.devtools/docker-compose.yml` and append an `include:` directive to the user's root file. Text-only append, never round-trip through a YAML parser.

4. **Taskfile version mismatch on includes** — Task requires all included files to use the same `version:` as the root. Mirror the user's existing `version:` value in all generated files; default to `'3'` only if no root Taskfile exists.

5. **Non-idempotent writes** — running the skill twice on the same service must produce identical output, not duplicate service blocks. Enforce "check-before-write" as a universal rule: read existing file, verify service not present, then and only then write.

6. **Taskfile `{{.TASKFILE_DIR}}` omission** — tasks in included files run from the root Taskfile's directory by default, not from the included file's directory. Every Docker Compose `-f` flag in generated Taskfiles must use `{{.TASKFILE_DIR}}/docker-compose.yml`.

7. **Skill description precision** — too generic triggers for unrelated Docker questions; too narrow misses "add a cache" or "I need a message broker." Description must enumerate all 5 service names and state "Writes Docker Compose and Taskfile configurations into `.devtools/`."

---

## Implications for Roadmap

Based on combined research, the project has a clear 4-phase build order with hard dependencies between phases.

### Phase 1: Templates & Credential Strategy
**Rationale:** Templates must be finalized before SKILL.md can reference them. Credential strategy must be designed before any template is authored — retrofitting `.env` patterns after templates exist means rewriting all of them. This phase has zero external dependencies and is pure content authoring.
**Delivers:** Complete `templates/` directory — 5 Compose fragments, 5 Taskfile fragments, 1 root Taskfile template; `.env.example` pattern established; no `version:` keys anywhere.
**Addresses:** All 5 services' compose definitions, named volumes, health checks, per-service tasks (up/down/logs/restart).
**Avoids:** Hardcoded credentials (Pitfall 5), `version:` key in output (Pitfall 4), `TASKFILE_DIR` omission (Pitfall 7).
**Research flag:** Standard patterns — well-documented Compose v2 and Taskfile v3 specs. Skip research-phase.

### Phase 2: Skill Core — Interactive Flow & Service Registry
**Rationale:** Depends on Phase 1 templates existing. This is the riskiest phase — skill description precision, idempotency rules, and merge-detection logic are all implemented here. Getting this wrong means a skill that either never triggers, always triggers for the wrong thing, or corrupts user files.
**Delivers:** `skills/add-service/SKILL.md` with complete frontmatter, interactive Q&A flow, service registry (5 services with defaults), merge-detection logic, check-before-write rule, post-generation output with connection strings.
**Addresses:** Interactive config, merge/no-clobber, idempotency, post-generation output, AI skill definition.
**Avoids:** Namespace collision (Pitfall 1), version mismatch (Pitfall 2), anchor destruction (Pitfall 3), idempotency failure (Pitfall 9), generic description (Pitfall 8), port collision UX (Pitfall 6).
**Research flag:** Needs careful design iteration — skill description testing against 10+ natural language prompts is a manual validation step. Consider `/gsd-research-phase` for interactive flow patterns if SKILL.md authoring patterns are unclear.

### Phase 3: Cross-Runtime Wiring & Documentation
**Rationale:** Depends on Phase 2. Mechanical step — symlink or copy the canonical SKILL.md to all three runtime discovery paths. Includes README and generated `.devtools/README.md` template.
**Delivers:** `.github/skills/add-service/SKILL.md`, `.agents/skills/add-service/SKILL.md`, optional `.claude/skills/add-service/SKILL.md` (symlinks); project `README.md` with installation and usage docs.
**Uses:** All runtime discovery paths from STACK.md research.
**Implements:** Cross-runtime wiring architecture component.
**Research flag:** Standard patterns. Skip research-phase.

### Phase 4: Polish — v1.x Features
**Rationale:** Add after Phase 3 is validated with real usage. These are the P2 features that add developer delight without changing the core contract.
**Delivers:** Service-specific CLI tasks (`redis:cli`, `postgres:psql`, etc.); `.env` / `.env.example` generation; version pinning prompts; RabbitMQ management UI port exposure.
**Addresses:** P2 features from FEATURES.md prioritization matrix.
**Research flag:** Standard additions. Skip research-phase.

### Phase Ordering Rationale

- **Templates before SKILL.md** — SKILL.md references templates by path; path layout must be locked first. Architecture research explicitly calls this a "critical dependency."
- **Credential strategy before templates** — once compose templates are written without env var interpolation, fixing them requires rewriting all 5. Pitfall 5 mandates this decision first.
- **Merge/idempotency in Phase 2, not later** — "merge / no-clobber cannot be bolted on later" (FEATURES.md). Must be designed into the initial write logic.
- **Cross-runtime wiring after SKILL.md** — nothing to symlink until the canonical file is complete and tested.
- **Polish after validation** — defer P2 features until the core interaction loop is confirmed working with real users.

### Research Flags

Needs research during planning:
- **Phase 2 (Skill Core):** Natural language skill description testing requires iteration. No automated way to verify "triggers correctly for 10+ prompts." Plan for manual validation step.

Standard patterns (skip research-phase):
- **Phase 1 (Templates):** Compose v2 and Taskfile v3 patterns are fully documented and verified in STACK.md.
- **Phase 3 (Cross-runtime):** Discovery paths for both runtimes are confirmed from live inspection (ARCHITECTURE.md sources).
- **Phase 4 (Polish):** Additive task patterns, no new architectural decisions.

---

## Confidence Assessment

| Area | Confidence | Notes |
|------|------------|-------|
| Stack | HIGH | All findings verified against official live specs (compose-spec GitHub, taskfile.dev schema, GitHub Copilot docs, MCP spec). Both Compose v2 `version:` obsolescence and Taskfile `includes:` behavior confirmed from authoritative sources. |
| Features | HIGH | Table stakes derived from competitor analysis (Lando, DDEV, Devbox, DevContainers) + PROJECT.md constraints. Anti-features well-reasoned. MEDIUM for competitive differentiator positioning (Lando/DDEV rendered SPAs not fully parseable). |
| Architecture | HIGH | Derived from live inspection of Cyboooooorg/copilot-toolkit and `.agents/skills/` conventions. Component boundaries and data flow are clear and consistent with SKILL.md format constraints. |
| Pitfalls | HIGH | 10 pitfalls with verified sources. Compose spec merge rules, Taskfile version semantics, YAML anchor round-trip loss all sourced from official specs or reproducible tests. |

**Overall confidence:** HIGH

### Gaps to Address

- **Skill invocation precision:** The exact description text that achieves reliable skill triggering across both Copilot CLI and Claude runtimes cannot be validated until the skill is written and tested against real prompts. Plan for a description-tuning iteration in Phase 2.
- **Compose `include:` vs app container networking:** FEATURES.md explicitly defers service-to-service networking (connecting the user's app container to devtools services). This gap will resurface in user onboarding — the generated `README.md` must document the `.devtools_default` network name clearly.
- **MySQL ARM64 (Apple Silicon) compatibility:** PITFALLS.md flags that `mysql:latest` has inconsistent ARM support. The v1 template should use `mysql:8` with a `platform: linux/amd64` note or recommend `mariadb` as an alternative. Decision needed during Phase 1 template authoring.
- **Volume naming strategy:** PITFALLS.md recommends using a `name:` top-level field in the generated compose file to avoid cross-project volume collisions. Whether to prompt for a project name or auto-derive it from the directory name is an open design decision for Phase 2.

---

## Sources

### Primary (HIGH confidence)
- `https://docs.github.com/en/copilot/how-tos/copilot-cli/customize-copilot/create-skills` — SKILL.md frontmatter fields, skill discovery paths
- `https://raw.githubusercontent.com/compose-spec/compose-spec/master/spec.md` — Compose v2 canonical spec, `version:` obsolescence, `include:` directive
- `https://taskfile.dev/schema.json` + `https://taskfile.dev/docs/reference/schema.md` — Taskfile v3 full schema
- `https://taskfile.dev/taskfile-versions/` — included Taskfile version parity requirement
- `https://modelcontextprotocol.io/specification/2025-06-18` — MCP current spec (confirmed SKILL.md is preferable over MCP for this use case)
- Live inspection: `github.com/Cyboooooorg/copilot-toolkit` skill structure and `.agents/skills/` convention

### Secondary (MEDIUM confidence)
- `https://lando.dev/features` — competitor feature analysis (rendered SPA, metadata only)
- `https://containers.dev` — DevContainers spec competitor analysis (training data cross-reference)
- `https://www.jetpack.io/devbox` — Devbox feature analysis (Nix-based, not Docker; MEDIUM confidence)

### Tertiary (LOW confidence / inference)
- Port collision defaults (16379, etc.) — community convention, not official spec. Validate during Phase 2 interactive flow design.

---
*Research completed: 2026-04-10*
*Ready for roadmap: yes*
