# SOLID Principles

**Default: every class and every piece of business logic follows SOLID.** It is the baseline for how we structure code. The principles are not academic — each one prevents a specific, common failure mode we've all paid for.

That said, SOLID is a tool, not a religion. The goal is maintainable code, not maximum abstractions. See [Pragmatic flexibility](#pragmatic-flexibility) for when to relax.

## The five, in one line each

- **S — Single Responsibility.** A class has one reason to change. If "pricing changed" and "tax rules changed" both touch the same class, split it.
- **O — Open/Closed.** Open for extension (add behavior via new types/strategies), closed for modification (don't edit stable code to add a case). Prevents a growing `if/else` on types.
- **L — Liskov Substitution.** A subtype must be usable wherever the supertype is, without the caller knowing. If overriding changes contracts (throws, nullability, side effects), it's a lie.
- **I — Interface Segregation.** Clients depend only on the methods they use. Many small, focused interfaces beat one 20-method god interface.
- **D — Dependency Inversion.** High-level code depends on abstractions, not concrete classes. Compose, don't `new`. Enables testing, swapping, and clear module boundaries.

## How this shows up in our code

- **Services are single-purpose.** One service per API — `CreateTestcaseService`, `GetTestcaseByIdService`. If a service does three unrelated things, split it. (See [backend/core](../SKILL.md).)
- **Depend on `*Repo` and `*Client` interfaces, never on `*RepoImpl` / `MongoTemplate` / `JdbcTemplate` directly.** That is Dependency Inversion made concrete.
- **New behavior = new type, not a flag.** A new payment method is a new `PaymentStrategy`, not an `else if (type == "upi")`. Open/Closed.
- **Interfaces describe one role.** A `TestcaseRepo` reads and writes testcases. A `TestcaseExporter` exports. They are not the same interface.
- **Subclasses honor the parent contract.** No throwing from a method the parent says can't throw. No silently returning `null` where the parent returns a value.

## Pragmatic flexibility

SOLID is a floor, not a ceiling. Over-applied, it produces three-layer indirection for a function that will have exactly one implementation forever. These are the sanctioned relaxations:

- **One implementation, no planned second? Skip the interface for now.** Add it when the second implementation (or the test mock, or the module boundary) actually arrives. YAGNI beats speculative abstraction.
- **DTOs, value objects, and request/response records don't need SOLID gymnastics.** They are data. A record with 10 fields is fine — it has one responsibility: carry data.
- **Small helper classes with two tightly-coupled methods don't need to be split** just to hit SRP. "One reason to change" is a judgment call, not a line count.
- **Prefer composition and plain functions over deep class hierarchies.** If you're reaching for Liskov to justify a four-level inheritance tree, step back — composition is usually clearer.
- **Framework-mandated shapes get a pass.** Spring `@Configuration` classes, JPA entities, controllers — their structure is dictated by the framework. Don't fight it.

**Rule of thumb for abstraction:** add it on the *second* occurrence, not the first. One is a case; two is a pattern; three means the abstraction is overdue.

## Red flags

- A single service class with `create`, `update`, `delete`, `export`, `notify`, and `report` methods. (SRP.)
- A `switch` or `if/else` on a type enum that grows every quarter. (OCP.)
- A subclass that overrides a method just to throw `UnsupportedOperationException`. (LSP.)
- An interface with 15 methods, where no caller uses more than 3. (ISP.)
- A service that `new`s its collaborators inside a method, making it un-mockable. (DIP.)

## Red flags of over-application

- An interface + impl pair for a class that will never have a second implementation.
- A `Factory` for an object with one line of construction.
- Three layers of indirection to call one method.
- Interface names like `IThing` paired with `ThingImpl` — a strong sign the interface was created for its own sake, not for a boundary.
