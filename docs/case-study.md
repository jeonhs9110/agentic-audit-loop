# Case study: one representative round

A walk-through of a real closed-loop round on a real production Next.js codebase (specifics anonymized). The goal is to make the abstract pattern concrete.

## Setup

- Codebase: Next.js 16 App Router / React 19 / TypeScript 5, hosted behind ALB + CDN, Postgres backend.
- Prior rounds had already caught the CRITICAL structural bugs — this round targeted the middle layer.
- Angles chosen: **SEO / meta / i18n** and **checkout / phase-2 stub honesty**.

## Discovery

Two audit agents spawned in parallel with the prompts from [`prompts/angles/seo-meta-i18n.md`](../prompts/angles/seo-meta-i18n.md) and a custom "phase-2 stub honesty" angle (the app had order/payment/checkout endpoints that weren't wired yet — the audit was whether the UI misled customers about them).

Each agent ran ~60 tool calls over ~5 minutes.

### SEO agent returned 15 findings

Highlights:

- **CRITICAL**: Root layout set `title.template = '%s · BRAND'` and every child page returned `title` as a string that ALREADY ended with `· BRAND` — so titles rendered as `Category · BRAND · BRAND` (double suffix, >60 chars, Google truncates).
- **CRITICAL**: `alternates.languages` used `{ kr: '...', en: '...' }` across 11 pages. Real BCP47 is `ko-KR` / `en-US` — `kr` isn't a language code. Google's validator was silently dropping the entire hreflang cluster.
- **CRITICAL**: OG image on 6 pages pointed to `/brand-logo.svg`. SVG isn't a valid OG image format on Kakao / Facebook / Twitter. Every share rendered broken.
- **HIGH**: Sitemap emitted `/kr/products/{id}` and `/en/products/{id}` as SEPARATE entries with no `<xhtml:link rel="alternate">` cross-refs. Google couldn't discover the cluster from the sitemap either.
- **MEDIUM/LOW**: 11 more findings — description leakage, geo-header trust, canonical patterns, product JSON-LD `priceValidUntil` missing.

### Phase-2 honesty agent returned 13 findings

Highlights:

- **HIGH**: Header nav showed "Order Lookup" as a first-class link → clicking landed on a MyPage tab that said "orders coming after payment integration". Nav promised a feature that didn't exist.
- **HIGH**: Chatbot system prompt asserted concrete shipping / return / payment policy ("free shipping over $50", "returns within 7 days", "credit card, ApplePay, bank transfer") — none of which the site actually honored.
- **HIGH**: Cart summary rendered a hardcoded "Shipping: Free" line regardless of order value, but the chatbot promised free shipping only over $50. The two lied about each other.
- **HIGH**: Orphaned `PromoBanner.tsx` component + three translation keys (`coupon.newMember`, `promo.message`, `promo.hideToday`) that had zero call sites. A trap for future engineers — "the copy is here, wire it up" would promise a coupon nobody built.
- **HIGH**: Admin payments page empty-state instructed operator to "run this SQL in Supabase" — but the site had moved off Supabase months ago. Operator would run SQL against a dead database.
- **MEDIUM/LOW**: 8 more.

## Planning: safe-subset sort

Apply now (16 total):
- All 4 CRITICAL SEO fixes + the sitemap consolidation
- 5 more MEDIUM SEO fixes (root description move, geo-header fix, product OG dims, JSON-LD priceValidUntil, first-img extract for post/page OG)
- Root not-found.tsx (was missing entirely)
- Language switcher preserves query string
- All 6 phase-2 honesty fixes (Header relabel, chatbot prompt update, cart free-shipping copy, delete PromoBanner + dead translations, admin payments empty-state, profile docstring)

Defer with reason:
- Ship a real `/public/og-default.png` (1200×630) — needs a binary asset a code-only round can't create
- `www` vs bare-domain canonical enforcement — CDN behavior, not code
- Shipping-addresses schema — needs migration + product decision
- Server-side cart_items table — needs schema + backend work
- JSON-LD AggregateRating — needs review-aggregate read path

Reject:
- One LOW polish finding on a menu-canonical-alias generalization — not worth reviewer time this round

## Execution

Fixes applied in one contiguous batch:
- 11 hreflang-code fixes (`kr` → `ko-KR`, `en` → `en-US`, add `x-default`)
- Root layout template dropped, layout description moved
- Sitemap refactored into `alternates.languages` shape
- Product JSON-LD `priceValidUntil` added (with an eslint-disable comment noting the RSC context)
- First-image extraction regex added for post/page/menu OG paths
- Root `not-found.tsx` created
- Language picker preserves `window.location.search`
- Header nav relabel: "Order" → "Order Info"
- Chatbot system prompt stripped of concrete shipping/return/payment claims
- Cart "Shipping: Free" → "Calculated at checkout"
- `PromoBanner.tsx` deleted, translation keys removed
- Admin payments empty-state replaced

Total files touched: 23. Diff: +290, -168.

## Verification

- `tsc --noEmit` — clean
- `eslint .` — clean (after adding one targeted eslint-disable comment on the `Date.now()` inside JSON-LD; server components can safely call it but the react-hooks/purity rule is client-oriented)

## PR body

Followed [`templates/pr-body.md`](../templates/pr-body.md). Grouped findings by angle, listed test-plan checkboxes, listed 8 skipped items with reasons, appended the trajectory table showing the previous 5 rounds.

Reviewer merged after ~30 minutes.

## Signals to take forward

- **Two non-overlapping angles paid off.** SEO agent didn't touch the phase-2 stub. Phase-2 agent didn't touch the metadata files. Combined coverage was ~40 real files audited without duplicate reads.
- **Skipped items became actionable.** The "ship og-default.png" item got a real designer engagement two weeks later. "Shipping addresses schema" turned into a handoff-doc item for the vendor.
- **CRITICAL find rate was still high** at this round → more rounds worth running. Two rounds later CRITICAL rate dropped to zero and the loop wound down.

## What this case study is NOT

This isn't a template you can copy-paste. Your findings will be different. Your severity thresholds will be different. Your safe-subset judgment will develop with experience.

What TO take from it:
- Two angles > one angle
- CRITICAL findings look like the ones above (real, specific, with a concrete failure mode) — not "consider refactoring X"
- Defer honestly and often
- The trajectory table is worth maintaining from round 1
