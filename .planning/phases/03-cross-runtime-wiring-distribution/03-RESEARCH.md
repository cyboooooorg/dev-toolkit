# Phase 3: Cross-Runtime Wiring & Distribution - Research

**Researched:** 2026-04-10
**Domain:** skills.sh ecosystem, agent skill discovery, README documentation
**Confidence:** HIGH

---

<user_constraints>
## User Constraints (from CONTEXT.md)

### Locked Decisions

**D-01: Distribution Method**
Publish to skills.sh ecosystem. Users install with `npx skills add Cyboooooorg/dev-tools`.
Replaces the original plan of manually wiring `.github/skills/` and `.agents/skills/` in this repo.

**D-02: Repo Structure (Already Correct)**
Keep canonical skill at `skills/add-service/SKILL.md`. This is the exact structure the `skills` CLI discovers. No changes to structure needed.

**D-03: SKILL-01 / Trigger Language**
Skill frontmatter description must explicitly enumerate all 5 services. Format: quoted string (YAML-safe). Status: DONE — fixed in `skills/add-service/SKILL.md` (commit 597bd26).

**D-04: DIST-01 / Discovery Path Wiring**
Use skills.sh distribution instead of manual `.github/skills/` and `.agents/skills/` symlinks. The `skills` CLI handles multi-agent path installation at the consumer side.

**D-05: README Content**
README must cover:
1. What this repo is (a dev-tools skill collection)
2. Install command: `npx skills add Cyboooooorg/dev-tools`
3. Supported services list (Redis, RabbitMQ, PostgreSQL, MySQL/MariaDB, MongoDB)
4. Quick usage example (invoke with `/add-service redis` or just `/add-service`)
5. What files get written to `.devtools/` so users know what to expect
6. Note that `.devtools/.env` is gitignored (credentials stay local)

**D-06: Requirements Reinterpretation**
SKILL-02 and DIST-01 reinterpreted: skill is *discoverable from this repo* via skills.sh in both Copilot and Claude environments. skills.sh handles both paths at install time.

**D-07: SKILL-03 Status**
Already satisfied by the quoted description in SKILL.md. Phase 3 plan should verify this, not re-implement it.

### Agent's Discretion
- How to split 03-01 and 03-02 plans (may fold 03-01 into 03-02 if verification is minimal)

### Deferred Ideas (OUT OF SCOPE)
- MCP server implementation (mentioned in requirements as explicitly out of scope)
- Manual `.github/skills/` or `.agents/skills/` symlinks in this repo
</user_constraints>

---

<phase_requirements>
## Phase Requirements

| ID | Description | Research Support |
|----|-------------|------------------|
| SKILL-01 | AI invokes skill when user says "add Redis" / "install RabbitMQ" | Covered by existing SKILL.md description (already quoted, enumerates services). Verify no further changes needed. |
| SKILL-02 | Skill installed in both Copilot (.agents/skills) and Claude (.claude/skills) paths | skills.sh `npx skills add` handles both paths automatically at consumer install time. No action in this repo. |
| SKILL-03 | Frontmatter explicitly enumerates all 5 services | Already implemented. Phase 3 verifies only. |
| DIST-01 | Canonical SKILL.md discoverable by skills.sh `npx skills add Cyboooooorg/dev-tools` | Repo structure `skills/<name>/SKILL.md` is exactly what skills CLI scans. Verified correct. |
| DIST-02 | README.md explains installation and supported services | Existing README is placeholder. Needs full rewrite. Primary work of 03-02. |
</phase_requirements>

---

## Summary

Phase 3 is almost entirely a documentation and verification phase. The hard work (SKILL.md, templates, interactive flow) is complete. Two deliverables remain: (1) confirm the repo is correctly wired for skills.sh discovery, and (2) write a useful README.md that replaces the current 4-line placeholder.

The skills.sh CLI (`npx skills`, npm package `skills` v1.5.0) discovers skills by recursively scanning a repo for directories containing a `SKILL.md` file. It only requires `name` and `description` as string frontmatter fields. The current `skills/add-service/SKILL.md` satisfies both requirements and is already valid. No additional manifest file is needed.

The key insight for 03-01 is that it is purely a **verification** task with zero file writes. The repo structure and SKILL.md frontmatter are already correct. The plan should confirm this via a read-and-check, not create or modify anything. This is thin enough that it could be a single Wave-0 + Wave-1 plan with one task each.

**Primary recommendation:** Keep 03-01 as a distinct (minimal) verification plan. Keep 03-02 as the README plan. Two plans makes intent traceable to requirements (DIST-01 vs DIST-02) and gives a clean verification checkpoint before writing user-facing docs.

---

## Standard Stack

