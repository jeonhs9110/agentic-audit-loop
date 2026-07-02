# Real-world scenarios

Five real production shapes and how to configure the loop for each. Adapted from Ben Stevenson's "quiz" format in the source video — each scenario names the goal, the angle pairing, the cadence, and what "done" looks like.

---

## Scenario 1: The B2C storefront

**Setup.** Direct-to-consumer e-commerce. 500 SKUs. Peak traffic during promotions. Mobile-first audience. Success metric: conversion rate + LCP.

**Loop goal.** Catch conversion-blockers before they hit production.

**Angle pairing.** `perf-cwv` + `seo-meta-i18n`

- Perf-CWV catches LCP regressions from hero images, blocking CSS, unbounded product-detail images.
- SEO-meta-i18n catches hreflang cluster bugs, OG image regressions, sitemap drift — every one of which hurts organic acquisition.

**Cadence.**
- Baseline: weekly (every Monday morning).
- Peak season (Black Friday, Chuseok, launch week): accelerate to daily during the two weeks before the traffic peak.
- Off-peak: monthly is fine.

**Done conditions.**
- LCP p75 < 2.5s on mobile 4G for 4 consecutive rounds
- Google Search Console shows zero hreflang errors for 2 consecutive rounds
- No CRITICAL findings in perf-cwv OR seo-meta-i18n for 2 consecutive rounds

**Estimated cost.** $15-30/week during baseline, $50-100/week during peak.

---

## Scenario 2: The B2B SaaS admin surface

**Setup.** Multi-user SaaS with an admin surface used by internal operators. Small team, high leverage per user. Success metric: incident count on the admin surface + operator satisfaction.

**Loop goal.** Prevent operator-caused data loss + reduce operator support tickets.

**Angle pairing.** `admin-csrf` + `error-observability`

- Admin-CSRF catches destructive-action gaps, silent-failure handlers, missing audit logs.
- Error-observability catches silent-swallow catches, missing structured events, alarm-worthy actions with no log trail.

**Cadence.**
- Bi-weekly (every other Monday) — admin surfaces evolve slower than customer surfaces.
- After any admin-facing feature ships, run one round within 48 hours.

**Done conditions.**
- Zero destructive-action gaps for 3 consecutive rounds
- Every write route emits a structured event
- No CRITICAL findings from either angle for 2 consecutive rounds
- Operator support ticket volume drops by ≥ 25%

**Estimated cost.** $10-20/week.

---

## Scenario 3: The multi-tenant platform

**Setup.** Platform where each customer has isolated data. Cross-tenant leaks are existential. Success metric: zero tenant-isolation regressions.

**Loop goal.** Regression-proof tenant isolation on every release candidate.

**Angle pairing.** `auth-flow` + `pii-privacy`

- Auth-flow catches session isolation, refresh cookie scope, cross-tenant token reuse.
- PII-privacy catches over-fetch that includes wrong-tenant fields, missing tenant_id in WHERE clauses.

**Cadence.**
- Every release candidate before it ships.
- Bi-weekly baseline in between.

**Done conditions.**
- Zero cross-tenant findings from either angle for 3 consecutive rounds
- Every DB query in a route handler includes the tenant scope (auditable)

**Estimated cost.** $10-15/week baseline; $10-20 per release candidate.

**Cautions.** Tenant isolation bugs are shipping-blockers. If the loop finds a CRITICAL cross-tenant leak, the release candidate does NOT ship until fixed.

---

## Scenario 4: The internal tools app

**Setup.** Internal-only app used by employees. Low external threat, high internal-workflow value. Success metric: reduce operator time-per-task + reduce IT tickets.

**Loop goal.** Reduce operator support tickets.

**Angle pairing.** `accessibility` + `user-content`

- Accessibility catches keyboard-navigation gaps, focus management, contrast issues.
- User-content catches (in this scenario) internal-note fields where operators paste data — reserved-name gaps, XSS surfaces in internal notes.

**Cadence.**
- Monthly, right after any major UI ships.
- Additional round after any escalation about "the tool is broken for keyboard users" or "someone posted a broken embed in a shared note".

**Done conditions.**
- Zero WCAG AA violations on the operator's most-used flows
- No internal-tool bug tickets tagged "accessibility" or "content-rendering" for 2 quarters

**Estimated cost.** $5-10/round, run monthly.

---

## Scenario 5: The pre-handoff codebase

**Setup.** You're winding down. The codebase will hand off to another team (vendor, in-house, next-generation product). Success metric: the receiving team can operate the code without you.

**Loop goal.** Hand off a clean state — no lurking defects, no dead code, no environment-specific assumptions.

**Angle pairing.** Rotate through **all 15 angles** over 8 rounds. Round 8 is a wrap-up round that revisits any angle that produced high-severity findings previously.

**Cadence.**
- 8 rounds in 2-3 weeks (2-3 rounds/week).
- One-time campaign. Not a permanent loop.

**Done conditions.**
- Zero CRITICAL findings on the last 3 rounds
- Skipped registry is empty OR every remaining item is a documented handoff item
- Every `.env.example` variable matches actual `process.env` reads (audit via `env-config-safety`)
- README + docs match the actual code (audit via a custom "handoff readiness" angle you write for this campaign)

**Estimated cost.** $80-150 total for the 8-round campaign.

**Post-campaign.** Loop stops. Handoff docs shipped. The receiving team gets a clean state + an intact `.audit-loop/skipped.md` + a `learnings.md` that captures every non-obvious codebase decision.

This is the shape the loop this repo codifies was originally run for — 10 rounds against a real production Next.js codebase during a contract wind-down. See [`case-study.md`](case-study.md) for what one representative round looked like.

---

## Scenario matrix

| Scenario | Angles | Cadence | Cost/mo | Done when |
|---|---|---|---|---|
| B2C storefront | perf-cwv + seo-meta-i18n | Weekly | $60-120 | LCP p75 < 2.5s for 4 rounds |
| B2B SaaS admin | admin-csrf + error-observability | Bi-weekly | $20-40 | Zero destructive gaps for 3 rounds |
| Multi-tenant | auth-flow + pii-privacy | Per RC | $50-100 | Zero cross-tenant for 3 rounds |
| Internal tools | accessibility + user-content | Monthly | $5-10 | Zero WCAG on core flows |
| Pre-handoff | Rotate all 15 | 8-round burst | $80-150 total | Skipped registry empty |

---

## Customizing for your scenario

Your scenario probably isn't in this list. Build your own:

1. **Name the failure mode you're trying to prevent.** ("A tenant leak", "A LCP regression", "A silent audit-log gap".)
2. **Pick the two angles whose findings would most reliably catch that failure.** Usually one for the surface where the failure would appear + one for the underlying invariant.
3. **Pick a cadence.** Match to release cadence, traffic peaks, or team availability. Don't run faster than your reviewers can absorb.
4. **Define done.** Concrete, measurable. "No CRITICAL from angle X for N rounds" is a good default.
5. **Budget it.** Cost per round × rounds/month + slack for peak periods. Set an alarm.

Every scenario is a customization of the same primitive: two angles, one cadence, one done condition.
