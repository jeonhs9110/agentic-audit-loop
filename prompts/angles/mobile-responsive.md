# Angle: Mobile + responsive

Paste this as the `{{ANGLE_SPECIFIC_CHECKLIST}}` in the audit-agent prompt.

Target: mobile-first UX defects. Touch targets, safe-area insets, viewport meta, hover-only interactions, orientation handling, in-app WebView quirks (KakaoTalk, Instagram, Facebook, WeChat).

## Look for

1. **Viewport meta missing or wrong.** No `<meta name="viewport" content="width=device-width, initial-scale=1">`? No `viewport-fit=cover` when the app has full-screen media?
2. **Touch targets below 44×44px.** WCAG minimum. Common failures: close-X in modals, icon-only nav buttons, small chip filters, pagination arrows.
3. **Hover-only interactions.** Menus that open on hover with no click/tap fallback. Product galleries that reveal info only on hover.
4. **Fixed positioning breaking on mobile.** Sticky headers that don't account for URL bar collapse. Fixed FABs that overlap iOS home indicator without `env(safe-area-inset-bottom)`.
5. **Safe-area insets not applied.** iOS notch / home indicator / Android gesture bar — `padding-top: env(safe-area-inset-top)` and `padding-bottom: env(safe-area-inset-bottom)` on the relevant edges.
6. **In-app WebView cookie filtering.** KakaoTalk and Instagram in-app WebViews strip path-scoped cookies. Auth refresh cookies must use `path: '/'`.
7. **In-app WebView JS-history issues.** iOS Kakao WebView mishandles POST-response `location.assign()` — use `location.replace()` for post-auth redirects.
8. **`vh` units breaking on mobile.** `100vh` includes the URL bar on mobile Safari — content clips when the bar retracts. Use `100dvh` or `100svh`.
9. **Horizontal overflow.** Any element with a fixed width that exceeds the mobile viewport (usually 375-393px) forces horizontal scroll on the whole page.
10. **`inputmode` missing on inputs.** Numeric fields should have `inputmode="numeric"`. Phone fields `inputmode="tel"`. Email `inputmode="email"`. Wrong keyboard = wrong values.
11. **`autocomplete` hints missing.** iOS and Chrome autofill are much better when `autocomplete="name" | "tel" | "email" | "postal-code" | "country-name"` etc. are present.
12. **`type="number"` for postal codes / phone numbers.** These aren't numbers (leading zeros matter). Use `type="text"` + `inputmode="numeric"`.
13. **`font-size: 16px` minimum on inputs (iOS).** iOS Safari auto-zooms into inputs with font size < 16px. UX-breaking.
14. **Modals covering the full viewport but not scrollable.** On mobile, a modal taller than the viewport with no overflow-scroll leaves content unreachable.
15. **Tap delay from missing `touch-action: manipulation`.** 300ms tap delay on some Android browsers when `touch-action` isn't set. Add on `<html>` or interactive elements.
16. **Interactive elements at the bottom edge.** iOS pull-up-to-refresh + Android back-swipe gestures conflict with bottom-edge tap targets. Move interactive elements at least 20px away from edges.
17. **Landscape orientation handling.** Modals, forms, and video players that assume portrait — landscape breaks the layout or hides submit buttons.
18. **Image intrinsic dimensions on `<Image fill>`.** Without `sizes`, next/image over-fetches. Under 4G this doubles bandwidth cost.

## Files to prioritize

- Root layout (viewport meta + safe-area setup)
- Header + navigation (sticky positioning + safe-area)
- Every modal / drawer / dialog component
- Forms (input attributes)
- Product / hero / gallery components
- Auth flows (login/register — especially cookie path if in-app WebView is a customer surface)

## Grep starters

```
<meta name="viewport"              # viewport meta setup
env\(safe-area-inset               # safe-area usage
inputmode=                         # keyboard hints
autocomplete=                      # autofill hints
100vh                              # legacy viewport units
touch-action                       # tap-delay guards
```

## What NOT to report

- "Consider a mobile app" — off scope
- "Would benefit from PWA support" — feature request unless there's a specific PWA defect
