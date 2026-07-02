# Angle: API design + versioning

Paste this as the `{{ANGLE_SPECIFIC_CHECKLIST}}` in the audit-agent prompt.

Target: REST/GraphQL API design defects. Breaking changes, status code misuse, pagination patterns, error envelope consistency, versioning strategy.

## Look for

1. **Inconsistent error envelopes.** Some routes return `{ error: "message" }`, others `{ ok: false, error: {...} }`, others just `{ message: "..." }`. Pick one shape, apply everywhere. Callers can't build reliable error handling against inconsistent shapes.
2. **Status code misuse.** Returning 200 with `{ ok: false }` — clients that check `res.ok` see success. Use 400 for client error, 401 for unauthenticated, 403 for unauthorized, 404 for not found, 409 for conflict, 429 for rate limit, 500 for server error.
3. **404 vs 200-with-null.** GET `/api/users/nonexistent` — should return 404, not 200 with null. Client caching semantics depend on it.
4. **Missing versioning.** No `/api/v1/` prefix. When a breaking change ships, every client breaks simultaneously.
5. **Breaking changes to public routes.** Adding a required field. Removing a field. Renaming a field. Changing a type. Look for changes since the last release that would break existing clients.
6. **Unbounded pagination.** `GET /api/items` returning every row. Even `?limit=` optional but no server-side cap is dangerous.
7. **Deep OFFSET pagination.** `?page=1000&limit=20` — full table scan. Use keyset pagination for anything the client can paginate deeply.
8. **Sorting on unindexed columns via query param.** `?sortBy=arbitrary_col` without validating against an allow-list.
9. **Filtering on unindexed columns via query param.** Same as sorting — validate against an allow-list.
10. **Idempotency on POST.** Payment / order / notification POSTs should accept an `Idempotency-Key` header and dedupe. Otherwise a network retry double-charges.
11. **Missing `Content-Type` validation.** Route accepts JSON but doesn't check `Content-Type: application/json`. A form-urlencoded body with hostile keys will still parse.
12. **Missing size caps on request body.** A 100MB POST body eats memory. Reject at the middleware layer.
13. **Verbose validation errors leaking schema.** `{ error: "expected object, got string at .user.profile.address.street" }` — leaks the internal shape.
14. **Missing `Retry-After` on 429.** Clients need to know how long to back off.
15. **Missing `WWW-Authenticate` on 401.** Not strictly required but expected.
16. **CORS `Access-Control-Allow-Origin: *` on authenticated routes.** Wide open.
17. **Rate limits missing on unauthenticated routes.** Sign-up, password reset, contact-form, chatbot — every unauthenticated write endpoint needs a rate limit.
18. **Ambiguous plural/singular routes.** `/api/user` vs `/api/users` — pick one convention. Both existing is a bug.
19. **HTTP verb misuse.** `GET /api/delete-user?id=...` — GET should be safe/idempotent. DELETE for deletes, PATCH for partial updates.
20. **Missing route documentation.** No OpenAPI spec / GraphQL schema exported anywhere. External integrators guess.

## Files to prioritize

- Every `src/app/api/**/route.ts` (Next.js) or `src/routes/**` or `pages/api/**`
- API gateway / router config
- Rate-limit helper (grep `createRateLimiter`)
- CORS middleware
- Error-handling middleware / global exception handler
- OpenAPI or GraphQL schema files (if any)

## Grep starters

```
export async function (GET|POST|PATCH|DELETE|PUT)   # route handlers
status: 200                                          # status usage
NextResponse.json                                    # response shape
Content-Type                                         # content-type handling
Access-Control-Allow                                 # CORS setup
```

## What NOT to report

- "Consider GraphQL" (or REST) — style choice, not a defect
- "API is not documented" — flag as handoff item once, not per-route
- "Endpoint could be faster" — perf, use `perf-cwv` or `db-query-hygiene` instead
