# Design System

Most companies have an internal UI library — typically a Storybook-backed package of buttons, inputs, modals, tables, and layout primitives. **Use it. First. Always.** The design system is not a suggestion; it is the baseline.

## The default

Before writing any UI code, check the design system for an existing component. If it exists, import it. If it exists but doesn't quite fit, extend it inside the design system — not around it in your feature code.

Order of preference:

1. **Use the component as-is.** Shortest path, zero drift.
2. **Compose design-system primitives.** Build the feature piece from `Button`, `Input`, `Stack`, `Modal`, etc.
3. **Request a change in the design system.** A new variant, a new prop, a new primitive. Lands once, used everywhere.
4. **Local component — only as last resort.** Must be feature-specific, not reusable. Document why the design system couldn't be used.

Steps 1-3 should cover ~95% of UI work. Step 4 should feel uncomfortable.

## Rules

- **Import from the design system, not from `@mui/*` / `antd` / raw primitives.** The design system wraps those for a reason — theming, accessibility, telemetry. Bypassing it fractures the UI.
- **No parallel component libraries.** One `Button`, one `Modal`, one `Table` across the app. If you need a different one, that's a design-system change, not a local one.
- **Theme tokens over hard-coded values.** Colors, spacing, radii, typography come from theme tokens. `#3366ff` and `padding: 12px` are code smells.
- **No one-off styles that duplicate existing variants.** If your "custom" button looks like the design system's `primary` button, it IS the primary button — use it.
- **Accessibility is the design system's job, not yours.** Trust its focus states, ARIA, keyboard handling. Overriding them silently breaks a11y.
- **Icons from the design-system icon set.** Not pasted SVGs, not a second icon library.
- **Report gaps, don't route around them.** Missing variant? Open a PR or issue against the design system. Workarounds compound.

## Working with the design system

- **Read the Storybook before building.** It's the catalog. Skipping it is why we end up with three different date pickers.
- **Match the component's intended usage.** A `Modal` is not a container for a page. A `Table` is not a layout grid. Misuse creates visual bugs and a11y bugs.
- **Prefer composition over configuration.** Ten props is a smell. If the design-system component needs a boolean maze to fit your case, it's probably the wrong component.
- **Keep feature code thin.** Feature components arrange design-system components and wire them to data. They don't restyle them.

## When to extend the design system

Add to it when the pattern will be used in **more than one place** or implements **a cross-cutting concern** (empty states, error boundaries, loading skeletons, form field layouts). One-off marketing art does not belong in the design system.

## Red flags

- A feature folder with its own `Button.tsx`, `Input.tsx`, or `Modal.tsx`.
- Styling overrides targeting design-system class names (`.ds-button { ... }`) from feature CSS.
- Hex colors or raw `px` values in feature components.
- Two components in the same app that look 90% the same but share no code.
- "The design system doesn't have X" — followed by a local build instead of a design-system contribution.
