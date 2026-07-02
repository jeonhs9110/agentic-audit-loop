# Agentic Audit Loop

**A closed-loop safety net for production web apps.** Run parallel LLM audits against your codebase, apply the safe subset, ship a PR, repeat. Designed to catch real bugs (auth holes, SEO regressions, PII leaks, perf regressions, DB query bombs) before your customers do.

Works with any LLM CLI: Claude Code, Codex CLI, Cursor's composer, plain API. Written prompts + templates — no framework lock-in.

---

## Why this exists

Modern web apps ship faster than any single reviewer can audit. Static analysis catches syntax; humans catch architecture; nothing catches the middle layer where real bugs live:

- Invalid `hreflang="kr"` (should be `ko-KR`) silently killing 40% of your Google alternates cluster
- An admin CSRF hole on `/api/admin/products` that survived because every other write route had it
- A blocking `<link rel="stylesheet">` in `<head>` costing 400ms of LCP on 4G
- A missing `LIMIT` on an admin-facing list route that will eventually exhaust the pool at scale

An LLM audit agent with a targeted prompt finds these in minutes. But a one-shot audit misses too much and applies too many risky fixes. The **loop** — bounded, iterated, with a safe-subset synthesis step and a human-gated merge — is what turns raw LLM findings into a reliable safety net.

## The pattern

```
  ┌──────────────────────────────────────────────────────────┐
  │  Goal (set once by human)                                │
  └────────────────────┬─────────────────────────────────────┘
                       ▼
       ┌──────────── Round N ─────────────┐
       │                                   │
       │   ┌──────── Discovery ────────┐   │
       │   │ 2 parallel audit agents   │   │
       │   │ on fresh non-overlapping  │   │
       │   │ angles                    │   │
       │   └───────────┬───────────────┘   │
       │               ▼                   │
       │   ┌──────── Planning ─────────┐   │
       │   │ Safe-subset synthesis     │   │
       │   │ (defer: needs migration,  │   │
       │   │ infra, big refactor)      │   │
       │   └───────────┬───────────────┘   │
       │               ▼                   │
       │   ┌──────── Execution ────────┐   │
       │   │ Apply fixes in parallel   │   │
       │   └───────────┬───────────────┘   │
       │               ▼                   │
       │   ┌────── Verification ───────┐   │
       │   │ typecheck + lint + tests  │   │
       │   │ + PR body test plan       │   │
       │   └───────────┬───────────────┘   │
       │               ▼                   │
       │   ┌──────── Iteration ────────┐   │
       │   │ Skipped items → next PR   │   │
       │   │ Trajectory table updated  │   │
       │   └───────────┬───────────────┘   │
       └───────────────┼───────────────────┘
                       │
                       ▼
              Human-gated merge
                       │
                       ▼
              Repeat with new angles
```

Five stages per round: **Discovery → Planning → Execution → Verification → Iteration.** Memory (the trajectory table + skipped-items registry) lives in the PR body so it survives across rounds and reviewers.

See [`docs/the-loop.md`](docs/the-loop.md) for the full explanation.

## Closed vs open loop — pick closed

**Open loop:** "Go find bugs, apply fixes, keep going." Great if you have unlimited budget. Not great if you'd like to still have a monthly LLM bill under four figures.

**Closed loop (this repo):** Each round has a bounded goal (2 audit angles, safe-subset apply, one PR). The human gates the merge before the next round starts. Cost is predictable; blast radius per round is small.

See [`docs/open-vs-closed.md`](docs/open-vs-closed.md).

## Quick start (Claude Code)

```bash
# 1. Clone this repo alongside your target project.
git clone https://github.com/<you>/agentic-audit-loop.git

# 2. Open your target project in Claude Code.
cd /path/to/your/webapp
claude

# 3. In the Claude Code session, paste the audit-agent prompt template
#    from prompts/audit-agent.md, filled in with:
#      - the angle you're auditing this round (e.g. "SEO / meta / hreflang")
#      - the working directory
#      - any codebase-specific context (framework, DB, auth provider)
#
#    Optionally spawn a SECOND parallel audit on a different angle for
#    coverage. Two non-overlapping angles per round is the sweet spot.

# 4. Once both audits return findings, synthesize the safe subset
#    (see docs/safe-subset.md), apply fixes, run typecheck + lint,
#    commit with the template in templates/commit-message.md,
#    push, open a PR with the body from templates/pr-body.md.

# 5. Merge, sync master, pick two new angles, repeat.
```

