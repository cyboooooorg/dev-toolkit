---
phase: 3
status: locked
---

# Phase 3 Context: Cross-Runtime Wiring & Distribution

## Key Decisions (Locked)

### D-01: Distribution Method
**Decision:** Publish to skills.sh ecosystem. Users install with `npx skills add Cyboooooorg/dev-tools`.  
**Rationale:** skills.sh is a growing open ecosystem for AI agent skills. The `skills` CLI handles installation to all major agent paths (GitHub Copilot, Claude Code, Cursor, etc.) automatically — no manual symlinking needed.  
**Impact:** Replaces the original plan of manually wiring `.github/skills/` and `.agents/skills/` in this repo.

### D-02: Repo Structure (Already Correct)
**Decision:** Keep canonical skill at `skills/add-service/SKILL.md`. This is the exact structure the `skills` CLI discovers.  
**Rationale:** The `skills` CLI scans for `skills/<name>/SKILL.md` by convention. No changes to structure needed.

### D-03: SKILL-01 / Trigger Language
**Decision:** Skill frontmatter description must explicitly enumerate all 5 services so AI runtimes match service-addition intents. Format: quoted string (YAML-safe, colons allowed).  
**Status:** DONE — fixed in `skills/add-service/SKILL.md` (commit 597bd26, quoted description).

### D-04: DIST-01 / Discovery Path Wiring
**Decision:** Use skills.sh distribution instead of manual `.github/skills/` and `.agents/skills/` symlinks within this repo. The `skills` CLI handles multi-agent path installation at the consumer side.  
**Rationale:** Manual wiring would only cover 2 agent paths; skills.sh covers 40+ agents automatically.

### D-05: README Content
**Decision:** README must cover:
1. What this repo is (a dev-tools skill collection)
2. Install command: `npx skills add Cyboooooorg/dev-tools`
3. Supported services list (Redis, RabbitMQ, PostgreSQL, MySQL/MariaDB, MongoDB)
4. Quick usage example (invoke with `/add-service redis` or just `/add-service`)
5. What files get written to `.devtools/` so users know what to expect
6. Note that `.devtools/.env` is gitignored (credentials stay local)

### D-06: Requirements Reinterpretation
**Decision:** SKILL-02 and DIST-01 originally said "symlinked into both `.github/skills/` and `.agents/skills/`". Reinterpret as: skill is *discoverable from this repo* via skills.sh in both Copilot and Claude environments. skills.sh already handles both paths at install time.  
**Rationale:** User explicitly chose skills.sh over manual symlinking.

### D-07: SKILL-03 Status
**Decision:** SKILL-03 (frontmatter enumerates all 5 services) is already satisfied by the quoted description in SKILL.md. Phase 3 plan should verify this, not re-implement it.

## What's Remaining for Phase 3

1. **03-01**: Verify skills.sh discoverability (frontmatter valid, structure correct), add any missing manifest metadata if needed
2. **03-02**: Write `README.md` (root-level) with install instructions + usage guide

## Requirements Coverage

| Req | Disposition |
|-----|-------------|
| SKILL-01 | Covered by SKILL.md description + trigger language (already implemented in Phase 2) |
| SKILL-02 | Covered by skills.sh multi-agent install (covers both Copilot `.agents/skills/` and Claude paths) |
| SKILL-03 | Already done — description quoted and enumerates all 5 services |
| DIST-01 | Covered by skills.sh — users run `npx skills add Cyboooooorg/dev-tools` |
| DIST-02 | README.md — to be written in 03-02 |