### Core
| Library | Version | Purpose | Why Standard |
|---------|---------|---------|--------------|
| `skills` npm CLI | 1.5.0 (current) | Installs skills from repos into agent paths | Official skills.sh CLI, used by all agents in the ecosystem |

### Supporting
None — this phase writes a static Markdown file and verifies YAML. No dependencies.

**Installation (for users):**
```bash
npx skills add Cyboooooorg/dev-tools
```
No install needed in this repo.

**Version verification:** `npm view skills version` → `1.5.0` [VERIFIED: npm registry]

---

## Architecture Patterns

### skills.sh Discovery Logic
[VERIFIED: github.com/vercel-labs/skills/src/skills.ts]

The `skills` CLI scans a repo for skill directories by:
1. Recursively walking all directories (up to depth 5)
2. Skipping: `node_modules`, `.git`, `dist`, `build`, `__pycache__`
3. Any directory containing a `SKILL.md` file is treated as a skill
4. `SKILL.md` is parsed for frontmatter — only `name` and `description` are required; both must be strings

**Critical implication:** The structure `skills/add-service/SKILL.md` is exactly what the CLI finds. **No additional manifest file is needed.** The optional `plugin-manifest.ts` handles `.claude-plugin/marketplace.json` for multi-plugin setups, which is irrelevant here.

### Agent Installation Paths (skills.sh)
[VERIFIED: github.com/vercel-labs/skills/src/agents.ts]

When a user runs `npx skills add Cyboooooorg/dev-tools`, the CLI installs symlinks (or copies) into each detected agent's skill directory:

| Agent | Project Path | Global Path |
|-------|-------------|-------------|
| GitHub Copilot | `.agents/skills/` | `~/.copilot/skills/` |
| Claude Code | `.claude/skills/` | `~/.claude/skills/` |
| Cursor | `.agents/skills/` | `~/.cursor/skills/` |
| Codex | `.agents/skills/` | `~/.codex/skills/` |
| Cline | `.agents/skills/` | `~/.agents/skills/` |
| Gemini CLI | `.agents/skills/` | `~/.gemini/skills/` |
| 40+ more | varies | varies |

**Key insight:** Most agents use `.agents/skills/` as their project-level path. GitHub Copilot and many others all converge on this. Claude Code is the main exception (`.claude/skills/`). The CONTEXT.md original plan of "symlink into both `.github/skills/` and `.agents/skills/`" would have only covered 2 paths; skills.sh covers 40+.

### SKILL.md Frontmatter — Required vs. Optional Fields
[VERIFIED: github.com/vercel-labs/skills/src/skills.ts + frontmatter.ts]

```yaml
# REQUIRED by skills CLI (must be strings):
name: add-service
description: "..."

# OPTIONAL — consumed by agent runtimes, NOT by skills CLI:
argument-hint: "<service-name>"      # Copilot CLI: shows hint in UI
allowed-tools: Read, Write, Bash, AskUserQuestion  # Copilot CLI only
license: MIT                          # informational
metadata:
  internal: true                      # if true, hides from default install
  author: ...
  version: "..."
```

**Current `skills/add-service/SKILL.md` frontmatter is valid.** Both `name` and `description` are present as strings. Description is correctly quoted (YAML-safe for colons). [VERIFIED: direct file inspection]

### `AskUserQuestion` Tool Name
[ASSUMED — not verified in runtime docs in this session]

`AskUserQuestion` is documented as a GitHub Copilot CLI tool name. The skills.sh CLI does not validate or use `allowed-tools` — it only passes through the raw SKILL.md content to the agent. Claude Code does not have an `AskUserQuestion` tool; it asks questions inline in the conversation. The `allowed-tools` field is silently ignored by agents that don't recognize a tool name. **This is not a blocking issue for skills.sh distribution.** However, it is a runtime behavioral difference: in Claude Code, the skill will ask questions conversationally (which is correct behavior), while in Copilot CLI, `AskUserQuestion` is used as a structured turn.

### Recommended README Structure (D-05)
[VERIFIED: github.com/vercel-labs/agent-skills README + locked decisions from CONTEXT.md]

Based on how vercel-labs/agent-skills README is organized, and locked decision D-05:

```markdown
# dev-tools

> One-line value prop (e.g., "AI skill for adding Docker dev services to any project.")

## Skills

### add-service
What it does, trigger phrases, supported services list

## Installation

\`\`\`bash
npx skills add Cyboooooorg/dev-tools
\`\`\`

## Usage

Trigger with natural language: "add Redis to this project"
Or by slash command: `/add-service redis`

## What Gets Written

Files written to `.devtools/` ...
.devtools/.env is gitignored ...

## Services

Table of all 5 services with descriptions
```

