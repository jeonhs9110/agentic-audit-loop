# Angle: Env vars + secrets + startup safety

Paste this as the `{{ANGLE_SPECIFIC_CHECKLIST}}` in the audit-agent prompt.

Target: env vars that will silently misconfigure, secrets that could leak, startup validation gaps, handoff-critical documentation.

## Look for

1. **`NEXT_PUBLIC_` (or `VITE_`, `REACT_APP_`) prefixed secrets.** Any variable that's actually sensitive but was declared with the public prefix and thus embedded in the client bundle. Anon keys (Supabase, Firebase) are fine. Service-role keys, secret keys, API keys with write scope are not.

2. **`process.env.FOO || 'default'` where the default is silently wrong for prod.** Especially: `DATABASE_URL || ''`, `API_KEY || 'test'`, `SITE_URL || 'http://localhost:3000'`. Missing env should fail loudly, not fall back to a broken default.

3. **Hardcoded URLs that should be env-driven.** Prod hostname scattered across the codebase. Fine in metadata (canonical URLs, OG tags) — not fine in server-to-server API calls.

4. **Missing startup validation.** Is there a central check that all required env vars are present? Or does the app boot and only surface missing vars when a specific route fires?

5. **Secrets accidentally logged.** Grep `console.log`, `console.error`, `console.warn` for lines that concatenate an env var or an API key.

6. **`.env` / `.env.production` / `.env.local` committed to the repo.** `git ls-files | grep -i env` — any file that looks like it holds a real secret should be in `.gitignore`.

7. **AWS credentials pattern.** Anything explicitly reading `AWS_ACCESS_KEY_ID` / `AWS_SECRET_ACCESS_KEY`? EC2 role should be used instead — hardcoded credentials are a handoff nightmare.

8. **Missing env var → silent 500.** Routes that read env vars, get undefined, and return a generic error instead of a clear "env not configured" message.

9. **Config in framework config file.** Any secrets in `next.config.ts` / `vite.config.ts` / etc.? Any dev-oriented flags left on for prod?

10. **`process.env.NODE_ENV` misuse.** Any place that reads NODE_ENV and does something risky (enabling debug endpoints, exposing internal state)? Also: any place that gates safety on `NODE_ENV !== 'production'` (which is `true` when NODE_ENV is unset)?

11. **Cookie configuration inconsistency.** sameSite / secure / httpOnly / domain — consistent? Any cookies set as `httpOnly: false` when they carry auth state? Any path-scoped cookies where the browser will store two independent copies at different paths?

12. **CORS / origin allow-list.** Endpoints with `Access-Control-Allow-Origin: *`. Missing origin validation on env-driven allow-lists.

13. **URL parsing without try/catch.** `new URL(process.env.SITE_URL)` throws on undefined.

14. **API key format validation.** Do routes validate the shape of API keys at module load? A stray whitespace or placeholder (e.g. `sk-...` unchanged from `.env.example`) should be rejected explicitly.

15. **Handoff-critical env vars documentation.** Is there a `docs/ENV.md` / `.env.example` / README section that lists every required env var? Do they match `grep 'process\.env\.'` across the codebase? A stale `.env.example` is a handoff booby-trap.

16. **Dev-only routes.** `if (process.env.NODE_ENV === 'development')`-gated endpoints — are any still shipping to prod (would fire on a misconfigured deploy)?

17. **Env var evaluated once at module load.** Any `const USE_X = process.env.X === 'true'` that will silently fall through on whitespace / case variants (`' true '`, `'TRUE'`, `'true\n'`)? Add trim + toLowerCase.

18. **`.env.example` completeness.** Does the example file list every variable the codebase actually reads? A missing var in the example means the handoff engineer will hit a runtime "why is this feature broken" mystery.

## Files to prioritize

- Framework config (`next.config.ts`, `vite.config.ts`, `astro.config.mjs`, etc.)
- `.env` / `.env.example` / `.env.production` (if present)
- Central config helpers (`src/lib/config/`, `src/lib/env/`)
- Auth helpers (they usually read pool ID / client ID env vars)
- Middleware (`src/middleware.ts` or equivalent)
- Grep `process.env\.` across the codebase for a full inventory

## Grep starters

```
process\.env\.                     # inventory all env reads
NEXT_PUBLIC_                       # inventory public-prefixed vars
console\.(log|warn|error).*process\.env    # potential secret logs
AWS_ACCESS_KEY_ID|AWS_SECRET       # hardcoded AWS creds
```

## What NOT to report

- "Env var X is not used" — dead-var cleanup, not a bug
- "Should use dotenv-safe" — tool recommendation, not a specific defect
- "Consider secret rotation" — infra policy, not code
