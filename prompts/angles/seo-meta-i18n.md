# Angle: SEO / meta / i18n / social share

Paste this as the `{{ANGLE_SPECIFIC_CHECKLIST}}` in the audit-agent prompt.

Target: organic search + discoverability defects. Broken hreflang clusters, blocking Google/Naver/Kakao indexing, broken/misleading social-share previews.

## Look for

1. **`robots.txt`.** Exists? Correct? Blocks `/admin`, `/api`, `/login`, `/register`? Sitemap URL correct?

2. **Sitemap.** Exists? Dynamic route (`app/sitemap.ts` in Next.js)? Includes products + posts + pages? hreflang alternates? Correct lastmod?

3. **Canonical URLs.** `<link rel="canonical">` emitted consistently across locale variants? Not just on the default locale?

4. **hreflang.** `<link rel="alternate" hreflang="ko-KR" ...>` and `en-US` pairs present on every localized page? Watch out for INVALID BCP47 codes like `hreflang="kr"` (not a language code — `ko` is Korean) or `hreflang="jp"` (not a language code — `ja` is Japanese). Google's validator silently drops the whole cluster.

5. **`x-default` hreflang.** Set on every localized page? Points to the same variant everywhere (e.g. `/en`)?

6. **Open Graph + Twitter Cards.** `og:title`, `og:description`, `og:image`, `og:url`, `twitter:card` present on product / post / homepage / category pages? Images the right size (1200×630 landscape or 1200×1200 square for e-commerce)? Explicit `og:image:width` + `og:image:height`?

7. **`og:image` format.** SVG is NOT a supported OG image format on Kakao / Facebook / Twitter / LinkedIn. If the fallback image is `/logo.svg`, every share renders as broken-image or text-only.

8. **JSON-LD structured data.** Product schema on product pages? Organization schema on the root? BreadcrumbList on nested pages? Review/AggregateRating on products with reviews? `priceValidUntil` on Offer (required field when price is set).

9. **Meta descriptions.** Unique per page (not just a global default)? Not empty? Under 155 chars? English default leaking onto localized pages?

10. **Title tags.** Unique per page? Template pattern applied? Under 60 chars? Watch for double-brand suffix from template + child both appending the brand name.

11. **Naver / Baidu / Yandex site verification.** Present in root layout? Not overridden by child pages?

12. **Accidental noindex.** Any `robots: { index: false }` metadata on pages that SHOULD be indexed? Common with copy-pasted "not found" metadata.

13. **404/500 pages.** Root `not-found.tsx` exists (not just `/[lang]/not-found.tsx`)? Emits `noindex`?

14. **Language switcher.** Preserves the current path? Writes the language cookie? Preserves query string?

15. **Redirect loops.** Any `/kr → / → /kr` patterns? Middleware misroutes?

16. **Duplicate-content risk.** `www.` vs bare-domain enforced (canonical or redirect)? Trailing-slash normalization consistent?

17. **Geo-header trust.** Any code reading `x-vercel-ip-country` on a non-Vercel deploy? Or `cf-ipcountry` on a non-Cloudflare deploy? These headers are client-spoofable when the CDN that injects them isn't in front.

## Files to prioritize

- `next.config.ts` — redirects, headers
- `src/middleware.ts` or `src/proxy.ts` — geo-routing
- `src/app/robots.ts` / `src/app/sitemap.ts` (if Next.js App Router)
- `src/app/layout.tsx` + `src/app/[lang]/layout.tsx`
- `src/app/[lang]/page.tsx` (homepage)
- `src/app/[lang]/products/[id]/page.tsx` (product detail)
- `src/app/[lang]/posts/[slug]/page.tsx` (or equivalent CMS)
- `src/app/not-found.tsx`, `src/app/[lang]/not-found.tsx`
- Language switcher / picker component

## Grep starters

```
languages:                         # hreflang cluster definitions
generateMetadata                   # per-page metadata
openGraph:                         # OG metadata
priceValidUntil                    # JSON-LD Offer
naver-site-verification            # verification tags
```

## What NOT to report

- "Consider adding structured data for X" — must be a real defect, not a suggestion
- "Sitemap could include more URLs" — vague
- Findings that would need a CMS/schema change to fix if the audit is code-only (but flag as "handoff item")
