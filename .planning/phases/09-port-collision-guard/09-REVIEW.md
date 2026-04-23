---
phase: 09-port-collision-guard
reviewed: 2025-07-14T00:00:00Z
depth: standard
files_reviewed: 1
files_reviewed_list:
  - skills/add-service/SKILL.md
findings:
  critical: 0
  warning: 3
  info: 2
  total: 5
status: issues_found
---

# Phase 9: Code Review Report

**Reviewed:** 2025-07-14
**Depth:** standard
**Files Reviewed:** 1 (`skills/add-service/SKILL.md`)
**Status:** issues_found

## Summary

Phase 9 inserts two additions into Step 5 of `skills/add-service/SKILL.md`:

1. **Port Scan sub-step** — collects all host-bound ports from existing `.devtools/` installs
   into a `BOUND_PORTS` map before Q2 is shown.
2. **Conflict-detection loop** — after the user enters a port in Q2, checks it against
   `BOUND_PORTS`, shows a conflict message with a suggested `NEXT_FREE` port, and presents a
   3-option escape menu (try different port / use internal only / cancel).

The overall design is sound. Loop termination is correct for all three menu options, Phase 8
integration (Option 2 → `ANSWERS[host_port]=false`) is consistent with downstream steps 9a
and 10a, alias subdirectory coverage is complete, and the `NEXT_FREE` suggestion is
guaranteed to be free of existing conflicts by definition.

Three issues merit attention before this ships:

- **WR-01** — The primary scan's grep matches only `127.0.0.1:`-prefixed port lines. Other
  valid Docker Compose port binding formats are silently skipped.
- **WR-02** — The `test -d .devtools` absence-check inside the Port Scan sub-step is
  unreachable dead code given the step execution order; it could mislead AI runtimes.
- **WR-03** — The instruction for unresolvable `${ENV_VAR}` compose references ("note the
  env var name for .env cross-reference") has no concrete implementation path in the failure
  case, leaving a port silently absent from `BOUND_PORTS`.

---

## Warnings

### WR-01: Primary scan grep misses non-`127.0.0.1` port binding formats

**File:** `skills/add-service/SKILL.md:346`

**Issue:**

The scan command is:

```bash
find .devtools -name "*.compose.yml" -print0 | xargs -0 grep -h "127\.0\.0\.1:" 2>/dev/null
```

This only matches lines that begin with the `127.0.0.1:` loopback prefix. Three other valid
Docker Compose port formats are silently skipped:

| Format | Example | Matched? |
|--------|---------|----------|
| Loopback | `"127.0.0.1:6379:6379"` | ✓ |
| All-interfaces | `"0.0.0.0:6379:6379"` | ✗ |
| Bare (any interface) | `"6379:6379"` | ✗ |
| IPv6 | `"[::1]:6379:6379"` | ✗ |

The skill's own generated templates always use `127.0.0.1:`, so skill-managed files are
safe. However, a user who manually added a service to `.devtools/` using bare `HOST:CONTAINER`
format, or a future template change, would silently bypass collision detection — the conflicting
port would appear free and be assigned without warning.

**Fix:**

Broaden the extraction regex to capture the host-side port number regardless of bind address.
One approach: replace the `127\.0\.0\.1:` grep with a pattern that matches the Docker Compose
ports list entry, then extract the second-to-last colon-delimited token:

```bash
# Capture any quoted port mapping line and extract host port
find .devtools -name "*.compose.yml" -print0 \
  | xargs -0 grep -hE '^\s*-\s*"?[^:]+:[^:]+:[0-9]+"?' 2>/dev/null
```

Then strip the bind-address prefix when extracting the host-port token (the value between
the first `:` and the second `:`), regardless of whether it is `127.0.0.1`, `0.0.0.0`, or
absent.

Alternatively, add a clear scope note: _"This scan covers only `127.0.0.1:`-bound ports.
Ports bound to `0.0.0.0` or using bare `HOST:CONTAINER` format are not detected."_ That
makes the limitation explicit rather than a hidden silent skip.

---

### WR-02: `.devtools/` absence check inside Port Scan sub-step is unreachable dead code

**File:** `skills/add-service/SKILL.md:334-340`

**Issue:**

The Port Scan sub-step opens with:

```
1. Check whether `.devtools/` exists:
   - If `.devtools/` does not exist (clean project, first install): set
     BOUND_PORTS={} (empty) and skip straight to Q2.
   - If `.devtools/` exists: continue to scan steps 2–3 below.
```

By the time Step 5 executes, `.devtools/` is **always guaranteed to exist**.  Step 2
("First-Run Setup") runs unconditionally before Step 5 and creates `.devtools/` when it is
absent. No execution path reaches Step 5 with `.devtools/` missing.

An AI runtime following these instructions literally could misread this as: _"on first
install, skip the scan"_ — which is correct behaviour for a different reason (nothing to scan
yet). But the dead branch invites misunderstanding: a runtime might treat the check as a
meaningful gate and skip later, valid scenarios where `.devtools/` exists but is empty.

