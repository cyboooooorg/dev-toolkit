---
phase: 08-host-port-opt-in
plan: "01"
subsystem: skill-add-service
tags: [skill, host-port, opt-in, compose, env-vars]
dependency_graph:
  requires: []
  provides: [host-port-opt-in-gate]
  affects: [skills/add-service/SKILL.md]
tech_stack:
  added: []
  patterns: [in-memory-template-modification, ANSWERS-dict-gate]
key_files:
  created: []
  modified:
    - skills/add-service/SKILL.md
decisions:
  - "Opt-in question inserted as first universal Q (Q1), before Port (Q2) — matches D-01, D-02"
  - "Per-service question numbers incremented by 1 to stay consistent — no content change"
  - "Port removal uses in-memory modification pattern matching existing Redis-no-password approach (D-10)"
  - "RabbitMQ UI port line kept after AMQP removal per D-09; empty blocks removed per D-08"
  - "Port env var conditionally omitted in Step 10a; RABBITMQ_UI_PORT always written (D-05)"
  - "Step 8 summary shows 'internal only (no host port)' exact phrase for opt-out (D-14)"
metrics:
  duration: "~10 minutes"
  completed_date: "2026-04-21"
  tasks: 3
  files_modified: 1
---

# Phase 08 Plan 01: Host Port Opt-In Gate Summary

**One-liner:** Host-port binding made opt-in via `ANSWERS[host_port]` gate controlling compose port removal, env var skip, and Step 8 summary display.

## What Was Built

Implemented the host-port opt-in feature across four locations in `skills/add-service/SKILL.md`:

1. **Step 5 Universal Q&A** — New question 1 `"Expose this service on a host port? [y/N]"` inserted before Port question. Port is now question 2 (conditional on `ANSWERS[host_port]=true`). Image version is question 3. Per-service credential questions renumbered +1.

2. **Step 9a (Write Compose File)** — New conditional in-memory modification block. When `ANSWERS[host_port]=false`: removes the main service port line for all 5 services; removes the entire `ports:` block when empty (redis, postgres, mysql, mongodb); leaves RabbitMQ management UI port intact after removing AMQP line; never modifies UI companion or monitoring exporter `ports:` blocks.

3. **Step 10a (Write .env)** — Conditional note before the env-var table: when opted out, omits the main port env var (REDIS_PORT, RABBITMQ_PORT, etc.) but still writes RABBITMQ_UI_PORT and all other vars.

4. **Step 8 (Configuration Summary)** — Port row now shows both states: port number when opted in, `internal only (no host port)` when opted out.

## Decisions Implemented (D-01 through D-15)

| Decision | Status | Notes |
|----------|--------|-------|
| D-01 | ✅ | Question inserted as first in Step 5 Universal Q&A |
| D-02 | ✅ | Exact wording: `"Expose this service on a host port? [y/N]"` |
| D-03 | ✅ | Stored as `ANSWERS[host_port]=true/false` |
| D-04 | ✅ | Port question skipped entirely when opted out |
| D-05 | ✅ | Port env var omitted in Step 10a; RABBITMQ_UI_PORT still written |
| D-06 | ✅ | Full port flow unchanged when opted in |
| D-07 | ✅ | Only main service port line removed; other containers untouched |
| D-08 | ✅ | Empty `ports:` block removed entirely; never written empty |
| D-09 | ✅ | RabbitMQ: AMQP port removed, management UI port retained |
| D-10 | ✅ | In-memory modification; no separate template variants |
| D-11 | ✅ | UI companion ports always host-mapped; explicitly excluded |
| D-12 | ✅ | Step 6 UI companion opt-in unaffected |
| D-13 | ✅ | Monitoring exporters have no `ports:` blocks; no changes needed |
| D-14 | ✅ | Step 8 shows `Port: internal only (no host port)` when opted out |
| D-15 | ✅ | Step 8 shows port number when opted in |

## Commits

| Task | Commit | Description |
|------|--------|-------------|
| 1 | `7938168` | Insert host-port opt-in question into Step 5 Universal Q&A |
| 2 | `8e43440` | Add port-line removal logic to Step 9a compose write |
| 3 | `a6aa219` | Update Step 10a env var skip and Step 8 summary Port row |

## Verification Command Outputs

All verification checks passed:

```
1. Expose this service on a host port  → line 322 ✅
2. ANSWERS[host_port] references       → lines 323, 328, 425, 426, 470, 572, 578 ✅
3. asked only if                       → line 328 ✅
4. ANSWERS[host_port]=false            → lines 426, 470, 572 ✅
5. Never write an empty ports: list    → line 496 ✅
6. RABBITMQ_UI_PORT:15672              → line 504 ✅
7. internal only (no host port)        → line 426 ✅
8. Omit the main service port env var  → line 572 ✅
9. Line count: 865 (original ~800 + ~65 added lines) ✅
```

## Deviations from Plan

None — plan executed exactly as written.

## Known Stubs

None — all logic is fully specified in SKILL.md; no UI or data rendering involved.

## Threat Flags

None — no new network endpoints, auth paths, file access patterns, or schema changes introduced. SKILL.md is a documentation/instruction file.

## Self-Check: PASSED

- [x] `skills/add-service/SKILL.md` exists and contains all required strings
- [x] Commits `7938168`, `8e43440`, `a6aa219` all exist in git log
- [x] Line count 865 ≥ original + expected additions