### Anti-Patterns to Avoid
- **Adding a root `skills.json` or `package.json`** — not required and adds confusion
- **Manually symlinking into `.github/skills/`** — skills.sh handles this at install time; doing it manually in this repo would create stale dangling paths
- **Listing monitoring as a user-facing service in README** — monitoring is in the service registry but is not one of the 5 primary services (SKILL-03 lists Redis, RabbitMQ, PostgreSQL, MySQL/MariaDB, MongoDB)

---

## Don't Hand-Roll

| Problem | Don't Build | Use Instead | Why |
|---------|-------------|-------------|-----|
| Multi-agent skill installation | Custom symlinking script | `npx skills add Cyboooooorg/dev-tools` | Covers 40+ agents, handles symlink vs copy, scoped to project or global |
| YAML frontmatter parsing | Custom regex parser | Already done correctly in SKILL.md | skills CLI uses the `yaml` npm package |

---

## Common Pitfalls

### Pitfall 1: Assuming `allowed-tools` needs agent-specific values
**What goes wrong:** Adding Claude-specific tool names vs Copilot-specific tool names creates divergent SKILL.md files
**Why it happens:** Different agents have different tool name conventions
**How to avoid:** The `allowed-tools` field is Copilot CLI convention. skills.sh passes the raw SKILL.md through unchanged. Claude Code will ignore tools it doesn't recognize. Keep a single SKILL.md.
**Warning signs:** If Claude Code fails to invoke the skill at all — but this is a Phase 2 runtime concern, not a Phase 3 distribution concern.

### Pitfall 2: Missing quotes on description with colons
**What goes wrong:** YAML parser treats unquoted `description: Add service: Redis` as a nested mapping
**Why it happens:** YAML 1.2 special-cases colon-space as a key-value delimiter
**How to avoid:** Already fixed (commit 597bd26). Description is quoted. ✅
**Warning signs:** `null` or `{}` description when parsed by skills CLI → skill skipped silently.

### Pitfall 3: README describes wrong install paths
**What goes wrong:** README says "install to `.github/skills/`" — old plan that was superseded
**Why it happens:** Original REQUIREMENTS.md said DIST-01 = symlink to both runtime paths
**How to avoid:** README must use `npx skills add Cyboooooorg/dev-tools` exclusively. No mention of manual symlinking in this repo.

### Pitfall 4: README lists `monitoring` as a supported service
**What goes wrong:** The compose-templates directory contains a `monitoring/` folder that is NOT in the 5 services users can install via the skill
**Why it happens:** monitoring was added as an internal template but SKILL-03 defines 5 services only
**How to avoid:** README service list = Redis, RabbitMQ, PostgreSQL, MySQL/MariaDB, MongoDB. Do not list monitoring.

---

## Code Examples

### Valid SKILL.md Frontmatter Pattern
```yaml
# Source: github.com/vercel-labs/skills/src/skills.ts (parseSkillMd)
# Only name + description are required; both must be strings
---
name: add-service
description: "Add a Docker dev service to this project. Supported services: Redis, RabbitMQ, PostgreSQL, MySQL/MariaDB, MongoDB. Writes Docker Compose and Taskfile configs to .devtools/."
argument-hint: "<service-name>"
allowed-tools: Read, Write, Bash, AskUserQuestion
---
```

### skills CLI Discovery (how it finds this repo's skill)
```bash
# User runs this once in their project:
npx skills add Cyboooooorg/dev-tools

# CLI:
# 1. Clones/fetches Cyboooooorg/dev-tools from GitHub
# 2. Walks directory tree, finds: skills/add-service/SKILL.md
# 3. Parses frontmatter: name="add-service", description="..."
# 4. Installs to each detected agent's skillsDir (e.g., .agents/skills/add-service/)
# 5. Creates symlink (default) or copy
```

### README install section pattern (from vercel-labs/agent-skills)
```markdown
## Installation

```bash
npx skills add Cyboooooorg/dev-tools
```

This installs the skill to all detected AI agents in your project
(GitHub Copilot, Claude Code, Cursor, and [more](https://agentskills.io)).
```

---

## Plan Structure Recommendation

**Research question 5: Should 03-01 be a separate plan?**

**Recommendation: Yes, keep separate.** Rationale:
- 03-01 maps to DIST-01 (discoverability), 03-02 maps to DIST-02 (documentation)
- 03-01 is verification-only — it produces no file changes (or at most trivially fixes a frontmatter field)
- Keeping them separate maintains traceability and allows 03-01 to serve as a gate before committing README
- If 03-01 reveals a frontmatter problem, it can be fixed in isolation before the README is written

**03-01 structure (minimal):**
- Wave 0: No setup needed
- Wave 1: Single task — verify `skills/add-service/SKILL.md` frontmatter parses correctly, confirm `name` and `description` are strings, confirm `skills/add-service/` path exists. Optionally run `npx skills add --list .` to confirm CLI discovery.

