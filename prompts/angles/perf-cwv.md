# Angle: Core Web Vitals + mobile perf

Paste this as the `{{ANGLE_SPECIFIC_CHECKLIST}}` in the audit-agent prompt.

Target: LCP, INP, CLS, TBT, bundle size, hydration cost, bandwidth.

## Look for

1. **LCP images.** Hero image not `priority`. Missing `sizes`. PNG where WebP/AVIF would win. Unbounded `fill` without `sizes`. Two LCP candidates competing on the preload scanner (e.g. mobile + desktop twins both firing eagerly). `remotePatterns` misconfigured so `<Image>` falls back to plain unoptimized URLs.

2. **Blocking scripts + stylesheets.** `<link rel="stylesheet">` in `<head>` on the critical path — Google Fonts, brand-CDN CSS, Typekit. Any `beforeInteractive` script that could be `lazyOnload`. Non-async third-party scripts (Meta Pixel, Naver script, hotjar-style).

3. **Client-component boundary depth.** `'use client'` higher up the tree than necessary. A Server Component that renders a Client wrapper which mostly passes props. Every unnecessary boundary ships an entire subtree to the browser.

4. **Bundle bloat.** Giant single-purpose deps loaded eagerly: `recharts`, `xlsx`, `@react-pdf/renderer`, `date-fns` full import instead of tree-shaken. Any admin-only dep pulled into the storefront bundle.

5. **`<Image>` misuse.** Missing `alt`. `alt=""` where meaningful text exists. `width`/`height` on `fill` images. Wrong `sizes` causing 2-3x overfetch. `priority` on below-the-fold images. Plain `<img>` for user-uploaded images (no lazy, no async decode, no responsive srcset).

6. **Hydration overhead.** Giant `dangerouslySetInnerHTML` blocks that could be static HTML. Large `JSON.stringify` payloads in `<script>` tags rendered per-request. Client components whose only job is `localStorage.setItem` on mount (should be an inline `<script>`).

7. **`useEffect` fetch-on-mount patterns that should be SSR.** Every mount-fetch adds a waterfall step. Config fetches for global chrome (chatbot config, feature flags, theme) especially painful.

8. **Client-side sort/filter over full catalog on keystroke.** Fine to filter 100 rows in memory; not fine to re-fetch on every keystroke.

9. **Client-side date libs.** `date-fns`, `moment`, custom date formatting shipped to the browser when `Intl.DateTimeFormat` would work.

10. **Font strategy.** Multiple weights preloaded that aren't referenced. `font-display: swap` missing. Non-Latin subset not carved out. TTF where WOFF2 would cut 60-70%.

11. **`revalidate` / cache TTL patterns.** RSC data-fetches without `revalidate` or with `force-dynamic` where a 60s cache would be fine.

12. **Analytics / third-party overhead.** Google Consent Mode v2 gate present? Anything not gated?

13. **Suspense boundaries missing.** Below-the-fold sections blocking hero paint. Third-party fetches from RSC (translation, RSS, geo) added to first-byte latency.

14. **Overfetching in RSC.** `SELECT *` on tables where the page reads 3 columns. `getAll()` followed by `.find(...)` for a single row.

15. **Third-party iframes at scale.** N Instagram/YouTube/Twitter embeds rendered eagerly. Each pulls 300-500KB+ of third-party runtime. Usually a static thumbnail + link is enough.

## Files to prioritize

- `src/app/layout.tsx`, `src/app/[lang]/layout.tsx` (or equivalent root layouts)
- `src/app/page.tsx`, `src/app/[lang]/page.tsx` (homepage — the LCP target)
- Hero / carousel components (they own the LCP element)
- Product / listing components (highest-repeat surface)
- `next.config.ts` — image config, headers, remotePatterns
- `package.json` — dep list
- Any component that grep matches `'use client'` at the top of the file

## Grep starters

```
'use client'                       # count client boundaries
<Image                             # find image usage
<link rel="stylesheet"             # find blocking CSS
useEffect.*fetch                   # find fetch-on-mount patterns
dangerouslySetInnerHTML            # find raw HTML injection
```

## What NOT to report

- "This component could use `React.memo`" — not a bug
- "Consider code splitting" — too vague
- "Bundle size might be large" — needs a concrete number and a concrete fix
- Findings that would require adopting a new framework or CDN
