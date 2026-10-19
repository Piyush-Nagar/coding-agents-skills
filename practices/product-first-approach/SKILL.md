# Product-First Approach

> **Be a Product Manager before you are an engineer.**
> We are not writing random code files. We are building a product.
> Open the problem before you open the IDE.

## The mindset shift

| Engineer-first (wrong default) | Product-first (right default) |
|---|---|
| "What do I build?" | "What problem are we solving, for whom?" |
| Accepts the ticket as the truth | Treats the ticket as a hypothesis |
| Starts with files, folders, frameworks | Starts with users, outcomes, metrics |
| Names things after tech (`DataProcessor`) | Names things after the domain (`InvoiceApprover`) |
| Silence on ambiguity → guess | Ambiguity → **ask** |
| Done = merged | Done = outcome moved |

## The flow — every requirement

```
Requirement lands
   → read as a PM, not an engineer
   → identify gaps / ambiguity
   → gaps exist?  → ASK. do not code.
   → no gaps?     → model the domain  → design in product language  → build the least thing  → measure
```

## Ask-for-clarification triggers

If any of these are missing or vague, **stop and ask**. Writing code on a bad spec is the most expensive mistake we make.

| Gap | What to ask |
|---|---|
| Who is the user? | "Which role / persona triggers this?" |
| What outcome? | "What should be true after this ships that isn't true now?" |
| Success metric | "How will we know it worked? What number moves?" |
| Scope boundary | "What's explicitly out of scope?" |
| Edge cases | "What happens when X is empty / invalid / duplicated?" |
| Permissions | "Who can / cannot do this?" |
| Failure behavior | "What should the user see when this fails?" |
| Priority | "If we have to cut, what's the first thing to drop?" |

**Rule:** if the ticket could be interpreted two sensible ways by two engineers, that's an ambiguity. Ask.

## PM-style requirement breakdown

A requirement is not a task list. It's a product change. Decompose it in product terms **before** touching code:

1. **User story** — "As a *<persona>*, I want *<outcome>*, so that *<value>*."
2. **Acceptance criteria** — behavior the user can observe, not implementation steps.
3. **Metric** — the one number this is supposed to move.
4. **Non-goals** — the things this is *not* trying to do (prevents scope creep).
5. **Open questions** — anything you asked about in the clarification step; keep them visible until resolved.
6. **Smallest shippable slice** — the minimum version that still produces the outcome.

Only after this do you break it into engineering tasks.

## Product-oriented naming — code reads like the product

Code names ARE product language. If a non-engineer PM can read the code and understand what it does, the names are right.

### Rules

| Element | Named after | Not named after |
|---|---|---|
| Class | Domain concept, user action, product noun | Pattern (`XxxManager`, `XxxHelper`, `XxxUtil`) |
| Method / function | What the user achieves | Internal mechanics (`processData`, `handleRequest`) |
| API endpoint | Resource + user intent | Verbs on your DB layer |
| DTO / request / response | The real-world thing it represents | Tech shape (`Payload`, `Dto1`) |
| Event / message | Domain event in past tense | Technical trigger |
| Error / exception | What went wrong *for the user* | Where in the stack it was raised |

### Examples

| Bad (tech-first) | Good (product-first) |
|---|---|
| `class DataProcessor` | `class InvoiceApprover` |
| `class UserManager` | `class CustomerOnboarding` |
| `fun processRequest()` | `fun submitPayment()` |
| `fun handleEvent()` | `fun markOrderDelivered()` |
| `POST /api/data` | `POST /api/invoices/{id}/approve` |
| `GET /api/getUserDataById` | `GET /api/customers/{id}` |
| `PaymentDto1`, `PaymentDto2` | `PaymentRequest`, `PaymentReceipt` |
| `OrderUpdatedEvent` (vague) | `OrderShippedEvent`, `OrderCancelledEvent` |
| `GenericException("invalid")` | `InvoiceAlreadyApprovedException` |

### A Java example

```java
// Bad — reads like plumbing
class OrderService {
    void processOrder(OrderDto dto) { ... }
    void updateStatus(Long id, String s) { ... }
}

// Good — reads like the product
class OrderCheckout {
    OrderReceipt placeOrder(PlaceOrderRequest req) { ... }
    void markShipped(OrderId id) { ... }
    void markCancelled(OrderId id, CancellationReason reason) { ... }
}
```

A reviewer can tell what *the user is doing* without reading a single method body.

## Rules

- **Think PM first, engineer second.** For every requirement, write the user story and metric before the first file.
- **Ambiguity → ask.** Never ship your guess as if it were the spec.
- **Done = outcome moved**, not code shipped.
- **Ship the ugliest version that proves the hypothesis.** Then measure. Then iterate — or kill.
- **"No" is a feature.** Every yes is a no to the next thing.
- **Tech serves the product, never leads it.** Pick what the product needs — not what you want to learn.
- **Names come from the domain.** If the PM wouldn't recognize the word, pick a different word.
- **Launch is day 1, not day 90.** Metric, dashboard, and kill criterion ship with the feature.

## Build vs. don't-build

| Signal | Build? |
|---|---|
| Real user asked, pain is understood | ✅ Yes |
| Metric clearly moves if we build it | ✅ Yes |
| Nice-to-have, no metric owner | ❌ No |
| "Our competitor has it" (only reason) | ❌ No |
| "We'll find a use case later" | ❌ No |
| Pure engineering preference | ❌ No |

## Scope-cutting playbook

Pressure says "build more" → cut the opposite way.

- **Fewer users** — one segment fully, ignore the rest.
- **Fewer states** — happy path only; errors phase 2.
- **Fewer configurations** — one default, no settings.
- **Manual before automated** — spreadsheet first, pipeline later.
- **Wizard-of-Oz** — human simulates the automation to prove demand.
- **Fake door** — ship the button, measure clicks, then build what's behind it.

## Red flags

- Opened the IDE before writing the user story.
- Ticket interpreted and shipped without a single clarifying question.
- Class/method/API names a PM wouldn't recognize.
- A spec that lists **features** but no **outcome metric**.
- "Phase 2 will handle it" — phase 2 never ships.
- Rebuilding before measuring the current version.
- Launches without a kill criterion.
- Design/code review focuses on tech patterns, not user outcomes.

## The one-liner

**Think like a PM. Name like the domain. Build the least thing. Measure. Keep what works. Kill what doesn't.**
