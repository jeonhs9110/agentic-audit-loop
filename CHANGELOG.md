# Changelog

## v0.2 — full-stack sophistication pass

### Added

- **8 new audit angles**, bringing the total to **15**:
  - `accessibility.md` — WCAG 2.1 AA compliance
  - `mobile-responsive.md` — mobile UX + in-app WebView quirks
  - `api-design.md` — REST/GraphQL design + versioning
  - `auth-flow.md` — sign-up / sign-in / refresh / reset / session
  - `error-observability.md` — silent-swallow catches, structured logs, health checks
  - `test-coverage.md` — critical-path tests, not total %
  - `dependencies-supply-chain.md` — outdated deps, license conflicts, supply-chain risk
  - `devops-cicd.md` — CI/CD, deploy safety, health checks, secrets
- **Orchestrator agent prompt template** at `prompts/orchestrator-agent.md` — turns a manual closed loop into a semi-automated one, with budget guardrails
- **`docs/orchestrator.md`** — when to use orchestration, when to stop it, angle rotation strategy, cost profile
- **`docs/memory.md`** — the trajectory + Skipped registry spec, plus the optional `.audit-loop/` directory pattern for richer state
- **`docs/scenarios.md`** — 5 real-world scenarios (B2C storefront, B2B SaaS admin, multi-tenant platform, internal tools, pre-handoff codebase) with angle pairings, cadences, done conditions, and cost estimates
- **`ARCHITECTURE.md`** — system overview + component responsibilities + data flow diagram + extension points + non-goals
- **`CONTRIBUTING.md`** — how to add new angles, what makes a good angle, severity ladder, PR process
- **`templates/weekly-schedule.md`** — cron cadence options + GitHub Actions example + budget monitoring
- **README rewrite** — badges, TOC, full-stack angle table grouped by frontend/backend/cross-cutting, recommended angle pairings, updated repository layout, credits to Boris Cherny + Peter Steinberger + Ben Stevenson

### Changed

- README leads with loop-engineering vocabulary from the source video (Boris Cherny, Peter Steinberger)
- Angle table reorganized by system layer (frontend, backend, cross-cutting) instead of a flat list
- Case study cross-references updated to link to `docs/scenarios.md` for the pre-handoff pattern

### Repository size

- v0.1: 17 files, ~2,500 lines
- v0.2: 27 files, ~5,500 lines

## v0.1 — initial public release

- README with quick start (Claude Code / Codex / Cursor)
- 7 ready-made audit angles (perf, DB hygiene, admin CSRF, env/config, SEO/i18n, PII/privacy, user-content integrity)
- Audit-agent prompt template
- PR body + commit message templates with trajectory table pattern
- Docs: the 5-stage loop, closed vs open loop cost management, safe-subset synthesis rubric, representative-round case study
- MIT license
