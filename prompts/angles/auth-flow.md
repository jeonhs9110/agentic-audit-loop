# Angle: Auth flow completeness

Paste this as the `{{ANGLE_SPECIFIC_CHECKLIST}}` in the audit-agent prompt.

Target: sign-up, sign-in, refresh, password reset, MFA, session lifecycle, logout, session invalidation. Every one of these is a place where users get stuck or attackers get through.

## Look for

1. **Sign-up: unverified email login.** Can a user log in before verifying their email? Should they be able to? Consistent behavior across social and email sign-up?
2. **Sign-up: rate limit missing.** Bots create thousands of dummy accounts to spam or reserve usernames. Rate limit at 10/hour per IP is a reasonable start.
3. **Sign-up: duplicate email handling.** Attempting to register with an existing email — what happens? Reveals account existence to attackers if the error message differs from "success" too obviously.
4. **Sign-in: enumeration.** Login errors that say "wrong password" (implies account exists) vs "user not found" — pick a generic message that doesn't leak existence.
5. **Sign-in: rate limit missing.** Brute force on login. Rate limit at 5/hour per IP + per email is common.
6. **Sign-in: session cookie configuration.** `httpOnly: true`, `secure: true` (prod), `sameSite: 'lax'`, `path: '/'`. Any deviation is worth a look.
7. **Sign-in: token TTL.** Access token TTL vs refresh token TTL — do they match the intended UX?
8. **Refresh: silent-refresh implementation.** Does the client wrap `fetch` in an `authedFetch` that catches 401, refreshes, and retries? Or does every 401 kick the user to login?
9. **Refresh: cookie path consistency.** Sign-in sets refresh cookie at `path: '/'`; refresh route sets it at `path: '/api/auth/refresh'`. Browser stores two independent cookies. Logout only clears one path. Auth material leaks after logout.
10. **Refresh: sliding vs fixed window.** Refresh token TTL slides forward on each refresh (30-day sliding) or fixed at 30 days from issue? Pick one, document it.
11. **Password reset: token TTL.** Reset tokens should expire in ~1 hour. 24 hours is too long. Days is a bug.
12. **Password reset: single-use enforcement.** Reset tokens must be single-use — no reusing a token after a successful reset.
13. **Password reset: no account enumeration.** "If an account exists at this email, we sent a link" — same message whether the account exists or not.
14. **Password reset: rate limit on request.** 3-5/hour per IP + per email.
15. **Logout: server-side invalidation.** Does logout revoke the refresh token on the auth provider? Or just clear the client cookie? Cookie-clear-only means a stolen refresh token stays valid.
16. **Logout: session cleanup across tabs.** BroadcastChannel or storage event to log out all tabs at once.
17. **Session expiration UX.** Silent 401 on the next request — does the client show a "session expired" message and redirect to `/login?next=<current_path>`? Or does the request just fail?
18. **MFA (if applicable).** Backup codes single-use? Recovery flow? Time-based OTP window (30-60s)?
19. **Password policy.** Length minimum (12+), complexity requirements, breach-check against HaveIBeenPwned or equivalent.
20. **Federated sign-in edge cases.** OAuth/Kakao/Google login without email scope — how does the app handle a user with no email address? Silent 500, or graceful?
21. **Reserved usernames / display names.** "Admin", "Staff", "System", "Support" — allowed as usernames or blocked? Impersonation vector.

## Files to prioritize

- `/api/auth/*` routes (sign-up, sign-in, refresh, sign-out, verify-email, reset-password)
- Auth helper functions (`requireCustomer`, `requireAdmin`)
- Client-side auth wrapper (`authedFetch` or equivalent)
- Login/register/reset pages
- Cognito/Firebase/Auth0/Supabase admin helpers
- Session-expiry listener (grep for `session-expired`)

## Grep starters

```
signIn|signOut|signUp                  # auth entry points
refresh_token|refresh-token            # refresh handling
password reset|forgot-password         # reset flow
cookies\.(set|delete)                  # cookie management
requireCustomer|requireAdmin           # auth guards
```

## What NOT to report

- "Add MFA" — feature request unless MFA is expected in the threat model
- "Use passkeys" — feature request
- Specific password-policy tuning — needs product decision, not a defect
