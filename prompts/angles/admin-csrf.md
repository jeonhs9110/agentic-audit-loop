# Angle: Admin route CSRF + operator UX hardening

Paste this as the `{{ANGLE_SPECIFIC_CHECKLIST}}` in the audit-agent prompt.

Target: admin/staff routes — CSRF gaps, silent-failure handlers, destructive-action confirmations, list refresh patterns, error surfacing.

## Look for

1. **CSRF gaps on admin write routes.** Every `POST`, `PATCH`, `PUT`, `DELETE` on `/api/admin/*` (or equivalent) should check `Origin` or `Referer` matches your site. Grep ALL admin routes and cross-reference against the CSRF helper. Any handler that doesn't call the helper is a hole — a phished admin cookie is enough to delete rows across the site.

2. **Destructive-action confirmations.** Admin destructive actions (delete product, delete user, drop banner, wipe cart) that go through native `confirm()` instead of an inline modal. Native confirm is dismissed by browser "prevent additional dialogs" and doesn't localize consistently.

3. **Silent-failure `fetch(...)` patterns.** Admin action handlers doing `const res = await fetch(...); if (res.ok) refresh();` with NO catch and NO else. On a network blip the handler crashes with no toast, no rollback. The operator clicks again — nothing. Gives up.

4. **List refresh patterns.** After a CRUD action, does the list actually refetch? Or does it optimistically remove the row and desync from server state on failure? Find `setRows(prev => prev.filter(...))` without a matching refetch.

5. **Empty states / loading states that lie.** "0 comments" during a load error is indistinguishable from a genuinely empty thread. The operator sees "no engagement" during a transient blip and makes bad decisions.

6. **Rate limits on admin write routes.** Admin is trusted so usually no rate limit needed, but flag anything that could be catastrophic if misfired (bulk import, CSV export, image regenerate).

7. **Audit logging.** Every admin destructive/privileged action (role change, user delete, config change, export) should emit a structured audit event. Grep for the audit helper and identify write paths that don't emit.

8. **Idempotency / double-submit.** Admin forms that could double-submit — button disabled + submitting state? A pattern of "no disabling → double POST → duplicate row" is common.

9. **Authorization gates.** Every `/api/admin/*` route should call `requireAdmin()` (or equivalent) at the top. A missed call means anyone with a storefront cookie can hit admin routes.

10. **File-upload validation.** Content-Type allowlist? Size caps? Path-traversal guards? Post-upload virus scan?

11. **Generic CRUD dispatchers.** Any `POST /api/admin/crud/[table]` that trusts the caller's JSON payload wholesale? Per-table column allow-lists needed.

12. **JSON payload validation on POST/PATCH.** Missing shape validation. A malformed body could shove garbage into a JSON column.

13. **Response error leaks.** Do admin routes return sensitive error details (SQL error strings, connection strings, stack traces) in the JSON body? Fine to log to CloudWatch, not fine to return to the caller.

14. **Redundant `readOnly` inputs.** Admin forms with `readOnly` fields that are still in the submitted form data — a curl user can PATCH them.

15. **rowCount checks.** DELETE handlers that return `{ok: true}` regardless of whether a row was actually deleted. Parallel operators race and both see "success" for the same delete.

## Files to prioritize

- All `src/app/api/admin/**/route.ts` (or equivalent)
- The auth helper (`requireAdmin`, `requireSuperAdmin`, etc.)
- The CSRF helper (`assertSameOrigin`, or `csrf.ts`)
- Admin components — every `use client` under `src/app/admin/**`
- The generic CRUD dispatcher if one exists

## Grep starters

```
export async function (POST|PATCH|DELETE|PUT)   # find write handlers
assertSameOrigin                                 # cross-ref with CSRF helper
requireAdmin                                     # cross-ref with auth gate
confirm\(                                        # native confirm usage
if \(res\.ok\)                                   # silent-failure patterns
```

## What NOT to report

- "Consider using a form library" — style choice
- "Missing input validation" — must be specific: what field, what constraint
- Findings on routes that are already gated but could be "more secure" — needs a concrete threat model
