# Phase 03 Validation Architecture

Phase 03 has no automated test suite — all verifications are manual shell commands against
committed files. Both plans produce deterministic, inspectable outputs.

---

## Plan 03-01: SKILL.md Frontmatter Verification

**What is validated:** `skills/add-service/SKILL.md` frontmatter is valid for skills.sh discovery.

### Checks

```bash
# Check 1: Frontmatter is correct (all 6 lines)
head -6 skills/add-service/SKILL.md
# Expected:
# ---
# name: add-service
# description: "Add a Docker dev service to this project. Supported services: Redis, RabbitMQ, PostgreSQL, MySQL/MariaDB, MongoDB. Writes Docker Compose and Taskfile configs to .devtools/."
# argument-hint: "<service-name>"
# allowed-tools: Read, Write, Bash, AskUserQuestion
# ---

# Check 2: File is at the skills.sh discovery path
ls skills/add-service/SKILL.md
# Expected: skills/add-service/SKILL.md (file exists, no error)
```

**Pass criteria:** Both commands succeed and output matches expected values.

---

## Plan 03-02: README.md Content Verification

**What is validated:** `README.md` satisfies DIST-02 content requirements.

### Checks

```bash
# Check 1: Install command appears exactly once
grep -c "npx skills add Cyboooooorg/dev-tools" README.md
# Expected: 1

# Check 2: All 5 services are mentioned
grep -E "Redis|RabbitMQ|PostgreSQL|MySQL|MongoDB" README.md | wc -l
# Expected: ≥ 5

# Check 3: Excluded content is absent (monitoring service, manual symlinks)
grep -i "monitoring\|\.github/skills\|\.agents/skills" README.md
# Expected: empty output (no matches)
```

**Pass criteria:** Check 1 returns `1`, Check 2 returns `≥ 5`, Check 3 returns no output.

---

## Coverage Summary

| Requirement | Validated By | Check |
|-------------|-------------|-------|
| SKILL-01 (name field) | 03-01 Check 1 | `head -6` → `name: add-service` |
| SKILL-02 (multi-agent via skills.sh) | 03-01 Check 2 | file at `skills/add-service/SKILL.md` |
| SKILL-03 (5 services in description) | 03-01 Check 1 | `head -6` → description enumerates 5 services |
| DIST-01 (skills.sh discoverable) | 03-01 Check 2 | correct discovery path present |
| DIST-02 (README) | 03-02 Checks 1-3 | install command, all 5 services, no excluded content |
