# Phase 8: Host Port Opt-In - Discussion Log

> **Audit trail only.** Do not use as input to planning, research, or execution agents.
> Decisions are captured in CONTEXT.md — this log preserves the alternatives considered.

**Date:** 2026-04-21
**Phase:** 08-host-port-opt-in
**Areas discussed:** Scope of opt-out, UI companion ports, Port question fate, Step 8 summary display

---

## Scope of Opt-Out

| Option | Description | Selected |
|--------|-------------|----------|
| Remove the entire `ports:` block from the service | Both main port and UI port (e.g. RabbitMQ 5672 + 15672) go internal-only | |
| Remove only the main service port line, keep UI port | Management UI port stays host-mapped | ✓ |
| You decide | Agent's discretion | |

**User's choice:** Remove only the main service port line, keep UI/management port if present.

**Notes:** Rule applies consistently across all services — not just RabbitMQ. For services with only one port entry (Redis, Postgres, MySQL, MongoDB), removing the main port line leaves an empty `ports:` block, which should be removed entirely.

---

## UI Companion Ports

| Option | Description | Selected |
|--------|-------------|----------|
| UI companion always uses host port | Browser needs host access — always mapped regardless of main service choice | ✓ |
| UI companion inherits the opt-out | If main service is internal-only, UI is too | |
| Ask separately per companion | One opt-in question per UI port | |

**User's choice:** UI companion always uses host port.

**Notes:** UI companions (RedisInsight, pgAdmin, phpMyAdmin, Mongo Express) are separate service containers and always need browser access. Their `ports:` entries are not affected by the main service opt-out.

---

## Port Question Fate

| Option | Description | Selected |
|--------|-------------|----------|
| Skip the port question entirely | No port number asked, no REDIS_PORT written to .env | ✓ |
| Still ask the port question | Record port in .env even though not host-mapped | |

**User's choice:** Skip the port question entirely — no env var written either.

**Notes:** Since the port env var (`REDIS_PORT`, etc.) is only used in the `ports:` host-mapping line, and that line is removed when opted out, there's no reason to ask for or record the value.

---

## Step 8 Summary Display

| Option | Description | Selected |
|--------|-------------|----------|
| Show "internal only (no host port)" | Makes the opt-out visible in confirmation table | ✓ |
| Omit the port row entirely | Nothing to show, don't show it | |
| You decide | Agent's discretion | |

**User's choice:** Show `Port: internal only (no host port)` in the confirmation table.

**Notes:** Makes the decision visible and explicit before the user confirms the write. When opted in, the port number displays as before.

---

## the agent's Discretion

- Exact regex/matching logic for removing the main port line from in-memory compose content
- Whitespace/blank line cleanup after port block modification
- Error handling for unexpected template formats

## Deferred Ideas

None.
