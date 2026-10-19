# CSS Practice

> **Style with tokens, not literals. Scope to components, not pages. Keep specificity low.**
> Most CSS pain — overrides, drift, dead rules — comes from forgetting one of these three. Pick a strategy, apply it consistently, and let the system do the work.

## Pick ONE styling strategy — don't mix

| Strategy | Fits | Avoid when |
|---|---|---|
| **Tailwind CSS** | App w/ a design system, fast iteration, strong token discipline out of the box | You need deep themability without a plugin + many brands |
| **CSS Modules** | Scoped, standard CSS, no runtime cost, works everywhere | Design-token sharing across packages (needs `postcss-custom-properties` or similar) |
| **CSS-in-JS** (vanilla-extract, Panda, Emotion) | Heavy runtime theming, dynamic styles tied to props | Perf-sensitive apps — runtime cost; prefer zero-runtime variants (vanilla-extract, Panda) |
| **Plain CSS + BEM** | Marketing/CMS sites, no component framework | Any medium+ app — scales poorly, specificity wars |

Mixing two in the same codebase ("just a bit of CSS-in-JS inside our Tailwind app") always ends in a cleanup epic. **Pick one. Document it. Enforce it in review.**

## Design tokens — the only values allowed

Hard-coded values are the #1 source of visual drift. **Every color, spacing, radius, shadow, and font value comes from a token.**

| Property | Source | Example |
|---|---|---|
| Color | `theme.colors.*` / CSS variable | `--color-brand-500`, not `#3366ff` |
| Spacing | `theme.spacing.*` / scale | `space-4` (16px), not `padding: 12px` |
| Font size | `theme.text.*` / scale | `text-sm`, not `font-size: 13.5px` |
| Radius | `theme.radii.*` | `radius-md`, not `border-radius: 6px` |
| Shadow | `theme.shadows.*` | `shadow-elevated`, not `0 2px 4px rgba(0,0,0,0.1)` |
| Breakpoint | `theme.screens.*` | `md:` / `--bp-md`, not `@media (min-width: 760px)` |

```css
/* Bad — magic numbers everywhere */
.card { padding: 14px; border-radius: 5px; color: #333; }

/* Good — tokens via CSS variables */
.card { padding: var(--space-4); border-radius: var(--radius-md); color: var(--color-fg-default); }
```

CSS variables (`--token-name`) are the lingua franca — Tailwind, CSS Modules, and CSS-in-JS can all consume them.

## Rules

- **Keep specificity low and flat.** Prefer one class selector. No `#id` for styling. Avoid chains like `.a .b .c .d`.
- **No `!important`.** It's a confession that specificity got away from you. Fix the cascade, not the symptom.
- **Scope styles to the component.** CSS Modules give you this free; Tailwind by colocation; CSS-in-JS by construction. Global styles live in exactly one `global.css`.
- **Mobile-first media queries.** Base styles are mobile; `@media (min-width: …)` adds for larger. Avoid the inverse — it's harder to reason about.
- **Layout = Flexbox or Grid.** No `float`, no `position: absolute` for layout. `inline-block` only for inline content.
- **Logical properties for i18n.** `margin-inline-start` over `margin-left`; `padding-block` over `padding-top/bottom`. Right-to-left comes free.
- **CSS variables for theming.** Dark mode, brand swap, user prefs → change one `:root` scope, not every component.
- **Container queries over breakpoint sniffing** when a component should adapt to its container, not the viewport (`@container (min-width: 400px) { … }`).
- **Animate only `transform` and `opacity`.** These run on the compositor — 60fps. Animating `width`, `height`, `top`, `left` triggers layout — jank.
- **Respect `prefers-reduced-motion`.** Wrap non-essential animations in `@media (prefers-reduced-motion: no-preference)`.
- **No inline `style={{...}}` for static styles.** Only for truly dynamic values (positions, computed widths). Everything else via class.
- **Delete dead CSS.** PurgeCSS / Tailwind JIT / build-time dead-code elim. Shipping unused rules is paying twice.
- **One source of truth for z-index.** A named token scale (`z-dropdown`, `z-modal`, `z-toast`). Random `z-index: 9999` → fights.

## Dark mode & theming

CSS variables + a `data-theme` attribute (or `prefers-color-scheme`) is the simplest, framework-agnostic approach.

```css
:root {
  --color-bg: #fff;
  --color-fg: #111;
}
[data-theme="dark"] {
  --color-bg: #0b0d10;
  --color-fg: #e7ebf0;
}
.card { background: var(--color-bg); color: var(--color-fg); }
```

- Semantic token names (`--color-fg-default`, `--color-bg-elevated`) beat palette names (`--gray-900`). The semantic name survives a palette swap.
- Avoid `color-scheme: light dark` auto-switching unless you've designed both themes.

## Composition — utility vs component classes

| Approach | Use for |
|---|---|
| **Utility classes** (Tailwind, atomic) | 90% of styling — spacing, color, flex, grid |
| **Component class** (CSS Module, `.card`) | Reusable visual patterns with internal structure (card, modal, tag) |
| **CSS variable override** | Theming the same component in different contexts |

Don't recreate utility classes in component CSS. Don't recreate component patterns via 20 inline utilities — extract.

## Red flags

- Hex colors or raw `px` values in a feature component.
- `!important` anywhere outside a reset / 3rd-party override.
- `.a .b .c .d { ... }` — specificity creep.
- Styles that work only because of load order.
- A 300 KB CSS bundle with 80% unused rules.
- Two "design systems" in one app — Tailwind + ad-hoc BEM, or CSS Modules + styled-components.
- Animating `height` / `width` / `top` instead of `transform`.
- `z-index: 9999` to win a stacking fight.
- Dark mode done by negating every color in every component instead of swapping variables.
- Media queries hard-coded as pixel values instead of referencing breakpoint tokens.
- Inline `style={{ marginTop: 12 }}` because "it was faster."
