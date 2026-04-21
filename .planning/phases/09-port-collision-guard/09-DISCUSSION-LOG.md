# Phase 9: Port Collision Guard - Discussion Log

**Date:** 2026-04-21
**Mode:** Interactive (discuss mode, one-question-at-a-time)
**Areas discussed:** Opt-out escape hatch, Conflict message detail, Empty .devtools/ handling,
UI companion port inclusion

---

## Area 1: Opt-out Escape Hatch

**Q:** When a collision is detected and the skill re-prompts for a new port, how does the user
escape back to "internal only / no host port"?

**Options presented:**
- Show explicit option each re-prompt: "Port? [default: 6379, or 0 to use internal only]"
- Show explicit option with Enter shortcut
- Auto-offer after 2 failures
- You decide

**User response (free text):** "Use a menu usable with arrows has in GSD"

**Follow-up Q:** When a collision is detected, what should the menu offer?

**Options presented:**
- Two-option menu: "Try a different port" / "Use internal only (no host binding)"
- Three-option menu: "Try a different port" / "Use internal only" / "Cancel service install"
- Two-option menu with pre-suggested next safe port

**Selected:** Three-option menu: "Try a different port" / "Use internal only" / "Cancel service install"

**Decision recorded:** D-08, D-09

---

## Area 2: Conflict Message Detail

**Q:** When a port conflict is detected, how detailed should the conflict message be?

**Options presented:**
- Brief: "Port 6379 is already used by redis." — service name only
- Descriptive: "Port 6379 is already used by redis (installed at .devtools/redis/)."
- Contextual: "Port 6379 is already used by redis — try 6380?" — name + suggested free port

**Selected:** Contextual: "Port 6379 is already used by redis — try 6380?"

**Decision recorded:** D-05, D-06, D-07, D-10

---

## Area 3: Empty .devtools/ / No Existing Services

**Q:** When .devtools/ doesn't exist yet (clean project, first install), what should the skill
do during the port scan step?

**Options presented:**
- Silent skip — proceed straight to port prompt
- Inform silently — note internally, no user-visible message
- Brief notice — "No existing services found, any port is safe."

**Selected:** Silent skip — no .devtools/ means no conflicts, proceed straight to port prompt

**Decision recorded:** D-04

---

## Area 4: UI Companion Port Inclusion

**Q:** Should the port scanner include UI companion ports (RedisInsight, pgAdmin, phpmyadmin,
mongo_express) as potential conflict sources?

**Options presented:**
- Yes — scan all host-mapped ports including UI companions
- Yes, but only if they appear in .devtools/*/*.env
- No — only scan main service ports

**Selected:** Yes — scan all host-mapped ports including UI companions (redisinsight:8001,
pgadmin, phpmyadmin, mongo_express)

**Decision recorded:** D-03
