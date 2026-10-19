# Performance (Frontend)

> **Measure first.** Intuition about frontend perf is wrong more often than not.
> The two biggest fixable costs are **bundle size** (first paint) and **re-renders** (interaction cost). Everything else is downstream of these.

## The targets — Core Web Vitals

| Metric | Good | Needs work | Poor |
|---|---|---|---|
| **LCP** — Largest Contentful Paint | ≤ 2.5s | 2.5–4s | > 4s |
| **INP** — Interaction to Next Paint | ≤ 200ms | 200–500ms | > 500ms |
| **CLS** — Cumulative Layout Shift | ≤ 0.1 | 0.1–0.25 | > 0.25 |
| **TTFB** — Time to First Byte | ≤ 800ms | 0.8–1.8s | > 1.8s |

Measure with Lighthouse (lab) **and** `web-vitals` library piping to your observability pipeline (field).

## The flow — fix perf

```
Observe (user complains / RUM alert)
   → measure (Profiler, Lighthouse, Perf tab)
   → classify  — load, interaction, or layout?
   → fix the single biggest cost
   → re-measure (prove the fix moved the number)
```

Never fix "feels slow" — fix a number.

## Rules — load performance (LCP / FCP / bundle)

- **Budget your bundle.** Set a size limit in CI (`size-limit` / `bundlesize`); fail builds that exceed it. Without a budget, bundles only grow.
- **Code-split at route boundaries.** `React.lazy()` + `<Suspense>` per route. Lazy-load heavy modals / editors on demand.
- **Audit dependencies.** `moment.js` → `date-fns`; full `lodash` → cherry-pick (`lodash/debounce`); whole icon library → tree-shaken SVGs.
- **Images:** modern formats (AVIF / WebP), `loading="lazy"` below the fold, explicit `width`/`height` to prevent CLS, responsive `srcset`.
- **Preload the LCP image.** `<link rel="preload" as="image" href=...>` for the hero.
- **Fonts:** `font-display: swap`, preload the above-the-fold weight, limit subsets.
- **Avoid blocking resources in `<head>`.** Defer non-critical JS/CSS.
- **SSR / streaming** if initial content matters for SEO or TTI.

## Rules — interaction performance (INP / re-renders)

- **Profile before optimizing.** React DevTools Profiler shows which components re-render and why. Guessing wastes a day.
- **`useMemo` / `useCallback` are not "free."** They have cost. Apply only to:
  - Values that are referentially-passed to memoized children.
  - Expensive computations (sort, filter 10k items).
  - Dependencies of other hooks (`useEffect`, `useMemo`).
- **Split components.** A heavy parent re-rendering drags its children; splitting isolates the cost.
- **Virtualize long lists** (`react-window`, `react-virtuoso`). > 200 rows → virtualize.
- **Debounce / throttle expensive inputs** (search, filter, slider).
- **Avoid re-rendering the whole app on URL change** — selector-based routing / stable layout.
- **Batch state updates.** React 18 batches by default; still, prefer `setState(prev => …)` for sequential updates.
- **Context carefully.** Every consumer re-renders when the value changes. Split contexts; memoize the value.
- **Avoid inline functions / objects in JSX of memoized children.** New reference every render → defeats the memo.
- **`startTransition` for low-priority updates** (filter typing, tab switch).

## Rules — layout stability (CLS)

- **Reserve space** for images, iframes, ads, embeds (explicit `width`/`height` or aspect-ratio box).
- **Avoid injecting content above existing content** (banners that push the hero down).
- **Font loading:** `font-display: swap` + size-adjust metrics; prevents reflow when the real font loads.
- **Skeletons match final size** — a 20px tall skeleton for 200px content causes CLS when real content lands.

## Red flags

- Wrapping every component in `memo` / every value in `useMemo` "to be safe." (Each adds overhead; most of these save nothing.)
- 500 KB bundle because `moment.js` + `lodash` + a chart lib imported in full.
- `<img>` without width/height, pushing the page as it loads.
- Loading states shorter than the data → "flash of loader" that measures as poor LCP.
- A 10,000-row table rendered without virtualization.
- Context whose value is a fresh object every render (`<Ctx.Provider value={{ user, logout }}>`).
- "Fixed perf" without a before/after number.
- Lighthouse score chased in lab mode while real users (RUM) still report poor INP.

## Measurement toolkit

| Tool | For |
|---|---|
| React DevTools Profiler | Which components re-rendered, how often, why |
| Lighthouse / PageSpeed Insights | Lab Core Web Vitals + bundle analysis |
| `web-vitals` library → analytics | Field CWV from real users |
| Chrome Perf tab (flame chart) | Scripting / layout / paint cost per frame |
| `source-map-explorer` / `vite-bundle-visualizer` | Who owns the bytes in your bundle |
| `size-limit` in CI | Budget enforcement |
