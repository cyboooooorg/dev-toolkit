# Milestones

## ✅ v1.0 MVP — SHIPPED 2026-04-21

**Status:** Complete
**Phases:** 7 (phases 1–7)
**Plans:** 14
**Timeline:** 2026-04-09 → 2026-04-21 (12 days)
**Commits:** 98
**Files changed:** 92 files, 17,121 insertions

### Delivered

Interactive `add-service` AI skill for Docker service configuration — 5 supported services (Redis, RabbitMQ, PostgreSQL, MySQL, MongoDB), full merge detection, multi-instance alias support, per-service subfolder output layout, and idempotent re-runs.

### Key Accomplishments

1. Docker Compose + Taskfile templates for 5 services + Monitoring — all credentials via `.env`, zero hardcoded secrets
2. Interactive `SKILL.md` (805 lines, 13 steps) — port/version/credential Q&A, confirmation gate, writes to `.devtools/<service>/`
3. Merge detection and multi-instance alias support — safe re-run, never overwrites existing services
4. Per-service subfolder layout (`.devtools/<slug>/`) with isolated `.env` per instance
5. Full alias prompt UX — slug-preview, normalization, conflict re-prompt loop, optional proactive prompt
6. Fixed 3 integration breaks post-refactor + REDIS_UI_PORT gap (Phase 6 retrospective fix)
7. MERGE-04 idempotency: alias slug re-run exits cleanly with "already installed — nothing to do"

### Tech Debt

- **BF-01** (High): Grafana password ignored at runtime — `env_file: .env` missing from monitoring.compose.yml grafana service
- **DOC-01** (Medium): README "What Gets Written" shows stale pre-Phase-4 flat layout
- **WR-01/WR-02** (Medium): Taskfile `up`/`up-ui` COMPOSE_PROFILES not loaded from `.devtools/.env`
- DOC-02, DOC-03, WR-03 (Low): Cosmetic — stale path comments, metadata port mismatch, description inconsistency

### Archive

- [Roadmap archive](.planning/milestones/v1.0-ROADMAP.md)
- [Requirements archive](.planning/milestones/v1.0-REQUIREMENTS.md)
- [Milestone audit](.planning/milestones/v1.0-MILESTONE-AUDIT.md)

---

*See [ROADMAP.md](.planning/ROADMAP.md) for current milestone phases.*
