# Exception Handling

Exceptions are for *exceptional* conditions. Treat them with the same care as any public contract — vague, swallowed, or over-caught exceptions are how production incidents go undiagnosed for hours. The rule is simple: **be clear about what you catch, why, and what you do next**.

## Rules

- **Fail fast on programmer errors.** Nulls where there shouldn't be nulls, invariants violated, impossible branches — throw immediately and let it bubble. Don't paper over bugs with defensive `try/catch`.
- **Catch at the layer that knows what to do.** A repository doesn't know whether to retry, return a 404, or email the user. Let the exception rise to the service / controller / global handler that can decide.
- **Never catch and swallow.** `catch (Exception e) { }` is a defect, every single time. If you catch, you *either* handle (take action) *or* rethrow — never neither.
- **Log once, at the boundary.** The caller that handles the exception logs it. Logging and rethrowing produces duplicate entries that make dashboards lie.
- **Preserve the cause when wrapping.** `throw new DomainException("…", cause)` — never `throw new DomainException(cause.getMessage())`. You lose the stack trace, you lose the incident.
- **Translate at module boundaries.** `db` throws `DataAccessException`; `capabilities` throws `TestcaseNotFoundException`. Low-level exceptions don't leak into higher layers — callers shouldn't need to import `JpaException` to call a service.
- **Use domain-specific exceptions.** `TestcaseNotFoundException`, `InvalidStateException`, `PaymentDeclinedException`. Not generic `RuntimeException("something went wrong")`. The type carries meaning the catcher can act on.
- **One global HTTP exception handler.** Maps exception types to HTTP status + an error-response DTO (`{code, message, requestId}`). Every endpoint goes through it. Never write per-controller try/catch for the same mapping twice.
- **Never leak internals in error responses.** No stack traces, no SQL, no internal paths. Give the client a stable `code`, a safe message, and a `requestId` to correlate with logs.
- **Expected outcomes are not exceptions.** "User not found" during a lookup is control flow — return `Optional.empty()` / `null` / `Result.notFound()`. Exceptions are for unexpected states, not boolean results.
- **Never catch `Throwable` or `Error`.** `OutOfMemoryError`, `StackOverflowError` — these mean the JVM is unwell. Catching them hides fatal state.
- **Use `try-with-resources` / `finally` for cleanup.** Connections, streams, locks — released on every path, exception or not. Don't rely on GC to close things.
- **Checked exceptions sparingly.** Prefer unchecked (`RuntimeException` subclasses) in modern code. Checked exceptions pollute every signature up the call stack and are rarely acted on.

## The hierarchy we use

```
RuntimeException
└── DomainException                        # base — never thrown directly
    ├── NotFoundException                  # → 404                 (permanent)
    ├── ValidationException                # → 400                 (permanent)
    ├── ConflictException                  # → 409                 (permanent, usually)
    ├── UnauthorizedException              # → 401                 (permanent)
    ├── ForbiddenException                 # → 403                 (permanent)
    └── ExternalServiceException           # → 502 / 503
        ├── TransientException             # retryable  — timeout, 503, connection reset
        └── PermanentException             # not retryable — 400 from downstream, bad request
```

- Each carries an **error code** (stable, machine-readable: `TESTCASE_NOT_FOUND`) and a **safe message**.
- The global handler maps type → HTTP status and renders the response DTO.
- Feature code throws subclasses only; the base `DomainException` is never thrown directly.
- `TransientException` is a **marker type** — retry/backoff logic keys off it, so a caller doesn't have to introspect messages or status codes.

## Transient vs permanent errors

An exception is **transient** if the same call *could succeed later* without any change to inputs. Everything else is **permanent**. Classifying this up front is what makes the rest of the system — retries, circuit breakers, dead-letter queues — actually safe.

**Transient (retry with backoff):**
- Network: connection reset, DNS failure, socket timeout.
- Downstream `503 Service Unavailable`, `504 Gateway Timeout`, `429 Too Many Requests`.
- DB: lock-wait timeout, serialization failure, deadlock, connection pool exhaustion.
- Cloud/infra: throttling (`ProvisionedThroughputExceeded`, `ThrottlingException`).

**Permanent (do NOT retry):**
- `4xx` from downstream (except `408`, `425`, `429`) — bad request, auth, not found.
- Validation failures, business-rule violations, duplicate-key errors.
- `NullPointerException`, `IllegalStateException`, parse errors — programmer bugs. Fix the code, don't retry.

### How to handle each

- **Classify at the lowest layer that knows.** The HTTP/DB client wraps the raw error into `TransientException` or `PermanentException` once. Nobody above re-classifies.
- **Retry only `TransientException`, and only for idempotent operations.** Exponential backoff + jitter, capped (e.g. 3–5 attempts, max 10s total). See [reliability](../nfr/reliability/SKILL.md).
- **Non-idempotent mutations need an idempotency key** before you're allowed to retry. Without one, a retried `POST /payments` double-charges.
- **Circuit-break on sustained transient failure.** When the error rate stays high, open the breaker and fail fast instead of hammering a sick dependency.
- **Permanent errors return to the caller.** No retry, no backoff. Map to the right HTTP status and move on.
- **Log transient at `WARN`, permanent at the appropriate level** (`INFO` for 4xx caller errors, `ERROR` for 5xx and bugs). A loud `ERROR` on every transient blip is alert fatigue in the making.
- **Preserve attempt count in logs/traces.** `attempt=3/5` tells the on-call that retries happened and ran out. Silent retries hide the real latency story.
- **Dead-letter async messages after the retry budget.** For queue consumers, a message that keeps throwing `TransientException` goes to DLQ with its context, not to an infinite retry loop.

### Red flags specific to transient handling

- Catching an exception, retrying unconditionally, no backoff, no cap.
- Retrying a `ValidationException` or a `404`. (It will fail every time.)
- Retrying a non-idempotent `POST` without an idempotency key.
- The same transient error logged at `ERROR` on every retry attempt.
- Code that inspects `e.getMessage().contains("timeout")` to decide whether to retry.

## Red flags

- `catch (Exception e) { log.error("error", e); }` with no rethrow and no recovery.
- `throw new RuntimeException("failed")` with no cause and no type.
- The same try/catch block copy-pasted into every controller.
- An error response that contains `com.example.jpa.…` or `SELECT * FROM …`.
- A method signature declaring `throws Exception` — the "I give up" of error handling.
- Catching an exception just to log and rethrow it unchanged.
- Using exceptions to implement normal control flow (e.g. parsing a number by catching `NumberFormatException` in a hot loop).
