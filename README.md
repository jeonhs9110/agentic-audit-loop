# Agentic Audit Loop

> A closed-loop safety net for production web apps.
> Two parallel LLM audits per round. Safe-subset synthesis. PR-gated merge. Repeat.
> Full-stack coverage: frontend, backend, DB, auth, SEO, PII, accessibility, DevOps.

[![License: MIT](https://img.shields.io/badge/license-MIT-blue.svg)](LICENSE)
[![Version](https://img.shields.io/badge/version-v0.2-informational)](CHANGELOG.md)
[![PRs Welcome](https://img.shields.io/badge/PRs-welcome-brightgreen.svg)](CONTRIBUTING.md)
[![Works with Claude Code](https://img.shields.io/badge/Claude%20Code-supported-orange)](https://docs.anthropic.com/en/docs/claude-code)
[![Works with Codex](https://img.shields.io/badge/Codex%20CLI-supported-lightgrey)](https://github.com/openai/codex)
[![Works with Cursor](https://img.shields.io/badge/Cursor-supported-purple)](https://cursor.sh)

---

## Contents

1. [Why loop engineering](#why-loop-engineering)
2. [The pattern](#the-pattern)
3. [Quick start (5 minutes)](#quick-start-5-minutes)
4. [The 15 built-in audit angles](#the-15-built-in-audit-angles)
5. [The orchestrator pattern](#the-orchestrator-pattern)
6. [Closed vs open loop — cost management](#closed-vs-open-loop--cost-management)
7. [Real-world scenarios](#real-world-scenarios)
8. [Repository layout](#repository-layout)
9. [When this pattern doesn't fit](#when-this-pattern-doesnt-fit)
10. [Contributing](#contributing)
11. [Credits](#credits)

---

## Why loop engineering

Boris Cherny (creator of Claude Code) and Peter Steinberger (creator of OpenClaw) have both publicly stated that this is now how they program. They don't prompt a chatbot and they don't write code by hand. They build **loops** — systems where LLMs prompt themselves through structured stages, iterate with memory that persists across runs, and hand back to a human only at the gates that matter.

The loop is the primitive. Prompting is the old way.

This repository codifies one specific application of that idea: **bounded, closed-loop production audits.** You point it at a real web app, set two audit angles, run the loop for one round, and get a reviewable PR. Merge, sync, rotate angles, run round two. Repeat until the critical-severity find rate drops to zero.

The pattern catches the middle layer of defects that neither static analysis nor human review reliably catches:

- Invalid `hreflang="kr"` codes silently killing 40% of your Google alternates cluster
- CSRF holes on admin write routes that survived because every other route had the guard
- Blocking `<link rel="stylesheet">` in `<head>` costing 400ms of LCP on 4G
- Missing `LIMIT` on an admin list route that will exhaust the pool at scale
- Reserved-name impersonation on user-generated content (`author_name: "Admin"`)
- `SELECT *` on user tables leaking PII fields no caller reads

A well-prompted LLM audit agent finds these in minutes. A **loop** turns those findings into a reliable, repeatable safety net.

---

## The pattern

Every round has five stages, executed in order, with memory that survives across rounds.

```
  ┌──────────────────────────────────────────────────────────┐
  │  Goal (set once by human)                                │
  └────────────────────┬─────────────────────────────────────┘
                       ▼
       ┌──────────── Round N ─────────────┐
       │                                   │
       │   1. DISCOVERY                    │
       │      Two parallel audit agents    │
       │      on non-overlapping angles    │
       │              │                    │
       │              ▼                    │
       │   2. PLANNING                     │
       │      Safe-subset synthesis        │
       │      (defer migrations/infra)     │
       │              │                    │
       │              ▼                    │
       │   3. EXECUTION                    │
       │      Apply fixes in parallel      │
       │              │                    │
       │              ▼                    │
       │   4. VERIFICATION                 │
       │      typecheck + lint + tests     │
       │              │                    │
       │              ▼                    │
       │   5. ITERATION                    │
       │      Update trajectory + skipped  │
       │      registry (memory)            │
       │                                   │
       └───────────────┬───────────────────┘
                       │
                       ▼
              Human-gated merge
                       │
                       ▼
              Rotate angles, repeat
```

Memory lives in the PR body's trajectory table + skipped-items registry. It travels with the git history, so any reviewer joining at round 8 can see the shape of rounds 1-7 in `git log`.

See [`docs/the-loop.md`](docs/the-loop.md) for the full stage-by-stage explanation.

---

## Quick start (5 minutes)

The fastest path: point Claude Code (or Codex CLI, or Cursor composer) at your target project and paste one audit prompt.

### 1. Clone this repo somewhere

```bash
git clone https://github.com/jeonhs9110/agentic-audit-loop.git
cd agentic-audit-loop
```

### 2. Open your target project in your LLM CLI of choice

```bash
cd /path/to/your/webapp
claude   # or: codex, cursor
```

### 3. Copy the audit-agent prompt template

Open [`prompts/audit-agent.md`](prompts/audit-agent.md) and fill in the four placeholders:

- `{{WORKING_DIR}}` — the absolute path of your target project
- `{{ANGLE}}` — the angle you're auditing this round (e.g. `SEO / meta / hreflang`)
- `{{PROJECT_CONTEXT}}` — one sentence describing framework, DB, auth provider
- `{{ANGLE_SPECIFIC_CHECKLIST}}` — paste the checklist from the corresponding file in `prompts/angles/*`

### 4. Spawn TWO parallel audit sessions

The point of parallelism is broader surface — not redundant coverage. Pair angles that touch different files. See [Ready-made angle pairings](#recommended-angle-pairings).

### 5. Synthesize + apply + verify + PR

Once both audits return their findings:

1. Sort findings into three buckets (see [`docs/safe-subset.md`](docs/safe-subset.md)): **Apply now**, **Defer with reason**, **Reject**.
2. Apply fixes.
3. Run `tsc --noEmit` + `eslint .` (or your project's equivalent) — clean before PR.
4. Commit with the template in [`templates/commit-message.md`](templates/commit-message.md).
5. Open a PR with the body from [`templates/pr-body.md`](templates/pr-body.md) — carry forward the trajectory table + skipped-items registry.

### 6. Merge, sync master, pick two new angles, repeat

That's the loop. See [`docs/case-study.md`](docs/case-study.md) for a full walkthrough of what one representative round looks like.

---

## The 15 built-in audit angles

Full-stack coverage. Pick two per round from different rows of the table. Don't repeat an angle in consecutive rounds — you'll hit diminishing returns fast.

### Frontend + user-facing

| Angle | Prompt file | Catches |
|---|---|---|
| **Core Web Vitals + mobile perf** | [`perf-cwv.md`](prompts/angles/perf-cwv.md) | Blocking CSS, LCP twins, unbounded images, hydration cost, bundle bloat |
| **Accessibility (WCAG)** | [`accessibility.md`](prompts/angles/accessibility.md) | Missing labels, focus traps, contrast ratios, keyboard nav, aria-live gaps |
| **Mobile / responsive** | [`mobile-responsive.md`](prompts/angles/mobile-responsive.md) | Touch targets under 44px, viewport meta, safe-area insets, hover-only interactions |
| **SEO / meta / i18n** | [`seo-meta-i18n.md`](prompts/angles/seo-meta-i18n.md) | hreflang cluster bugs, double-brand titles, missing OG dims, sitemap alternates |
| **User-content integrity** | [`user-content.md`](prompts/angles/user-content.md) | Reserved-name impersonation, XSS sanitizer holes, rate-limit gaps, ownership check bugs |

### Backend + data layer

| Angle | Prompt file | Catches |
|---|---|---|
| **DB query hygiene** | [`db-query-hygiene.md`](prompts/angles/db-query-hygiene.md) | Unbounded SELECT, N+1, missing indexes, `SELECT *` on hot paths, deep OFFSET |
| **API design + versioning** | [`api-design.md`](prompts/angles/api-design.md) | Breaking changes, missing versioning, unclear status codes, unbounded pagination |
| **Auth flow completeness** | [`auth-flow.md`](prompts/angles/auth-flow.md) | Sign-up/sign-in/refresh/reset gaps, session desync, token TTL mismatches |
| **Admin route CSRF + hardening** | [`admin-csrf.md`](prompts/angles/admin-csrf.md) | Missing `assertSameOrigin`, silent-failure handlers, destructive `confirm()`, audit-log gaps |
| **File uploads** *(coming in v0.3)* | — | Content-type validation, size caps, path-traversal, virus scan |

### Cross-cutting concerns

| Angle | Prompt file | Catches |
|---|---|---|
| **Env vars + secrets + startup safety** | [`env-config-safety.md`](prompts/angles/env-config-safety.md) | Silent fallbacks, `NEXT_PUBLIC_` leaks, missing startup validation |
| **PII / privacy / audit trail** | [`pii-privacy.md`](prompts/angles/pii-privacy.md) | Unvalidated PATCH inputs, missing audit logs, `SELECT *` leaks, weak delete flows |
| **Error handling + observability** | [`error-observability.md`](prompts/angles/error-observability.md) | Silent-swallow catch blocks, missing error boundaries, unstructured logs, alarm-worthy events with no logs |
| **Test coverage on critical paths** | [`test-coverage.md`](prompts/angles/test-coverage.md) | Untested auth, checkout, PII delete flows; brittle tests; missing E2E on money paths |
| **Dependencies + supply chain** | [`dependencies-supply-chain.md`](prompts/angles/dependencies-supply-chain.md) | Outdated deps with CVEs, deprecated packages, license conflicts, unused deps |
| **DevOps / CI-CD / deploy safety** | [`devops-cicd.md`](prompts/angles/devops-cicd.md) | Deploy path without rollback, missing health checks, unbounded workflow triggers |

### Recommended angle pairings

Pair angles that touch different files:

- `perf-cwv` + `db-query-hygiene` — client-side perf + server-side perf, no file overlap
- `admin-csrf` + `env-config-safety` — auth surface + config surface
- `seo-meta-i18n` + `pii-privacy` — metadata layer + data layer
- `accessibility` + `error-observability` — user-facing quality + operator-facing quality
- `user-content` + `perf-cwv` — user-content routes + rendering perf
- `api-design` + `auth-flow` — API surface + auth surface
- `test-coverage` + `dependencies-supply-chain` — verification depth + supply-chain hygiene

---

## The orchestrator pattern

For teams running this at scale, the loop can itself be orchestrated by an agent. This is the pattern the source YouTube video by Ben Stevenson describes with the pickleball-store example: one **orchestrator agent** delegates work to **specialist agents**, synthesizes their outputs, updates memory, and either loops again or hands back to the human.

Adapted to auditing, the orchestrator:

1. Reads the last N PR body trajectory tables (memory).
2. Picks two audit angles for this round based on rotation + coverage gaps.
3. Spawns two audit agents in parallel with the corresponding prompt files.
4. Awaits both, synthesizes findings into the safe-subset buckets.
5. Applies fixes, runs verification.
6. Opens a PR with the standard trajectory + skipped-items body.
7. Waits for merge, then rotates and repeats.

The full orchestrator prompt is in [`prompts/orchestrator-agent.md`](prompts/orchestrator-agent.md). The design discussion is in [`docs/orchestrator.md`](docs/orchestrator.md).

**Warning:** an orchestrator agent turns a closed loop back into a semi-open loop. Cost controls are essential — see [`docs/open-vs-closed.md`](docs/open-vs-closed.md) before enabling.

---

## Closed vs open loop — cost management

The single biggest failure mode of loop engineering is unattended token spend.

- **Open loop** — the LLM decides when to stop. Great for research budgets. Reported costs: $1,000–$10,000 per week of unattended runs.
- **Closed loop** (this pattern) — human gates the merge before each round starts. Predictable costs of $5–15 per round at typical Sonnet pricing.

The video describes both explicitly. **Pick closed unless you have a research-lab budget.**

Concrete guardrails, cost math per stage, and stop conditions are in [`docs/open-vs-closed.md`](docs/open-vs-closed.md).

---

## Real-world scenarios

Adapted from the video's quiz format, applied to production web apps. Each scenario names the goal, the angle pairing, and the expected cadence.

| Scenario | Goal | Angle pairing | Cadence |
|---|---|---|---|
| **The B2C storefront** | Catch conversion-blockers before they hit production | `perf-cwv` + `seo-meta-i18n` | Weekly, pre-Black-Friday accelerated to daily |
| **The B2B SaaS admin surface** | Prevent operator-caused data loss | `admin-csrf` + `error-observability` | Bi-weekly on the admin subdirectory |
| **The multi-tenant platform** | Regression-proof tenant isolation | `auth-flow` + `pii-privacy` | Every release candidate |
| **The internal tools app** | Reduce operator support tickets | `accessibility` + `user-content` | Monthly, right after major UI ships |
| **The pre-handoff codebase** | Hand off clean state to next team | Rotate through all 15 angles over 8 rounds | One-time campaign |

Full walkthroughs, prompts, and expected outputs are in [`docs/scenarios.md`](docs/scenarios.md).

---

## Repository layout

```
agentic-audit-loop/
├── README.md                      ← you are here
├── LICENSE                        (MIT)
├── CONTRIBUTING.md                (how to add angles / improve prompts)
├── ARCHITECTURE.md                (system overview + data flow)
├── CHANGELOG.md                   (release notes)
│
├── docs/
│   ├── the-loop.md                (the 5 stages explained)
│   ├── open-vs-closed.md          (cost management + stop conditions)
│   ├── safe-subset.md             (the synthesis rubric)
│   ├── orchestrator.md            (multi-agent orchestration)
│   ├── memory.md                  (memory spec: trajectory + skipped registry)
│   ├── scenarios.md               (5 real-world scenarios with cadence)
│   └── case-study.md              (representative round walkthrough)
│
├── prompts/
│   ├── audit-agent.md             (main audit-agent prompt template)
│   ├── orchestrator-agent.md      (orchestrator prompt template)
│   └── angles/                    (15 ready-made audit angles)
│       ├── perf-cwv.md
│       ├── accessibility.md
│       ├── mobile-responsive.md
│       ├── seo-meta-i18n.md
│       ├── user-content.md
│       ├── db-query-hygiene.md
│       ├── api-design.md
│       ├── auth-flow.md
│       ├── admin-csrf.md
│       ├── env-config-safety.md
│       ├── pii-privacy.md
│       ├── error-observability.md
│       ├── test-coverage.md
│       ├── dependencies-supply-chain.md
│       └── devops-cicd.md
│
└── templates/
    ├── pr-body.md                 (trajectory table + skipped registry)
    ├── commit-message.md          (structured commit shape)
    └── weekly-schedule.md         (automation cadence guide)
```

---

## When this pattern doesn't fit

Be honest about limits.

- **Not a replacement for real security review.** For genuinely sensitive systems (payments, PHI, financial trading) pair this with a dedicated pentest and an SOC-2-adjacent review.
- **Not a replacement for real code review.** Human PR review catches design decisions and business-context regressions the LLM misses.
- **Diminishing returns after ~10 rounds** on a stable codebase. Rotate to novel angles or stop the loop when the CRITICAL find rate drops to zero for two consecutive rounds.
- **Cost scales with agent depth.** Deep audits (50-90 tool calls each) can burn six-figure tokens per round if you're not careful. Read [`docs/open-vs-closed.md`](docs/open-vs-closed.md) before enabling any automation.
- **Not for greenfield.** The pattern needs an existing codebase to audit. On a project less than a week old, code review + type checking give you more per dollar.
- **Not a "AI replaces engineers" pitch.** It's a "AI is a safety net that catches what humans miss between reviews" pitch. Very different framing.

---

## Contributing

New angles are the highest-value contribution. If you run this pattern against your own codebase and one of your rounds turns up findings that the ready-made prompts miss, PR the prompt back here.

See [CONTRIBUTING.md](CONTRIBUTING.md) for:
- How to structure a new angle prompt
- What "concrete defect" means (versus "vague suggestion")
- The severity ladder used across all angles
- How to redact case studies for public sharing

Small improvements welcome too: typo fixes, better example prompts, clearer edge-case discussions in the docs, new scenarios in `docs/scenarios.md`.

---

## Credits

The loop-engineering pattern is being publicly popularized by:

- **Boris Cherny** — creator of Claude Code, at Anthropic
- **Peter Steinberger** — creator of OpenClaw

The specific closed-loop production-audit application in this repo was developed across 10+ live rounds on a real production Next.js codebase. Findings from those rounds were used to construct the 15 audit angle prompts — the case study in [`docs/case-study.md`](docs/case-study.md) walks through one representative round with the codebase-specific details anonymized.

Video reference (via Ben Stevenson): the 5-stage loop diagram, the open-vs-closed framing, the orchestrator + specialist agents pattern, and the "grow a pickleball store with 3 agents" example are all his. This repo is one specific application of the general framework he demonstrates.

---

## License

[MIT](LICENSE). Prompts and templates are free to adopt, modify, and redistribute. If you publish a fork with new angles, a link back is appreciated but not required.