Works the same with Codex CLI or Cursor's composer — the prompts are model-agnostic.

## Ready-made audit angles

Drop-in prompt templates for common audit domains. Each one names the specific defect classes to look for, the files to prioritize, the output format, and the severity ladder.

| Angle | Prompt file | Typical findings |
|---|---|---|
| Core Web Vitals / mobile perf | [`prompts/angles/perf-cwv.md`](prompts/angles/perf-cwv.md) | Blocking CSS, unbounded images, LCP twins, hydration cost |
| DB query hygiene | [`prompts/angles/db-query-hygiene.md`](prompts/angles/db-query-hygiene.md) | Unbounded SELECT, N+1, missing indexes, `SELECT *` on hot paths |
| Admin route CSRF + hardening | [`prompts/angles/admin-csrf.md`](prompts/angles/admin-csrf.md) | Missing `assertSameOrigin`, silent-failure handlers, destructive `confirm()` |
| Env vars + secrets + startup safety | [`prompts/angles/env-config-safety.md`](prompts/angles/env-config-safety.md) | Silent fallbacks, `NEXT_PUBLIC_` leaks, missing startup validation |
| SEO / meta / i18n | [`prompts/angles/seo-meta-i18n.md`](prompts/angles/seo-meta-i18n.md) | hreflang cluster bugs, double-brand titles, missing OG dims, sitemap alternates |
| PII / privacy / audit trail | [`prompts/angles/pii-privacy.md`](prompts/angles/pii-privacy.md) | Unvalidated PATCH inputs, missing audit logs, `SELECT *` leaks |
| Community / user-content integrity | [`prompts/angles/user-content.md`](prompts/angles/user-content.md) | Reserved-name impersonation, rate-limit gaps, XSS sanitizer holes |

Pick two per round. Don't repeat the same angle in consecutive rounds — you'll find diminishing returns.

## What you get

- [`prompts/audit-agent.md`](prompts/audit-agent.md) — the audit-agent prompt template (fill in the angle, output shape, severity ladder)
- [`prompts/angles/`](prompts/angles/) — 7 pre-written angles ready to use
- [`templates/pr-body.md`](templates/pr-body.md) — the PR body with trajectory table + skipped-items registry
- [`templates/commit-message.md`](templates/commit-message.md) — structured commit message shape
- [`docs/the-loop.md`](docs/the-loop.md) — the 5-stage loop explained
- [`docs/open-vs-closed.md`](docs/open-vs-closed.md) — cost management
- [`docs/safe-subset.md`](docs/safe-subset.md) — the synthesis rubric (what to apply, what to defer)
- [`docs/case-study.md`](docs/case-study.md) — a walk-through of one representative round

## When this pattern doesn't fit

Be honest about limits:

- **Not a replacement for real security review.** For genuinely sensitive systems (payments, PHI, financial trading) pair this with a dedicated pentest.
- **Not a replacement for real code review.** Human PR review still catches design decisions the LLM misses.
- **Diminishing returns after ~10 rounds** on a stable codebase. Rotate to novel angles or stop the loop when the CRITICAL findings dry up.
- **Cost scales with agent depth.** Deep audits (50-90 tool calls each) can burn six-figure tokens per round if you're not careful. Read [`docs/open-vs-closed.md`](docs/open-vs-closed.md) before running unattended.

## Contributing

New audit angles are the most valuable contribution. If you run this pattern against your own codebase and find that one angle turns up findings the ready-made prompts miss, PR the prompt back here.

## License

MIT. See [LICENSE](LICENSE).

## Credits

The loop pattern is being popularized by Boris Cherny (creator of Claude Code) and Peter Steinberger (creator of OpenClaw). This repo codifies one specific application of it — bounded, closed-loop production audits — with prompts and templates that a real production run generated across 10+ rounds of live use.
