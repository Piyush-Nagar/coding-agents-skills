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
    ├── NotFoundException                  # → 404
    ├── ValidationException                # → 400
    ├── ConflictException                  # → 409 (duplicate, version conflict)
    ├── UnauthorizedException              # → 401
    ├── ForbiddenException                 # → 403
    └── ExternalServiceException           # → 502 / 503
```

- Each carries an **error code** (stable, machine-readable: `TESTCASE_NOT_FOUND`) and a **safe message**.
- The global handler maps type → HTTP status and renders the response DTO.
- Feature code throws subclasses only; the base `DomainException` is never thrown directly.

## Red flags

- `catch (Exception e) { log.error("error", e); }` with no rethrow and no recovery.
- `throw new RuntimeException("failed")` with no cause and no type.
- The same try/catch block copy-pasted into every controller.
- An error response that contains `com.example.jpa.…` or `SELECT * FROM …`.
- A method signature declaring `throws Exception` — the "I give up" of error handling.
- Catching an exception just to log and rethrow it unchanged.
- Using exceptions to implement normal control flow (e.g. parsing a number by catching `NumberFormatException` in a hot loop).