**Fix:**

Remove the dead branch entirely. Replace with:

```
1. Scan compose files under `.devtools/` (created in Step 2 if this is the first install).
   If no compose files exist yet, the scan produces empty results and BOUND_PORTS={}.
```

This is accurate, simpler, and does not imply a false exit path.

---

### WR-03: `${ENV_VAR}` resolution failure produces a silent false negative in BOUND_PORTS

**File:** `skills/add-service/SKILL.md:350-352`

**Issue:**

When the primary scan encounters a compose port line such as
`"127.0.0.1:${REDIS_PORT}:6379"`, the instruction says:

> extract the env-var's resolved value **if available**, or **note the env var name for
> .env cross-reference**.

The phrase "note the env var name for .env cross-reference" has no concrete implementation.
The supplemental scan (step 3) independently scans `_PORT=[0-9]+` patterns — it does not
consume any "notes" from step 2 and has no special logic to resolve env vars that step 2
flagged as unresolved.

If the corresponding `.env` file is absent (corrupted install, manual deletion) or does not
contain the expected `_PORT=` entry:

- Step 2 leaves the port unresolved.
- Step 3 also finds nothing (no `.env` to scan).
- The port is **never added to `BOUND_PORTS`**.

Result: a genuinely bound port is invisible to the collision guard — it appears free and will
be silently assigned to the new service, causing a real port conflict at `docker compose up`.

The practical risk is low (requires a broken prior install), but the instruction gives no
fallback behaviour for this case and the "cross-reference" wording implies a concrete
mechanism that doesn't exist.

**Fix:**

Either (a) explicitly define the failure behaviour:

> "If the `.env` file for that slug is absent or does not contain the referenced variable,
> skip that port and continue. The scan is best-effort for broken installs."

Or (b) restructure the two steps so that step 2 builds a set of `{slug → [env_var_names]}`
that need resolution, and step 3 resolves them — making the cross-reference explicit:

> "3. For each `${ENV_VAR}` reference not yet resolved in step 2, look up `ENV_VAR` in
> `.devtools/<slug>/.env` and add the resolved value to `BOUND_PORTS`. If the file or key
> is absent, skip."

---

## Info

### IN-01: `NEXT_FREE` suggestion has no upper-bound guard

**File:** `skills/add-service/SKILL.md:398-400`

**Issue:**

The next-free-port algorithm scans `entered_port + 1`, `entered_port + 2`, … upward
indefinitely until it finds a port not in `BOUND_PORTS`. There is no check that
`NEXT_FREE ≤ 65535`. If `BOUND_PORTS` happened to contain a contiguous range reaching
65535 (theoretically impossible in practice for a local dev setup), the algorithm would
suggest an invalid port number.

**Fix:**

Add a ceiling: `… until a value not present in BOUND_PORTS **and ≤ 65535** is found. If no
free port exists below 65535, display: "No free port found above [entered_port] — please
enter a port manually."` This is a degenerate case but makes the spec unambiguous.

---

### IN-02: Q2 port input has no specified validation range

**File:** `skills/add-service/SKILL.md:386-388`

**Issue:**

Q2 asks for a port number with no stated valid range. An AI runtime presented with user
input of `0`, `65536`, or a non-numeric string would run the conflict-check against
`BOUND_PORTS` unchanged. A non-numeric value would not match any entry and would proceed
to `ANSWERS[port]=<invalid>`, producing a malformed compose file downstream.

**Fix:**

Add a brief validation note to Q2:

> "Accept only integers in the range 1–65535. If the user enters an out-of-range or
> non-numeric value, re-prompt: 'Port must be a number between 1 and 65535. Try again: _'"

---

_Reviewed: 2025-07-14_
_Reviewer: gsd-code-reviewer (AI agent)_
_Depth: standard_
