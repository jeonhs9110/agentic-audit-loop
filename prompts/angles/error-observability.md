# Angle: Error handling + observability

Paste this as the `{{ANGLE_SPECIFIC_CHECKLIST}}` in the audit-agent prompt.

Target: silent-swallow catch blocks, missing error boundaries, unstructured logs, alarm-worthy events with no logs, incident-response ability.

## Look for

1. **`catch {}` blocks that swallow errors silently.** No `console.error`, no re-throw, no fallback behavior. The error disappears; the caller thinks everything succeeded.
2. **`catch (err) { /* ignore */ }`** with a comment that says "ignore".** Same pattern. Rarely justifiable.
3. **`try { await x() } catch { return null }` on hot paths.** Returning null on failure means the downstream code renders "empty state" instead of "error state" — the operator sees "0 items" during outages.
4. **Alerts to Sentry / Datadog / CloudWatch missing on critical paths.** Auth failures, payment failures, data export failures — should emit structured logs that alarm on frequency spikes.
5. **Missing error boundaries in React.** A rendering error in one section takes down the whole page instead of showing "This section failed to load" with a retry.
6. **`console.log` used for structured events.** Should be `console.log(JSON.stringify({event: 'thing.happened', ...}))` for CloudWatch metric filters or equivalent.
7. **Error messages leaking internals.** `Error: relation "users" does not exist` returned to the client. Leaks schema. Log the full error server-side; return a generic message to the client.
8. **No request ID / correlation ID.** Debugging a production issue without a request ID means grepping timestamps and guessing.
9. **Time zone confusion in logs.** `new Date().toISOString()` for UTC + separate local-time label. Not just `new Date()` (which serializes as local).
10. **Missing user context in logs.** "User X failed to Y" — but which user? Log the actor ID (hashed for PII compliance) so audits can reconstruct.
11. **`Error` used as control flow.** `if (thing) throw new Error(...); ... catch { }` — control flow via exceptions is expensive and hides intent.
12. **Unhandled promise rejections.** `void doThing()` fire-and-forget without a `.catch`. Node crashes on unhandled rejections in some configurations.
13. **`Promise.all` where one rejection loses N-1 successes.** Use `Promise.allSettled` and inspect per-promise.
14. **Rate limiter denials logged, but not the actor.** Log `ip_prefix` (truncated for privacy) so abuse patterns are attributable.
15. **Audit log missing on privileged actions.** Role change, user delete, data export, config change — every one should emit an audit event with actor + target + outcome + timestamp.
16. **Audit log includes sensitive values.** Full email addresses, phone numbers, PII in the log body. Hash them.
17. **CSP violation report endpoint missing or broken.** If CSP is deployed, `/api/csp-report` should exist and log violations.
18. **No structured event names.** Ad-hoc log strings ("processing user X"). Prefer `event: 'user.processing.started'` schema — grep-able + alarmable.
19. **Health check that lies.** `/api/health` returns 200 without actually pinging the DB. ALB sees "healthy" during a DB outage; traffic keeps flowing.
20. **Health check with no timeout.** `/api/health` blocks on a slow query — ALB times out; instance deregisters even though it's mostly healthy.

## Files to prioritize

- Global error boundary (React `error.tsx` or equivalent)
- API route handlers — grep for `try {` and `catch`
- Audit log helper (grep for `auditLog` or `audit_log`)
- Health check route (grep for `/health` or `/healthz`)
- Rate-limit helper (log format)
- Analytics / tracking endpoints

## Grep starters

```
catch \{\s*\}                       # empty catch
catch \(.*\) \{\s*\}                # empty catch with arg
catch \{ .* ignore                  # explicitly-ignored catches
console\.log                        # inventory unstructured logs
JSON\.stringify.*event              # inventory structured logs
throw new Error                     # exception-as-control-flow
```

## What NOT to report

- "Consider Sentry" — vendor recommendation
- "Improve logging" — vague; must name a specific event that isn't logged and should be
