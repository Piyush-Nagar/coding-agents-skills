# Zero-Defect Mindset

> **This is priority #1. Above speed. Above elegance. Above cleverness.**
> Every change must satisfy two conditions, together:
>
> 1. **Zero defects in the new behavior** we are shipping.
> 2. **Zero regressions in the existing behavior** we are not supposed to touch.
>
> One without the other is a failed change.

## The two commitments

| Commitment | What it means |
|---|---|
| **Correct new behavior** | The new feature does exactly what the spec says — happy path, sad path, edge cases, failure modes. |
| **No collateral damage** | Business logic unrelated to this change behaves EXACTLY as before. Not "mostly." Not "I think so." Exactly. |

## The flow — every change

```
Understand new ask
   → map blast radius (what will change, what won't)
   → list edge cases BEFORE coding
   → implement — only what the requirement needs
   → mentally run inputs through the code
   → run tests (existing + new) → both green
   → self-review the diff
   → verify in a running environment
   → only then: "done"
```

## 1. Scope discipline — don't touch what isn't yours to change

The biggest source of production bugs is **not** the new code. It's the "while I'm here, let me also…" refactor that silently alters existing behavior.

| Rule | Why |
|---|---|
| Change only files the requirement forces you to change | Each extra touched file is a new regression surface |
| No "drive-by" refactors in the same PR | Mixes risk; a reviewer can't separate "feature" from "cleanup" |
| If existing code is wrong but not in scope → file a ticket, don't fix it now | Reduces blast radius; keeps PRs reviewable |
| Shared function used by this feature + 10 others → **extend, don't modify** | Modifying the shared function risks the other 10 call sites |
| Rename / move only if the requirement needs it | A rename touches every caller — that's never free |

**Rule of thumb:** if a file in your diff is NOT justified by the ticket, remove it.

## 2. Edge cases — list them BEFORE you write code

Writing the edge-case list first is the single highest-leverage habit for zero defects. Do it on paper, in the PR description, in a comment — anywhere visible.

For every input, ask:

| Dimension | Values to consider |
|---|---|
| Empty / missing | `null`, empty string, empty list, missing field |
| Boundary | `0`, `-1`, `MAX`, exact size, size + 1 |
| Type | Wrong type, wrong format, unexpected shape |
| Volume | 1 item, 1M items, zero items |
| Concurrency | Two requests at the same time, double-click, double-submit |
| Time | Timezone, DST, past dates, future dates |
| State | Entity in state the operation doesn't support |
| Permission | Caller doesn't own the resource / has wrong role |
| Failure | DB down, external service times out, partial failure |

If you can't name what the code does for each → you don't know it works. You hope it works.

## 3. Mental execution — walk real input through your diff

Pick two inputs: the happiest and the ugliest. In your head, trace them line by line through the code. If you can't → it's not reviewable yet.

```java
// Example input to trace:
//   invoiceId = "inv_123", status = APPROVED, approver = "u_7"
// Walk through: controller → service → repo → event → response
// At every step, ask: what if this is null? what if this throws?
```

## 4. Regression safety — prove the untouched stuff still works

| Rule | Why |
|---|---|
| **All existing tests must pass unchanged.** | Red tests you "fixed" by changing the expectation = silent regression |
| **If an existing test needed changing**, write in the PR *why* that assertion was wrong before | Forces honesty — most of the time, the test was right |
| **Add a test for the new path.** No new path ships without one. | The test is the executable spec |
| **Add a test for the edge case you were tempted to skip.** | That's where the bug lives |
| **Touch a shared file → run tests for every module that depends on it.** | Blast radius ≠ just your folder |

## 5. Self-verification — compile ≠ works

A green build is the floor, not the ceiling.

| Step | What it proves |
|---|---|
| Code compiles | Syntax is valid |
| Unit tests pass | The paths you tested behave as you expected |
| Integration tests pass | Pieces fit together |
| **You ran it against a realistic input in a running environment** | It actually does the thing |
| **You tried the ugly input you listed in step 2** | Edge cases don't surprise prod |

If you have not run the new code against a real input → it is unverified. Say that explicitly; don't call it "done."

## 6. The pre-ship checklist

Before you say "done", every box is `✅`:

| ✅ / ❌ | Check |
|---|---|
| ☐ | Every file in the diff is justified by the ticket |
| ☐ | Edge-case list exists and each case is handled (or explicitly out of scope) |
| ☐ | Happy path mentally traced |
| ☐ | Ugly path mentally traced |
| ☐ | All existing tests pass unchanged |
| ☐ | New path has a test |
| ☐ | The edge case I was tempted to skip has a test |
| ☐ | Ran the change in a real environment with real input |
| ☐ | Logs / errors inspected — nothing noisy or missing |
| ☐ | Rollback plan clear (feature flag, config, or revert-safe migration) |
| ☐ | `/code-review` run on the diff (see [practices/code-review](../code-review/SKILL.md)) |

A single `☐` means **not done.**

## Rules

- **Correctness > everything else.** Not speed. Not elegance. Not pattern purity.
- **Blast radius minimization.** If the ticket doesn't need the change, don't change it.
- **No silent assumptions.** Null-check. Type-check. Permission-check. Prove it, don't hope it.
- **Edge cases before code.** List them first — they shape the design.
- **Run it before you trust it.** Compiling and passing CI is a prerequisite, not proof.
- **One PR = one change.** Feature + refactor + rename in one diff is three PRs pretending to be one.
- **If you broke a test, understand WHY the test existed before you touch the assertion.**
- **Reversible by default.** Feature flag, config switch, or revert-safe migration — know how to undo.

## Red flags

- "It should work" / "It probably works" / "I think it works." → it's not verified.
- A PR diff that touches 20 files when the ticket is a 2-field addition.
- Existing test assertions updated with no explanation.
- "Fixed a small thing in X while I was there." — now the PR is two changes.
- No edge case list anywhere — in the PR, in comments, nowhere.
- "I didn't run it, but the tests pass."
- Changes to shared/common code with no check of downstream callers.
- A diff you can't walk through in your head line by line.

## The one-liner

**Change exactly what's asked. Verify it works. Prove the rest is untouched.**
