# Phase 9: Port Collision Guard - Context

**Gathered:** 2026-04-21
**Status:** Ready for planning

<domain>

## Phase Boundary

When a user opts into host port binding (Phase 8: `ANSWERS[host_port]=true`), validate their
chosen port against every host-bound port already written by installed services. Re-prompt until
a conflict-free port is confirmed, or the user escapes via a 3-option menu. No files are written
until a safe port (or internal-only fallback) is confirmed.

This phase modifies `skills/add-service/SKILL.md` only — specifically the Step 5 port Q&A flow
that was introduced in Phase 8.

</domain>

<decisions>

## Implementation Decisions

### Port Scan Scope (PORT-03)

- **D-01:** Scan all `.devtools/*/.*compose.yml` files for `host:port:container` port mappings —
  extract the host-side port from every `ports:` entry across main service, UI companion, and
  monitoring exporter containers.
- **D-02:** Also scan all `.devtools/*/.env` files for port variables (lines matching
  `*_PORT=<number>`) as a secondary source. Compose file scanning is primary; .env is supplemental.
- **D-03:** Scan all host-mapped ports — main service ports AND UI companion ports (e.g.,
  redisinsight:8001, pgadmin, phpmyadmin, mongo_express) AND monitoring exporter ports. Do not
  skip any container type.
- **D-04:** If `.devtools/` does not exist (clean project, no services installed yet), perform a
  silent skip — no user-visible message, proceed directly to the port prompt. No conflicts are
  possible on a fresh project.

### Conflict Detection & Message (PORT-04)

- **D-05:** After the user enters a port number, compare it against the collected set of bound
  ports. If a match is found, identify the service by its `.devtools/<service>/` directory name.
- **D-06:** Conflict message format (contextual):
  `"Port <N> is already used by <service> — try <next_free_port>?"`
  Where `<next_free_port>` is the lowest port ≥ N+1 not in the bound-ports set.
- **D-07:** If multiple services share the same port (edge case), name the first one found.

### Re-Prompt Loop & Escape Hatch (PORT-05)

- **D-08:** The re-prompt loop presents a **3-option menu** after each conflict:
  1. "Try a different port" → asks for a new port number, then re-validates
  2. "Use internal only (no host binding)" → sets `ANSWERS[host_port]=false`, skips port, continues
     install (same as Phase 8 opt-out path)
  3. "Cancel service install" → aborts the entire install with a clean exit message
- **D-09:** There is no hard retry limit — the loop continues as long as the user keeps choosing
  "Try a different port". It resolves when they either find a free port or pick one of the
  other two menu options.
- **D-10:** The suggested free port in the conflict message (D-06) is a hint only. The user may
  enter any port they choose on the re-prompt — the suggestion is not pre-filled.

### Integration with Phase 8 Flow

- **D-11:** The collision guard activates only when `ANSWERS[host_port]=true`. The Phase 8 "no
  host port" path (opted out before port question) is unaffected.
- **D-12:** The scan happens **before** showing the port prompt (i.e., collect all bound ports
  into an in-memory set first, then ask). This avoids repeated file I/O on each re-prompt cycle.
- **D-13:** The guard applies to alias installs too — same port question and same re-prompt loop
  (alias installs use the same Step 5 Q&A path).

### Agent Discretion

- The exact regex / glob pattern for extracting ports from compose files and .env files is left
  to the planner / executor. Standard `ports:` YAML parsing and `KEY=NUMBER` env parsing are
  sufficient.
- Next-free-port suggestion (D-06) algorithm: linear scan from N+1 upward, checking against the
  bound-ports set, stop at first free value. Simple is fine — this is a hint, not a guarantee.

</decisions>

<refs>
## Canonical References

- `skills/add-service/SKILL.md` — single file modified by this phase (Step 5 port Q&A section)
- `.planning/phases/08-host-port-opt-in/08-CONTEXT.md` — prior decisions D-01–D-15 (the Phase 8
  context this phase builds on top of)
- `.planning/REQUIREMENTS.md` — PORT-03, PORT-04, PORT-05 (the three requirements for this phase)
</refs>
