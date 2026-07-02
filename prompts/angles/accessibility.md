# Angle: Accessibility (WCAG 2.1 AA)

Paste this as the `{{ANGLE_SPECIFIC_CHECKLIST}}` in the audit-agent prompt.

Target: keyboard-only navigation, screen reader announcements, focus management, color contrast, semantic HTML. Aim for WCAG 2.1 AA compliance as the practical bar.

## Look for

1. **Missing form labels.** Every `<input>` needs a `<label htmlFor>` or aria-label. Placeholder text is NOT a label.
2. **Icon-only buttons without aria-label.** `<button><TrashIcon /></button>` announces as "button" to a screen reader — useless.
3. **Non-semantic clickable divs.** `<div onClick={...}>` isn't keyboard-focusable and isn't announced as interactive. Use `<button>` or add `role="button"` + `tabIndex={0}` + `onKeyDown` for Enter/Space.
4. **Focus traps missing on modals.** A modal open with Tab escaping to the page underneath is unusable with a keyboard.
5. **Focus restoration on modal close.** Focus should return to the element that opened the modal.
6. **`aria-live` regions missing.** Dynamic content (toasts, form errors, cart-count updates) should announce to screen readers. Grep for `role="status"` / `role="alert"` / `aria-live`.
7. **Color contrast below 4.5:1 (text) or 3:1 (UI).** Common failures: light-gray placeholder text (`text-neutral-400` on white), disabled buttons with washed-out contrast, brand colors on white backgrounds.
8. **Focus indicators removed or invisible.** `outline: none` without a replacement `:focus-visible` style leaves keyboard users lost.
9. **Skip-to-content link missing.** WCAG 2.4.1 Bypass Blocks. Keyboard users press Tab and traverse 20+ header links before reaching page content.
10. **Heading hierarchy broken.** `<h1>` → `<h3>` skipping `<h2>`. Multiple `<h1>` on one page. `<h1>` used for styling rather than semantics.
11. **`alt` text quality.** `alt=""` on informative images. `alt="image"` (redundant). `alt="[product name]"` when the product name is already visible next to the image (should be empty then).
12. **Language attributes.** `<html lang="...">` set correctly per page? Mixed-language content uses inline `<span lang="...">`?
13. **Touch targets under 44×44px.** WCAG 2.5.5 Target Size. Common failures: icon-only buttons, small close-X in modals, tab bar icons.
14. **`readOnly` inputs that should be display text.** A `readOnly` input announces as an input to a screen reader — misleading. Use a `<p>` or `<span>`.
15. **Autoplay video/audio.** WCAG 1.4.2 Audio Control. Autoplaying media with sound is a hard fail.
16. **Forms without error announcements.** A submit that fails should focus the first error field and announce the error via `aria-describedby` or `role="alert"`.
17. **Modals that don't return `role="dialog" aria-modal="true"`.** Or don't have an aria-labelledby pointing at the title.
18. **Table structure.** Data tables without `<th>` and `<caption>`.

## Files to prioritize

- Every form component
- Every modal / dialog / drawer component
- Every icon-only button (grep for `<button>{` followed by `Icon`)
- Header + navigation components
- Toast / snackbar / error banner components
- The root layout (for `<html lang>` and skip-link)

## Grep starters

```
<button                            # count buttons; look for icon-only ones
placeholder=                       # candidates for missing labels
onClick={.*} className=".*div"     # clickable divs
outline: none                       # focus-indicator removals
aria-label                         # inventory a11y attributes
role="                             # inventory ARIA roles
```

## What NOT to report

- "Consider a11y testing" — vague
- Full WCAG conformance audit — too broad for one round; focus on the operationally-common patterns above
