# Accessibility (a11y)

> **Semantic HTML first. ARIA is a fallback, not a strategy.**
> Accessibility is not optional — it is correctness. A product the keyboard can't drive is a broken product. Target **WCAG 2.1 AA** at minimum.

## The checks that catch 80% of issues

| # | Check | Pass criterion |
|---:|---|---|
| 1 | Keyboard-only navigation | Every interactive element reachable + operable with Tab / Shift+Tab / Enter / Space / Arrow |
| 2 | Visible focus | A clear outline on every focusable element — never remove without replacement |
| 3 | Semantic HTML | `<button>` for actions, `<a>` for navigation, `<label>` for inputs, `<nav>/<main>/<header>` for landmarks |
| 4 | Form labels | Every input has an associated `<label>` (via `for`/`id` or wrapping) |
| 5 | Image alt text | Meaningful `alt=""` or `alt=""` if decorative — never missing |
| 6 | Color contrast | Text ≥ **4.5:1**, large text (≥18pt / 14pt bold) ≥ **3:1** |
| 7 | Not color-only | Error/status also carries icon or text, not only red/green |
| 8 | Heading hierarchy | One `<h1>` per page, no skipped levels |
| 9 | Dynamic content | Live regions announce toasts, errors, async results |
| 10 | Modal focus | Trap focus inside; restore to trigger on close |

## Rules

- **Semantic HTML first.** `<button>` beats `<div role="button" tabIndex={0} onKeyDown={...}>` every time — the browser handles 12 things for you.
- **One `<h1>` per page. No skipped levels.** `h1 → h2 → h3`, not `h1 → h4`.
- **Every `<input>` has a `<label>`.** No `placeholder` as label — placeholders disappear on type.
- **Interactive = focusable.** Any `onClick` on a non-focusable element (`div`, `span`) is a bug.
- **Visible focus is mandatory.** `outline: none` without replacement is one of the most common a11y failures.
- **Icon-only buttons need `aria-label`.** `<button><TrashIcon /></button>` reads as "button" to a screen reader — nothing else.
- **Alt text is a description, not a filename.** `alt="hero.png"` is useless. Decorative images → `alt=""`.
- **Color never alone.** Required field → asterisk + text. Error → icon + message + color. Status → text + badge, not only color.
- **Live regions for async updates.** `aria-live="polite"` for toasts and non-urgent updates; `aria-live="assertive"` only for critical.
- **Modals trap focus** and restore it to the opening trigger on close. First focusable element gets focus on open.
- **Skip link** at the top of the page so keyboard users skip the nav.
- **Don't disable zoom.** `user-scalable=no` is an accessibility violation.
- **Target size** — interactive elements ≥ 24×24 CSS px (WCAG 2.2), ideally 44×44 for touch.
- **Prefer accessible primitives.** Radix UI, Headless UI, React Aria — behavior + a11y built in. Styled on top.

## Keyboard contract — the minimum

| Element | Keys |
|---|---|
| Button | `Enter`, `Space` |
| Link | `Enter` |
| Checkbox | `Space` |
| Radio group | Arrow keys to move, `Space`/`Enter` to select |
| Combobox / select | `Arrow` to move, `Enter` to pick, `Esc` to close |
| Modal | `Esc` to close, `Tab`/`Shift+Tab` cycle within |
| Tabs | `Arrow` to move, `Enter`/`Space` to activate |

Use `role` + `aria-*` only when a native element can't do the job.

## ARIA — rules of use

ARIA is a **patch**, not a feature. Prefer native HTML. When you must use ARIA:

- **First rule of ARIA: don't use ARIA.** If a native element exists, use it.
- Don't change native semantics (`<button role="link">` — just use `<a>`).
- `aria-label` on anything users can see must match the visible label.
- `aria-hidden="true"` on an interactive element is a trap — users can tab to something invisible to a screen reader.
- `role="button"` without `tabIndex={0}` and keyboard handlers is a bug.

## Testing — three layers

| Layer | Tool | When |
|---|---|---|
| Automated | **`axe-core`** / `jest-axe` / Playwright `@axe-core/playwright` | Every PR in CI — catches ~30-50% of issues |
| Manual keyboard | Just unplug the mouse | On every feature |
| Screen reader | **VoiceOver** (macOS), **NVDA** (Windows), TalkBack (Android) | On every meaningful feature |

Automated tools catch missing labels and bad contrast. They **cannot** tell you if the keyboard flow makes sense — only a human can.

## Red flags

- `<div onClick>` instead of `<button>`.
- Icon-only button with no `aria-label`.
- Modal that doesn't trap focus — Tab leaks to the page behind.
- Red border is the only indicator of a form error.
- `outline: none` with no `:focus-visible` replacement.
- `placeholder="Email"` with no `<label>`.
- Scroll container that only responds to mouse wheel.
- Hover-only tooltip with critical info.
- Carousel that autoplays and can't be paused.
- `aria-label` that doesn't match the visible text.
- Decorative image with `alt="icon"` instead of `alt=""`.
- "We'll add a11y later." (Later never comes; retrofitting is 10× the cost.)
