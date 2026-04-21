# Roadmap: dev-tools

## Milestones

### ✅ v1.0 MVP — SHIPPED 2026-04-21

7 phases (1–7), 14 plans — Interactive `add-service` AI skill with merge detection, multi-instance aliases, subfolder output layout, and idempotent re-runs for 5 Docker services.

See [`.planning/milestones/v1.0-ROADMAP.md`](.planning/milestones/v1.0-ROADMAP.md) for full phase details.

---

## v1.1 Port Management

### Phases

- [ ] **Phase 8: Host Port Opt-In** — Skill asks whether to expose the service on a host port; compose output matches the user's choice exactly
- [ ] **Phase 9: Port Collision Guard** — Chosen port validated against all already-installed services; re-prompt loop until conflict-free or user opts out

### Phase Details

#### Phase 8: Host Port Opt-In
**Goal**: Users decide whether their service needs a host port, and the compose output reflects that choice exactly
**Depends on**: Phase 7 (v1.0 foundation)
**Requirements**: PORT-01, PORT-02
**Success Criteria** (what must be TRUE):
  1. Skill asks "Do you want to expose this service on a host port?" (default: no) before any port-number question
  2. When user says no, the written compose file contains no `ports:` block for that service
  3. When user says yes, the skill proceeds to the port-number question
  4. A service installed with the no-port path is reachable only within the Docker network — no host port appears anywhere in its compose file
**Plans**: TBD

#### Phase 9: Port Collision Guard
**Goal**: When a user requests host port binding, their chosen port is validated against all installed services before any file is written, and the skill re-prompts until a safe port is confirmed or the user backs out
**Depends on**: Phase 8
**Requirements**: PORT-03, PORT-04, PORT-05
**Success Criteria** (what must be TRUE):
  1. Before asking for a port number, skill scans every `.devtools/*/.*compose.yml` and `.devtools/*/.env` to collect all already-bound host ports (main, UI, and monitoring)
  2. If the entered port is already bound, skill names the conflicting service (e.g., "port 6379 is already used by redis") and prompts for a different port
  3. Re-prompt loop continues until the user enters a non-conflicting port
  4. User can opt out of host binding at any point in the re-prompt loop; result is no `ports:` entry (same as Phase 8 no-port path)
  5. When a non-conflicting port is confirmed, the written compose file contains the correct `ports:` entry with that port
**Plans**: TBD

### Progress

| Phase | Plans Complete | Status | Completed |
|-------|----------------|--------|-----------|
| 8. Host Port Opt-In | 0/? | Not started | - |
| 9. Port Collision Guard | 0/? | Not started | - |

---

## v2.0 Phases

*(Next milestone — start with `/gsd-new-milestone`)*
