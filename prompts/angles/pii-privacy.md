# Angle: PII / privacy / audit trail

Paste this as the `{{ANGLE_SPECIFIC_CHECKLIST}}` in the audit-agent prompt.

Target: privacy-adjacent defects — unvalidated PII writes, missing audit logs, PII leaks in over-fetched responses, delete flows that leave zombie data. Applicable regardless of local regulation (GDPR, CCPA, PIPA, LGPD, PDPA, etc.).

## Look for

1. **Unvalidated PII on PATCH/POST.** Server accepts arbitrary strings for `phone`, `birthday`, `country`, `gender`, `postal_code`, `national_id` without format validation. Downstream code that trusts these values (`findCountry()`, SMS routing, CSV export) silently degrades.

2. **`SELECT *` on user/profile tables** returning fields the caller doesn't need. Every extra field is a potential PII surface if a future column holds sensitive data.

3. **Missing audit logs on write.** Writes to PII-bearing tables (user, profile, address, consent) should emit a structured audit event. Field NAMES only, not values (log values doubles the PII footprint).

4. **Consent lifecycle.** Marketing consent tracked as a boolean with no `consent_at` / `consent_source` / `consent_withdrawn_at` timestamps. If a regulator asks "prove this user gave consent on date X", you have a boolean.

5. **Account deletion completeness.** DELETE flow leaves rows in: comments, reviews, wishlists, chatbot leads, order history (should anonymize, not delete, for accounting compliance), audit logs (keep). Check every table that could reference the user.

6. **Zombie sessions after deletion.** Account deletion succeeds on the DB side but Cognito/Firebase/Auth0 delete is "best effort" and silently fails — user's session cookie still works for hours.

7. **Delete-flow friction.** Uses `window.confirm()` twice as the only friction. No email confirmation, no "download my data" export, no grace period.

8. **Data export missing.** GDPR/CCPA/PIPA-style data portability: is there a "download my data" endpoint? Or does the user have to email support?

9. **Cross-tab race / last-write-wins.** No optimistic concurrency control on profile PATCH. Two tabs open → tab A edits phone → tab B (loaded before A saved) hits Save with stale form → clobbers A's edit silently.

10. **Full PII displayed in the clear.** Profile page renders email/phone/birthday verbatim. Shoulder-surfing risk. Screenshot-to-support-ticket risk. No masking / show-hide toggle.

11. **PII in logs.** Grep `console.log`, `console.error` for lines that log email addresses, phone numbers, birthdays, addresses. Hash them (use SHA-256 truncated) before logging.

12. **PII in URL query strings.** GET `/api/user?email=foo@bar.com` — email lands in server logs, CDN logs, referrer headers.

13. **Reserved / staff-name impersonation on user-content fields.** Any comment / post / review form that lets a customer set their display name to "Admin", "Staff", "관리자" (Korean for "admin"), "サポート" (Japanese for "support") — need a reserved-name blocklist with NFKC normalization.

14. **Rate limits on account creation, password reset, comment/post.** Missing rate limits let one leaked credential flood the site.

15. **Delete cascades leaving orphans.** Foreign keys with no ON DELETE clause — deleting a user leaves orphan comments/orders/wishlist rows referencing a nonexistent user_id.

16. **Custom-fields JSONB bloat.** Any `custom_fields: Record<string, string>` shipped straight into a jsonb column with no key-count cap, no value-length cap, no key-name allow-list? A compromised browser extension could inject arbitrary keys.

## Files to prioritize

- `/api/customer/profile/*` (or equivalent PATCH/DELETE for user profile)
- `/api/customer/me/*` (or session/identity route)
- Sign-up / registration routes
- Account deletion routes
- Any component displaying PII
- Audit log helper (grep for one; if there isn't one, that IS the finding)
- Cognito / Firebase / Auth0 admin helpers

## Grep starters

```
customer_profiles|user_profiles|users     # find profile helpers
SELECT \*.*(FROM|from).*(user|profile)     # over-fetch on user tables
auditLog|audit_log|audit\.emit             # find audit helper
console\.log.*email|console\.log.*phone    # potential PII logs
window\.confirm                            # weak delete friction
```

## What NOT to report

- "Should encrypt PII at rest" — infra decision, not code
- "Consider consent management platform" — vendor recommendation
- "GDPR requires X" — cite the specific defect, not the regulation
