# Contributing

The highest-value contribution is a **new audit angle** that catches a class of defects the existing 15 angles miss. Second-highest is improving an existing angle's checklist with a real-world defect it should have caught but didn't.

Everything else — typo fixes, better examples, clearer edge-case discussion, new scenarios — is welcome too.

---

## What makes a good angle

Every angle in `prompts/angles/*.md` follows the same shape. When adding a new one:

### Structure

1. **One-line target statement.** What class of defects does this angle catch?
2. **10-20 "look for" items.** Each item names a specific, concrete defect pattern — not a vague concern.
3. **Files to prioritize.** Where does the auditor start?
4. **Grep starters.** Regexes or literal strings that surface candidate files quickly.
5. **What NOT to report.** The negative space matters. An angle that doesn't tell the LLM what to skip produces noise.

### The "concrete defect" test

Every "look for" item should pass this test: **can you name a specific failure mode a real user or system would experience?**

Good: *"Sign-up rate limit missing. Bots create thousands of dummy accounts to reserve usernames or spam the referral system. Rate limit at 10/hour per IP."*

Bad: *"Consider improving rate limiting."*

Good: *"`SELECT *` on user tables returning fields the caller doesn't need. Every extra field is a potential PII surface if a future column holds sensitive data."*

Bad: *"Query optimization opportunity."*

If a "look for" item doesn't name a real failure mode, delete it.

### Severity ladder

Every angle uses the same severity ladder. Copy this verbatim into your angle:

- **CRITICAL** — silent data loss, auth bypass, secret leak, or a defect that will take the service down at scale
- **HIGH** — real customer-visible bug OR silent failure on a hot path OR compliance-adjacent gap that would matter in an audit
- **MEDIUM** — bug on a warm path, silent-swallow error handler, N+1 that won't fire today but will next year
- **LOW** — polish, consistency, stale comments, dead code

### Naming

- File: `prompts/angles/<kebab-case-noun>.md`. Match the existing pattern: `perf-cwv.md`, `admin-csrf.md`.
- Angle name (used in README + orchestrator): title-case with slashes if compound. "Core Web Vitals + mobile perf" beats "core_web_vitals".

---

## Improving an existing angle

If you ran the loop against your codebase and one angle missed a defect it should have caught, contribute the defect pattern back.

Format the PR as: "angle X should catch pattern Y" with:

- The new "look for" bullet (concrete, testable)
- A one-paragraph example of the defect you found (redacted for confidentiality)
- Optional: a grep starter that would have surfaced it

---

## Adding a scenario

`docs/scenarios.md` grows over time. A good scenario has:

- **A named audience.** Not "a web app" — "a B2C storefront with 500 SKUs".
- **A concrete failure mode.** Not "improve quality" — "prevent conversion-blocking LCP regressions".
- **An angle pairing.** Two angles, why these two.
- **A cadence.** Weekly, bi-weekly, per-release.
- **Concrete done conditions.** Measurable, ideally in Google Search Console / Grafana / your equivalent.
- **A cost estimate.** Rough $/month at typical Sonnet pricing.

---

## Documentation contributions

**Typo fixes and clarifications** — PR directly, no discussion needed.

**Restructuring an existing doc** — open an issue first to discuss. The docs are stable; changing their shape breaks any external references.

**New docs** — open an issue first. New docs should have a clear "why" that maps to a real question users ask.

---

## What NOT to contribute

- **Framework migrations.** "Convert everything to Yarn." "Move to pnpm." Choice belongs to individual users.
- **Vendor recommendations.** "Use Sentry for observability." Vendor-neutral prompts stay portable.
- **AI-model-specific optimizations.** Prompts should work on any frontier LLM. Model-specific tuning belongs in a separate document.
- **Scenarios based on your specific company.** Anonymize before contributing. If the scenario is too specific to redact, it's a case study for your internal docs, not this repo.

---

## PR process

1. **Small PRs.** One angle per PR. One doc per PR. Big-bang PRs are hard to review.
2. **Include a test drive.** Run the new/updated angle against a real codebase (yours or a public example) and paste 2-3 real findings it produced in the PR description. Proves the prompt works.
3. **Update the README's angle table.** If you added an angle, add a row. The table is the discoverability surface.
4. **Update `CHANGELOG.md`.** Note the addition in the current unreleased section.

---

## Style

**Second person, imperative.** "Look for X" not "the auditor should look for X".

**Concrete, not general.** "Rate limit at 5/hour per IP" not "rate limits should be configured".

**Skip padding.** No introductions, no summaries, no "in conclusion" wrap-ups. Get to the checklist.

**Match existing tone.** Read 2-3 existing angles before writing yours. Consistency > cleverness.

---

## Questions

Open an issue tagged `question`. Response time depends on maintainer availability — this is a side project, not a supported product.
