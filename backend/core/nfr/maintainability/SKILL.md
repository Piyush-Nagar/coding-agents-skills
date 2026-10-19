# Maintainability

Code is read far more often than it is written. Maintainability is what separates a 3-year-old codebase that's a joy from one that's a tar pit. Optimize for the next developer, who may be you in six months.

## Rules

- **One feature, one place.** Package by feature (see [backend/core/packaging](../../packaging/SKILL.md)). A single change should touch adjacent files, not scattered layers.
- **Interfaces at module boundaries only.** Inside a feature, concrete classes are fine. Interfaces exist to let callers depend on contracts across boundaries, not to decorate every class.
- **Delete, don't comment out.** Dead code and `// TODO`-forever comments rot. Git remembers; the IDE doesn't need to.
- **Prefer small, pure functions.** Easier to test, easier to reason about, easier to compose. Side effects at the edges of the system, not scattered through it.
- **Tests cover behavior, not implementation.** Rewriting internals shouldn't break tests. If it does, the test is coupled to the wrong thing.
- **Public APIs are versioned.** Breaking changes behind a new version or a feature flag — never a silent rename.
- **Dependency injection over static singletons.** Statics are untestable and hide coupling. Pass dependencies in; mock them out in tests.
- **Pin and regularly update dependencies.** Lockfiles committed; Renovate/Dependabot auto-PRing. Falling 2 years behind means an unplanned 2-week migration.
- **Document the "why", not the "what".** Code shows what; comments explain the business rule, the historical constraint, the non-obvious trade-off.
- **Lint, format, and type-check in CI.** Not advisory, blocking. Humans should debate design, not spacing.
- **Keep the build green and fast.** Flaky or slow tests get ignored, and once ignored, they protect nothing.

## Red flags

- A single file over ~500 lines, or a class with over ~10 public methods — usually a feature trying to escape.
- Shared "utils" package where orphaned helpers go to die.
- Mocking the thing under test to make a test pass.
- "Don't touch that module, nobody understands it anymore."
- CI build time measured in tens of minutes.
