# Component API Design

> **Design the component you'd want to use.**
> Composition over configuration. A good API is small, predictable, and gets out of your way. A bad API is 20 props and a Confluence page.

## The default: composition > configuration

A component with 15 flags is a design problem.

```tsx
// Bad — configuration-heavy, inflexible, grows forever
<Card
  title="Invoice"
  subtitle="Due in 3 days"
  showIcon
  iconName="receipt"
  showAction
  actionLabel="Pay"
  onAction={pay}
  footerText="Processed by Stripe"
/>

// Good — composition, flexible, small surface
<Card>
  <Card.Header icon={<ReceiptIcon />} title="Invoice" subtitle="Due in 3 days" />
  <Card.Body>…</Card.Body>
  <Card.Footer>
    <Button onClick={pay}>Pay</Button>
    <Text size="sm">Processed by Stripe</Text>
  </Card.Footer>
</Card>
```

If your prop list grew past ~5–7, step back and reach for composition.

## Rules

- **Composition first.** `children`, slots, compound components beat booleans and strings. Consumers compose; you provide structure.
- **One job per component.** If you can't name it in a product noun (`InvoiceRow`, `PriceTag`), it's doing too much.
- **Minimal prop surface.** Start with the smallest set. Add a prop only when a real caller needs it — not when you imagine someone might.
- **Controlled vs uncontrolled.** Default to **uncontrolled** with `defaultValue`; allow **controlled** via `value` + `onChange`. Don't make callers manage state they didn't want to.
- **Forward refs.** `React.forwardRef` on anything that could sensibly accept one (inputs, buttons, containers). Needed for focus, measurement, animation libs.
- **Spread the rest.** Spread unknown props to the right underlying element (`<button {...rest}>`). Lets callers add `aria-*`, `data-*`, `onBlur`, etc. without API churn.
- **Standard event names.** `onClick`, `onChange`, `onSubmit` — never `onClicked`, `handleClick`, `clickHandler`.
- **Boolean props: name for the ON state.** `isDisabled`, `isLoading`, `hasError` — not `enabled`, `active`, `error`.
- **Avoid mutually exclusive booleans. Use a variant.** `variant="primary" | "secondary" | "danger"` beats `isPrimary`, `isSecondary`, `isDanger`.
- **Props as data, children as content.** `title` is a prop; freely composed content is `children`.
- **No required callback with no default handler.** If `onX` is required for the component to function, document it; or provide a safe default.
- **Polymorphic `as` prop for low-level primitives.** `<Button as="a" href="...">` so a design-system `Button` can also be a link without duplicating styles.
- **Headless + styled layers.** Behavior (keyboard, a11y) lives in a headless primitive (Radix / React Aria). Visuals live on top. Keeps a11y correct by construction.
- **Document with Storybook.** Every variant, every state (default, hover, focus, disabled, loading, error). Stories are the contract.

## Patterns worth knowing

| Pattern | When to use |
|---|---|
| **Compound components** (`Tabs.Root / Tabs.List / Tabs.Trigger / Tabs.Content`) | Related parts that share implicit state |
| **Render props / function children** | Caller needs access to internal state (list item index, open flag) |
| **Slots** (named children props) | Multiple "regions" that aren't a simple list |
| **Polymorphic `as` prop** | Primitive components (Button, Box, Text) that must adapt their element |
| **Controlled + uncontrolled** | Any form-like component |
| **Headless primitive + styled wrapper** | Complex behavior where a11y + keyboard are non-trivial |

## Controlled + uncontrolled — the canonical pattern

```tsx
type ToggleProps = {
  value?: boolean;             // controlled
  defaultValue?: boolean;      // uncontrolled
  onChange?: (v: boolean) => void;
};

export function Toggle({ value, defaultValue = false, onChange }: ToggleProps) {
  const [internal, setInternal] = React.useState(defaultValue);
  const isControlled = value !== undefined;
  const current = isControlled ? value : internal;

  const set = (next: boolean) => {
    if (!isControlled) setInternal(next);
    onChange?.(next);
  };

  return <button aria-pressed={current} onClick={() => set(!current)}>…</button>;
}
```

## Rules of thumb — stop and rethink when…

- Prop count hits 8+ → consider composition.
- Two props must always be set together → merge them into one object or a variant.
- Two props can never be set together → that's a variant, not two booleans.
- The component takes a `renderXxx` prop AND a `xxx` prop → pick one.
- You added `extra`, `extra2`, `custom` escape hatches → the API is wrong.

## Red flags

- 15+ props, mostly booleans.
- `isPrimary`, `isSecondary`, `isDanger` instead of `variant`.
- No `ref` forwarding on an input / button / container.
- Props named after visuals (`biggerText`, `moreMargin`) instead of intent.
- Non-standard event names (`onClicked`, `clickHandler`).
- Component "owns" state that the caller actually wants to control, with no `value` escape hatch.
- Storybook story for only the happy path — no loading, no error, no disabled.
- Styles leaked via `className` + inline overrides because the API didn't offer the hook properly.
- `<Modal fullScreen bottomSheet confirm destructive />` — this is four components wearing a trenchcoat.