**03-02 structure (full README):**
- Wave 0: No setup needed
- Wave 1: Rewrite `README.md` per D-05 content requirements

---

## Assumptions Log

| # | Claim | Section | Risk if Wrong |
|---|-------|---------|---------------|
| A1 | `AskUserQuestion` is silently ignored by Claude Code at runtime (not a block) | Architecture Patterns — AskUserQuestion | If Claude Code errors on unknown tool name, runtime behavior breaks. Low impact on Phase 3 (distribution phase), would affect Phase 2 runtime testing. |
| A2 | `npx skills add Cyboooooorg/dev-tools` works with a public GitHub repo at `github.com/Cyboooooorg/dev-tools` | Architecture Patterns | If repo is private or CLI can't resolve the org name, install fails. Users would need `--token` or a different source format. |

---

## Open Questions

1. **`monitoring` service in templates**
   - What we know: `compose-templates/monitoring/` and `taskfile-templates/monitoring/` exist; the SKILL.md service registry includes monitoring; SKILL-03 only lists 5 services
   - What's unclear: Is `monitoring` intentionally excluded from the user-facing README? Or should it be listed as "optional companion service"?
   - Recommendation: Omit from README services list per SKILL-03 (5 services). If in doubt, check with user.

2. **`npx skills add` vs `npx skills@latest add`**
   - What we know: CLI is v1.5.0 on npm; `npx skills add` uses whatever is cached
   - What's unclear: Should README recommend `npx skills@latest add` to ensure latest version?
   - Recommendation: Use `npx skills add` (without version pin) — consistent with how vercel-labs/agent-skills documents it.

---

## Environment Availability

Step 2.6: SKIPPED — Phase 3 is purely file writes (README.md) and YAML verification. No external services, runtimes, or CLIs beyond `git` are required for execution. `npx` is available system-wide for the optional CLI verification step.

---

## Validation Architecture

### Test Framework
| Property | Value |
|----------|-------|
| Framework | None — no test suite in this project |
| Config file | none |
| Quick run command | `npx skills add --list .` (manual CLI verification) |
| Full suite command | N/A |

### Phase Requirements → Test Map
| Req ID | Behavior | Test Type | Automated Command | File Exists? |
|--------|----------|-----------|-------------------|-------------|
| SKILL-01 | Description triggers on service-addition intents | manual | — | N/A |
| SKILL-02 | Skill installs to both Copilot + Claude paths via skills.sh | manual | `npx skills add --list .` | N/A |
| SKILL-03 | Frontmatter enumerates all 5 services | manual-inspect | `head -6 skills/add-service/SKILL.md` | ✅ |
| DIST-01 | Repo discoverable by skills CLI | manual | `npx skills add --list .` | N/A |
| DIST-02 | README covers install + services | manual-inspect | — | ❌ Wave 0 (write README) |

### Wave 0 Gaps
- [ ] `README.md` — full rewrite needed (03-02)
- No test framework gaps — this project uses manual verification

---

## Security Domain

This phase writes only a `README.md` (public documentation) and verifies a YAML frontmatter field. No authentication, session management, input validation, cryptography, or user data handling is involved.

**ASVS categories applicable:** None for Phase 3 work specifically. The SKILL.md itself executes in an AI agent context where the agent runtime handles its own security model.

---

## Sources

### Primary (HIGH confidence)
- `github.com/vercel-labs/skills` (src/skills.ts, src/agents.ts, src/frontmatter.ts, src/types.ts) — discovery logic, frontmatter requirements, agent install paths [VERIFIED: direct source inspection]
- `npm view skills version` → `1.5.0` [VERIFIED: npm registry]
- `skills/add-service/SKILL.md` — current frontmatter content [VERIFIED: direct file read]
- `.planning/phases/03-cross-runtime-wiring-distribution/03-CONTEXT.md` — locked decisions D-01 through D-07

### Secondary (MEDIUM confidence)
- `github.com/vercel-labs/agent-skills` README — README format patterns for skills.sh repos [VERIFIED: direct fetch]
- `github.com/vercel-labs/skills` README — install command syntax and options [VERIFIED: direct fetch]

### Tertiary (LOW confidence)
- `AskUserQuestion` behavior in Claude Code — training knowledge, not verified against Claude Code docs in this session [ASSUMED]

---

## Metadata

**Confidence breakdown:**
- skills.sh discovery requirements: HIGH — read source directly
- SKILL.md frontmatter validity: HIGH — read source + file directly
- Agent install paths: HIGH — read agents.ts directly
- `AskUserQuestion` cross-agent behavior: LOW — ASSUMED
- README format patterns: HIGH — verified against live examples

**Research date:** 2026-04-10
**Valid until:** 2026-05-10 (skills.sh is fast-moving; check for agent list changes if > 30 days)
