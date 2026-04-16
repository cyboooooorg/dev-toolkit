# Phase 5: New Instance Alias Prompt — Context

**Gathered:** 2026-04-16
**Status:** Ready for planning

<domain>
## Phase Boundary

Improve the alias prompt UX for the `add-service` skill when a user installs a service into an
already-occupied slot. This covers: (a) the conflict-resolution alias prompt (Branch B of Step 1a),
(b) alias input validation and normalization, (c) alias conflict re-prompting, and (d) an optional
proactive alias prompt offered on first (non-conflicting) installs.

**In scope:** Changes to `SKILL.md` alias prompt phrasing, alias validation/normalization logic,
alias conflict re-prompt loop, and the new optional alias prompt on first install.

**Out of scope:** Port conflict detection (violates zero-dependency constraint — v2 concern),
changes to the rename-before-add flow (Phase 4), changes to template contents, changes to
cross-runtime wiring.

</domain>

<decisions>
## Implementation Decisions

### Alias Prompt UX (Branch B — conflict resolution)

- **D-01:** The prompt stays minimal — no listing of existing instances, no examples banner.
  Just ask for the alias with a concise inline hint.
- **D-02:** The user types only the **alias suffix** (e.g. `cache`), not the full slug
  (`redis-cache`). The skill derives the slug as `${SERVICE}-${ALIAS}`.
- **D-03:** The prompt must show the resulting slug inline so the user knows what they'll get:
  ```
  Enter alias (e.g. 'cache' → redis-cache): _
  ```

### Alias Validation & Normalization

- **D-04:** If the alias contains uppercase letters or invalid characters, **silently normalize**:
  lowercase the input, strip or replace special chars with hyphens, trim leading/trailing
  hyphens. No error message — just proceed with the normalized slug.
- **D-05:** After normalization, the skill proceeds **silently** — it does NOT announce what
  the alias was normalized to. The result is visible in the done summary slug anyway.
- **D-06:** If the alias is blank or produces an **empty slug after normalization**, the skill
  re-prompts **once** with a clear message:
  ```
  Alias cannot be empty. Enter alias (e.g. 'cache' → redis-cache): _
  ```
  If the second attempt is also empty, exit: `"Nothing written. Exiting."`
- **D-07:** If the alias equals the service name (e.g. alias `redis` for service `redis`),
  the skill **rejects it and re-prompts** with:
  ```
  Alias must differ from the service name. Enter alias (e.g. 'cache' → redis-cache): _
  ```

### Alias Conflict Re-prompt

- **D-08:** If the provided alias slug already exists (`.devtools/${SERVICE_SLUG}/` already
  installed), the skill re-prompts with:
  ```
  redis-session is already installed. Enter a different alias (or 'cancel'): _
  ```
- **D-09:** The re-prompt loop is **unlimited** — the skill keeps asking until the user
  provides a free alias or types `cancel`. Typing `cancel` exits with `"Nothing written. Exiting."`

### Proactive Alias on First Install

- **D-10:** On a **first install** (when the service slot is not yet occupied), the skill offers
  an optional alias prompt **before** the port/version/credentials Q&A:
  ```
  Install with a custom alias? (optional — press Enter to install as 'redis'): _
  ```
- **D-11:** If the user presses Enter (no alias), the install proceeds as a standard install
  (slug = `${SERVICE}`). The Q&A flows as normal.
- **D-12:** If the user provides an alias on first install, the skill validates it using the
  same normalization rules (D-04–D-07) and sets `MODE=alias`. The Q&A then runs for the
  aliased slug (e.g. `redis-cache`).
- **D-13:** The proactive alias prompt is placed **before Step 3** (the port/version/credentials
  Q&A), so the slug is locked before any service-specific naming is derived.

### the agent's Discretion

- Exact loop structure for the unlimited re-prompt (while-loop vs recursive step reference) —
  agent picks the most readable approach for SKILL.md's prose-step format.
- Whether to fold D-06's one-retry-then-cancel into the same unlimited-loop pattern as D-09,
  or keep them distinct — agent decides based on consistency.

</decisions>

<canonical_refs>
## Canonical References

**Downstream agents MUST read these before planning or implementing.**

- `skills/add-service/SKILL.md` — primary file to update; read completely before planning.
  Key sections: Step 1 (detection), Step 1a (merge/rename flow, Branch B), Step 2 (first-run
  setup, where the new proactive alias prompt will be inserted), Step 12 (alias substitution).
- `.planning/phases/02-skill-core-interactive-flow-merge-logic/02-CONTEXT.md` — Phase 2
  alias decisions (D-24–D-27): alias naming convention, idempotency rule. Phase 5 extends
  these, not replaces them.
- `.planning/phases/04-subfolder-output-structure-per-service-subdirectory-layout-i/04-CONTEXT.md` —
  Phase 4 decisions (D-15–D-18): rename-before-add flow. Phase 5's Branch B comes after
  this flow completes.

</canonical_refs>

<code_context>
## Existing Code Insights

### Reusable Assets

- `skills/add-service/SKILL.md` Step 1a Branch B — existing alias prompt (to be replaced/extended by Phase 5)
- `skills/add-service/SKILL.md` Step 12 — alias substitution logic (no changes needed here)

### Established Patterns

- SKILL.md uses prose-step format with `Ask:` blocks for interactive prompts — new prompts
  must follow the same format.
- Alias validation rules already referenced in Branch B (`lowercase, letters/numbers/hyphens only`);
  Phase 5 makes the normalization behavior explicit rather than leaving it implied.

### Integration Points

- Step 1a Branch B: replace the bare `"Add another instance with an alias, or cancel?"` with
  the new slug-aware prompt (D-03) and re-prompt loop (D-08/D-09).
- Between Step 1 and Step 2: insert the proactive alias prompt (D-10–D-13) for first installs.

</code_context>

<specifics>
## Specific Ideas

- User wants the resulting slug shown inline in the prompt — `'cache' → redis-cache` format.
- Port conflict detection (reading existing instance's port from `.env` and showing it as a
  warning) was discussed but deferred — see Deferred Ideas.

</specifics>

<deferred>
## Deferred Ideas

- **Port awareness on multi-instance installs:** When installing a second instance, show the
  existing instance's configured port during the port Q&A so the user can pick a non-conflicting
  one. Deferred to v2 — all values would come from `.devtools/<slug>/.env` (no dependencies),
  but scoped out of Phase 5 to keep the phase focused.
- **Active port conflict detection:** Checking if a port is actually in use on the system is
  explicitly out of scope (`REQUIREMENTS.md`) due to the zero-dependency constraint.

</deferred>

---

*Phase: 05-new-instance-alias-prompt-ask-user-for-alias-when-installing*
*Context gathered: 2026-04-16*
