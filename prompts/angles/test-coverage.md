# Angle: Test coverage on critical paths

Paste this as the `{{ANGLE_SPECIFIC_CHECKLIST}}` in the audit-agent prompt.

Target: NOT total coverage %. Only tests on paths whose failure would cause customer/operator harm. Money paths, auth paths, PII delete paths, admin destructive paths, external-integration paths.

## Look for

1. **Auth entry points untested.** Sign-up, sign-in, refresh, sign-out — at least one happy-path + one wrong-password test each. If any auth route has no test at all, flag it.
2. **Payment / checkout paths untested.** If the app has checkout, every money-touching path should have integration tests (unit tests aren't enough — test the fetch → server → DB round trip).
3. **PII delete flow untested.** Account deletion is regulatory. A regression here creates zombie accounts. Test that the DELETE actually removes rows across every table.
4. **Admin destructive actions untested.** Delete product, delete user, drop table — each should have at least one integration test that verifies the delete happened AND permissions are enforced.
5. **CSRF guard untested.** For every write route that should reject cross-origin requests, one test that confirms a request without `Origin` matching the allow-list gets 403.
6. **Rate limiter untested.** Fire N+1 requests, confirm the (N+1)th returns 429.
7. **Silent-swallow catches untested.** Any `catch (err) { logSomething() }` — is the logging actually happening? Add a test that mocks the error and asserts the log fired.
8. **Third-party integration untested (with mocks).** OpenAI, Stripe, Twilio, SendGrid — each integration path should have a test using a mock client, exercising both success and failure responses.
9. **Cache invalidation untested.** When admin edits a product, does the storefront cache actually flush? Test the write → cache-tag emit → downstream read sees new value flow.
10. **Concurrency / race conditions untested.** Two concurrent PATCH requests on the same row — does last-write-wins? First-write-wins? Optimistic-conflict rejection? Whatever the answer, there should be a test.
11. **Feature flags untested.** If the app has a feature flag, tests for both flag-on and flag-off paths.
12. **Migration reversibility untested.** For every schema migration, an equivalent down-migration should exist and be tested (rare but valuable when needed).
13. **Test file naming inconsistency.** `Foo.test.ts` mixed with `Foo.spec.ts` mixed with `test-foo.ts`. Not a bug per se; flag for consistency.
14. **E2E on critical paths missing.** Playwright / Cypress smoke tests on the checkout, sign-up, and sign-in flows should exist. If not, that's the flag.
15. **Snapshot tests on volatile output.** Snapshot tests that break on every unrelated change are noise. Prefer explicit assertions.
16. **Tests that don't actually assert.** `it('foo', () => { doThing() })` with no `expect`. Passes by not throwing — silently useless.
17. **Skipped tests (`.skip` / `xit` / `xdescribe`) not tracked.** Every skip should have a comment linking to why it's skipped + when it'll be re-enabled.
18. **Tests that hit real external services.** Real Stripe API calls in the test suite. Real DB writes. Real email sends. Every one is a flakiness landmine.

## Files to prioritize

- `tests/e2e/` (Playwright / Cypress)
- `src/**/__tests__/` or `tests/**/*.test.ts`
- `package.json` scripts (`test`, `test:e2e`, `test:integration`)
- CI workflow files (which tests run on PR?)
- Auth routes + tests for auth routes
- Payment / checkout routes + tests

## Grep starters

```
\.test\.(t|j)sx?$                  # unit test files
\.spec\.(t|j)sx?$                  # spec files
describe\(|it\(|test\(             # test cases
\.skip\(|xit\(|xdescribe\(         # skipped tests
expect\(                           # assertion count vs test count
```

## What NOT to report

- "Increase test coverage to N%" — coverage %s are a lagging indicator; specific paths that lack tests are the finding
- "Add unit tests for X" — vague; must name the specific untested behavior
- "Consider TDD" — process opinion
