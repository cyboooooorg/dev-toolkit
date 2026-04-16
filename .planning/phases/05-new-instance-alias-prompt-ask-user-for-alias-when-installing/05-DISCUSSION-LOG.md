# Phase 5: New Instance Alias Prompt — Discussion Log

> **Audit trail only.** Do not use as input to planning, research, or execution agents.
> Decisions are captured in CONTEXT.md — this log preserves the alternatives considered.

**Date:** 2026-04-16
**Phase:** 05-new-instance-alias-prompt-ask-user-for-alias-when-installing
**Areas discussed:** Alias prompt UX, Alias validation & error feedback, Alias conflict re-prompt, Proactive alias at first install

---

## Alias Prompt UX

| Option | Description | Selected |
|--------|-------------|----------|
| Show existing instances | List what's already in .devtools/ so the user picks a non-clashing name | |
| Show a naming example only | Give a concrete example like 'cache', 'session', 'primary' | |
| Keep it minimal | Just ask for the alias with no extra context (current behavior) | ✓ |

**User's choice:** Keep it minimal

| Option | Description | Selected |
|--------|-------------|----------|
| Ask for alias only | Single prompt for alias suffix (e.g. 'cache' → redis-cache) | ✓ |
| Ask for full slug | User types the complete name 'redis-cache' | |
| Ask for a label/description | User types a human-readable label; skill generates slug | |

**User's choice:** Ask for alias only — suffix approach

| Option | Description | Selected |
|--------|-------------|----------|
| Show the resulting slug inline | Prompt shows 'Enter alias (e.g. cache → redis-cache):' | ✓ |
| Show format rules only | Prompt shows '(lowercase, letters/numbers/hyphens)' | |
| Neither | Bare prompt with no annotation | |

**User's choice:** Show the resulting slug inline

---

## Alias Validation & Error Feedback

| Option | Description | Selected |
|--------|-------------|----------|
| Silently normalize | Lowercase and strip/replace invalid chars automatically, then proceed | ✓ |
| Reject and re-prompt | Show error and ask again if input contains invalid chars | |
| Reject and cancel | Invalid alias exits the skill | |

**User's choice:** Silently normalize

| Option | Description | Selected |
|--------|-------------|----------|
| Show what it normalized to | Announce the final slug before proceeding | |
| Proceed silently | Use the normalized slug without announcing it | ✓ |

**User's choice:** Proceed silently

| Option | Description | Selected |
|--------|-------------|----------|
| Re-prompt once | Tell user alias results in empty slug and ask again | ✓ |
| Cancel | Exit with 'Nothing written. Exiting.' | |

**User's choice:** Re-prompt once for empty slug

| Option | Description | Selected |
|--------|-------------|----------|
| Allow it | Alias can equal the service name | |
| Reject and re-prompt | Alias must differ from the service name to avoid confusion | ✓ |

**User's choice:** Reject and re-prompt when alias equals service name

---

## Alias Conflict Re-prompt

| Option | Description | Selected |
|--------|-------------|----------|
| Re-prompt | Tell user that alias is taken and ask for a different one | ✓ |
| Cancel | Exit with 'redis-session is already installed — nothing to do.' | |
| Re-prompt up to N times, then cancel | Allow a few retries before giving up | |

**User's choice:** Re-prompt

| Option | Description | Selected |
|--------|-------------|----------|
| Unlimited re-prompts | Keep asking until user picks a free alias or types 'cancel' | ✓ |
| Re-prompt once, then cancel | If second alias is also taken, exit cleanly | |

**User's choice:** Unlimited re-prompts

---

## Proactive Alias at First Install

| Option | Description | Selected |
|--------|-------------|----------|
| No | Alias prompt is conflict-only; first install always lands in standard slot | |
| Yes | Ask 'Install with a custom alias? (optional)' even on first install | ✓ |

**User's choice:** Yes — offer optional alias on first install

| Option | Description | Selected |
|--------|-------------|----------|
| Before the Q&A | Ask for alias before port/version/credentials | ✓ |
| After the Q&A | Ask for alias just before writing files | |

**User's choice:** Before the Q&A

**Notes:** User also raised port conflict checking — determined to be out of scope
(REQUIREMENTS.md explicitly excludes automatic port conflict detection due to zero-dependency
constraint). Deferred to v2. A lighter version (reading existing instance's port from
`.devtools/<slug>/.env`) was also deferred as a v2 concern.

---

## the agent's Discretion

- Loop structure for the unlimited alias re-prompt (while-loop vs recursive step reference)
- Whether to unify the empty-slug one-retry with the unlimited conflict re-prompt loop

## Deferred Ideas

- Port awareness on multi-instance installs (reading existing port from `.env`)
- Active port conflict detection (explicitly out of scope — zero-dependency constraint)
