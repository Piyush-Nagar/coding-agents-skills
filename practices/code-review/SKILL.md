# Code Review

Every change gets reviewed before it leaves your machine. **Before `git push`, run a code review on the current diff.** Surface findings, let the user decide fix-or-skip per finding, then push.

## The pre-push loop

1. **Trigger:** before `git push` (or opening a PR), invoke the `/code-review` skill on the current diff.
2. **Report findings** grouped by severity:
   - **Blocker** — correctness bug, security issue, data loss risk, broken contract.
   - **Should-fix** — bad abstraction, missing test, unclear naming, dead code, perf foot-gun.
   - **Nit** — style, minor polish, suggestion.
3. **For each finding, ask the user: fix or skip.**
   - Blockers default to **fix**. Skipping requires an explicit reason.
   - Should-fix defaults to **fix**. Skip is one keystroke.
   - Nits default to **skip** unless the user asks to apply all.
4. **Apply chosen fixes**, re-run the review on the new diff until it comes back clean (or user-skipped).
5. **Only then push.** Never push through unaddressed blockers silently.

> Use `/code-review --fix` to review and apply in one step, or `/simplify` for the "review + fix all" shortcut. For PRs already open, add `--comment` to post findings inline.

## Review techniques (ordered by value)

- **Read the diff twice before commenting.** First pass: understand intent. Second pass: look for defects. Jumping to comments on pass one anchors you on surface issues.
- **Review for correctness first.** Does the code do what it claims? Does it handle the edge cases the author probably forgot — empty input, null, concurrency, failure of the thing it calls? Everything else is secondary.
- **Look at what's missing, not just what's there.** The absent test, the absent timeout, the absent null-check, the absent log line — these are the bugs that reach prod.
- **Trace one real request through the diff.** Pick the happiest-path input and walk the control flow in your head. Then pick the ugliest input and do it again. If you can't, that's a review comment.
- **Check that tests test behavior, not implementation.** A test that mocks the thing it's verifying, or asserts internal method calls, protects nothing.
- **Scope creep is a bug.** A "fix N+1 query" PR with 400 lines of unrelated refactoring is two PRs that should've been split. Say so.
- **Prefer questions over verdicts.** "Why this approach over X?" invites discussion; "Use X instead" invites a fight. The author knows context you don't.
- **Automate what humans shouldn't review.** Formatting, import order, lint rules — CI should block, not a reviewer. Save human attention for intent and correctness.
- **Keep PRs small.** Review quality drops sharply past ~400 LOC. If you're reviewing more than that, ask for a split before you start.
- **Separate blockers from nits in your comments.** A reviewer who sprays 30 same-weight comments buries the one that actually matters.
- **If you don't understand the code in 5 minutes, that's the comment.** "Hard to follow — can we add a comment / split this / rename this?" The code will be read many times; this is the moment to fix that.

## Reviewer's lenses

Run the diff past each lens briefly. Different bugs hide in different lenses.

- **Correctness** — logic, edge cases, error paths, concurrency.
- **Security** — input validation, authz, secrets, injection, PII handling. See [backend/core/nfr/security](../../backend/core/nfr/security/SKILL.md).
- **Operational** — logs, metrics, timeouts, retries, failure modes. See [backend/core/nfr/reliability](../../backend/core/nfr/reliability/SKILL.md) and [observability](../../backend/core/nfr/observability/SKILL.md).
- **Performance** — N+1s, unbounded lists, hot-path allocations. See [backend/core/nfr/performance](../../backend/core/nfr/performance/SKILL.md).
- **Maintainability** — naming, scope, duplication, premature abstraction. See [backend/core/nfr/maintainability](../../backend/core/nfr/maintainability/SKILL.md).
- **API contract** — breaking changes, spec drift, versioning. See [backend/core/spec-driven-development](../../backend/core/spec-driven-development/SKILL.md).

## Rules

- **No push with unaddressed blockers.** Fix or justify skipping, in writing, inside the PR description.
- **Skip is explicit, not silent.** The reason gets recorded — in the PR body, a comment, or a commit message — so the next reader isn't surprised.
- **Review the whole diff, not just the files you touched.** Includes generated files, migrations, config — those are where sharp edges live.
- **Author resolves threads, reviewer re-approves.** A reviewer's comment stays open until the author replies with the resolution.

## Red flags

- Pushing straight to remote without running review on the diff.
- "Address later / follow-up PR" on blockers.
- Nits piling up alongside real bugs with equal visual weight.
- A reviewer who approves everything in under a minute.
- PRs that touch > 500 LOC and claim to be "simple."
