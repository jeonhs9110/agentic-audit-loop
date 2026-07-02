# Audit Agent Prompt Template

Paste this into your LLM CLI (Claude Code / Codex CLI / Cursor composer) at the start of an audit round. Fill in the four `{{PLACEHOLDERS}}` before sending.

For maximum coverage, spawn TWO instances in parallel with different `{{ANGLE}}` values. Do NOT overlap angles — the whole point of parallelism is broader surface, not redundant coverage.

---

You are auditing the codebase at `{{WORKING_DIR}}` for defects in the **{{ANGLE}}** domain. This is one round of a repeated safety-net review pattern.

Context: {{PROJECT_CONTEXT}}

Look for concrete, file:line defects that a real user would encounter. Not stylistic nits, not "consider refactoring", not TODO-flagged code that the team has already acknowledged. Real bugs that will fire in production if unshipped.

## Specifically look for

{{ANGLE_SPECIFIC_CHECKLIST}}

## How to work

1. Sweep the surface first: grep for the relevant patterns, list candidate files.
2. Read the top ~10 candidate files in full — not just the matched lines.
3. Cross-reference: a defect in one file often implies the same defect in a sibling file. If you see one CSRF gap on `POST /admin/foo`, check every other `/admin/*` POST.
4. Rank by severity (see ladder below).
5. Filter out anything not actionable: no "consider using X", no "might be worth refactoring".

## Output format

For each finding:

```
**N. [SEVERITY] path/to/file.ts:LINE**
- **Issue:** one-sentence defect summary
- **Scenario:** concrete failure — what does a real user (or bot, or crawler, or attacker) see?
- **Fix:** the concrete change to make
```

Sort by severity descending. Cap at 10-15 findings — quality over volume. If you can't find 10 real defects at MEDIUM+ severity, return fewer. Padding the list with LOW-severity polish dilutes the signal.

## Severity ladder

- **CRITICAL** — silent data loss, auth bypass, secret leak, or a defect that will take the service down at scale
- **HIGH** — real customer-visible bug OR silent failure on a hot path OR compliance-adjacent gap that would matter in an audit
- **MEDIUM** — bug on a warm path, silent-swallow error handler, N+1 that won't fire today but will next year
- **LOW** — polish, consistency, stale comments, dead code

Do NOT report:
- Style issues
- "This could be refactored"
- "There is no test for X" (unless the untested path is silently broken today)
- Findings that would require adding a dependency (defer those with a "handoff item" flag)

## At the end

Absolute file paths for every file you looked at (helps the next round skip re-reads).

Return ONLY the findings block. No preamble, no "I found N issues" wrap-up, no next-steps recommendations. Just the ranked findings and the file-paths list.

---

## Example fill-in

```
{{WORKING_DIR}}    = c:/Users/me/my-webapp
{{ANGLE}}          = SEO / meta / hreflang / structured data / social share
{{PROJECT_CONTEXT}} = Next.js 15 App Router / TypeScript 5 / bilingual (English + Japanese) / hosted on Vercel / target audience mobile.
{{ANGLE_SPECIFIC_CHECKLIST}} = (paste the checklist from prompts/angles/seo-meta-i18n.md)
```

See `prompts/angles/*` for the ready-made checklists per angle.
