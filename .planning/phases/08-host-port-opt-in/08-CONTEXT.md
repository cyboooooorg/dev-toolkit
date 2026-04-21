# Phase 8: Host Port Opt-In - Context

**Gathered:** 2026-04-21
**Status:** Ready for planning

<domain>
## Phase Boundary

Add a yes/no host-port opt-in gate to the `add-service` skill Q&A flow. The question appears
**before any port-number question**. If user opts out, the compose output omits the main
service's host port mapping. If user opts in, the flow proceeds to the port-number question as
before.

**In scope:** The opt-in question (PORT-01), compose output modification when opted out (PORT-02),
behaviour for multi-port service blocks (RabbitMQ), and UI companion port handling.

**Out of scope:** Port collision detection and re-prompt loop (PORT-03, PORT-04, PORT-05 — Phase 9).

</domain>

<decisions>
## Implementation Decisions

### Opt-In Question Placement & Wording

- **D-01:** The opt-in question is inserted as the **first question in Step 5 (Universal Q&A)**,
  before the existing "Port? [default: X]" question.
- **D-02:** Exact wording: `"Expose this service on a host port? [y/N]"` (default: N — internal
  Docker network only). Matches PORT-01 intent from the roadmap success criteria.
- **D-03:** Store answer as `ANSWERS[host_port]=true/false`.

### Port Question Skip When Opted Out

- **D-04:** When `ANSWERS[host_port]=false` (user opted out), **skip the "Port? [default: X]"
  question entirely** — do not ask for a port number.
- **D-05:** When opted out, the port env var (e.g. `REDIS_PORT`) is **not written** to
  `.devtools/${SERVICE_SLUG}/.env`. There is no host port to record.
- **D-06:** When `ANSWERS[host_port]=true`, the flow continues to the existing port question
  unchanged — no other changes to the Q&A path.

### Compose Output — Main Service Port

- **D-07:** When opted out, **remove only the main service port line** from the `ports:` block
  of the main service container. Do not touch other service containers (UI companion, monitoring
  exporter).
- **D-08:** If removing the main service port line leaves the `ports:` block empty (e.g. Redis,
  which only has one port entry), **remove the entire `ports:` block**. Never write an empty
  `ports:` list.
- **D-09:** Services where the service block has both a main port and a UI/management port
  (e.g. RabbitMQ with `RABBITMQ_PORT:5672` and `RABBITMQ_UI_PORT:15672` in the same block):
  **remove only the main port line**, keep the UI/management port line. The `ports:` block
  remains with just the UI port entry.
- **D-10:** Template modification uses the existing **in-memory content modification** pattern
  (same approach as Redis password stripping). No separate no-port template variants needed.

### UI Companion Container Ports

- **D-11:** UI companion containers (RedisInsight, pgAdmin, phpMyAdmin, Mongo Express) are
  **always host-mapped**, regardless of the main service opt-in/opt-out choice. The browser
  requires host access to reach the UI.
- **D-12:** The UI companion opt-in question (Step 6) is unaffected by the host-port opt-out.
  If the user enables a UI companion, its port is always written to the compose output.

### Monitoring Exporter Containers

- **D-13:** Monitoring exporter containers (`redis_exporter`, `rabbitmq_exporter`, etc.)
  already have **no `ports:` block** in their templates — they are internal-only by design.
  No changes needed for exporters.

### Step 8 Confirmation Summary

- **D-14:** When user has opted out of host port, the summary table shows:
  ```
  Port:           internal only (no host port)
  ```
  instead of a port number. This makes the opt-out decision visible before the user confirms.
- **D-15:** When user has opted in, the summary table shows the port number as before:
  ```
  Port:           6379
  ```

### the agent's Discretion

- Exact string matching logic for removing the main port line from the in-memory compose
  content (e.g., regex on the port mapping format `"127.0.0.1:${SERVICE_PORT}:XXXX"`)
- Whether to strip trailing whitespace/blank lines after port block removal
- Error handling if the template `ports:` block has an unexpected format

</decisions>

<canonical_refs>
## Canonical References

**Downstream agents MUST read these before planning or implementing.**

### Skill Entry Point
- `skills/add-service/SKILL.md` — The single canonical SKILL.md. Step 5 (Universal Q&A) and
  Step 9a (Write Compose File) are the primary modification points for Phase 8.

### Compose Templates (affected by port opt-out)
- `compose-templates/redis/redis.compose.yml` — Single port entry; becomes portless when opted out
- `compose-templates/rabbitmq/rabbitmq.compose.yml` — Two ports in same block; only main port removed
- `compose-templates/postgres/postgres.compose.yml` — Single port entry
- `compose-templates/mysql/mysql.compose.yml` — Single port entry
- `compose-templates/mongodb/mongodb.compose.yml` — Single port entry

### Requirements
- `.planning/REQUIREMENTS.md` — PORT-01 and PORT-02 are the only requirements for this phase

</canonical_refs>

<code_context>
## Existing Code Insights

### Reusable Assets
- In-memory template modification pattern (SKILL.md Step 9a): already used to strip Redis
  password args when user opts out of auth. Same approach applies for port line removal.

### Established Patterns
- All Q&A uses `AskUserQuestion` in Step 5; new opt-in question follows the same pattern
- `ANSWERS[key]` dictionary stores all user responses; `ANSWERS[host_port]` is a new key
- Templates are fetched via `curl` at runtime and modified before writing — no template files
  need to change for Phase 8

### Integration Points
- **Step 5 Universal Q&A** — Insert opt-in question before the existing Q1 (Port)
- **Step 9a (Write Compose File)** — Add conditional port-line removal to in-memory content
- **Step 10a (.env write)** — Conditionally skip writing the port env var when `host_port=false`
- **Step 8 Summary** — Update port row display to reflect opt-out state

</code_context>

<specifics>
## Specific Ideas

- RabbitMQ has two ports in one `ports:` block (`RABBITMQ_PORT` + `RABBITMQ_UI_PORT`). When
  user opts out, only the main AMQP port line (`RABBITMQ_PORT:5672`) is removed; the management
  UI port line stays so the web UI remains browser-accessible.
- The summary line when opted out: `Port: internal only (no host port)` — exact wording chosen
  to be clear and consistent with the success criteria language.

</specifics>

<deferred>
## Deferred Ideas

None — discussion stayed within phase scope.

</deferred>

---

*Phase: 08-host-port-opt-in*
*Context gathered: 2026-04-21*
