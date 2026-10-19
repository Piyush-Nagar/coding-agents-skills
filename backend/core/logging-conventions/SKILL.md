# Logging Conventions

> Logs are read in a hurry, during an incident, by someone who did not write the code.
> **Every log line = a readable sentence + the identifiers to correlate it + just enough context — nothing more, nothing less.**

Companion to [observability](../nfr/observability/SKILL.md). This file is the "how to write a log line" rulebook.

## The structure of a good log line

```
<level> <timestamp> <correlationId> <logger>  <readable message>  k=v  k=v  k=v
```

Three non-negotiables:

| Required | Why |
|---|---|
| **Correlation / trace ID** (always) | Stitches logs across services and threads. Without it, logs are useless at scale. |
| **Readable English message** | A human must understand the line without reading the code. |
| **Structured fields** (JSON) | Dashboards, queries, alerts all key off fields — not substrings of messages. |

## Log levels — when to use each

| Level | Use for | How often |
|---|---|---|
| `ERROR` | Unexpected failure that needs human attention. On-call might page. | Rare |
| `WARN` | Recoverable issue, degraded path, something to notice but not fix now | Occasional |
| `INFO` | Request lifecycle, state transitions, significant business events | Steady, bounded |
| `DEBUG` | Developer-only detail for diagnosing; off in prod | Off in prod |
| `TRACE` | Hyper-granular; usually off even in dev | Off |

**Rule of thumb — the "not too much, not too little" test:**

- `INFO` on every request = fine. `INFO` inside a tight loop = **noise**.
- `ERROR` on every retry = alert fatigue. One `ERROR` after the retry budget is used = right.
- Zero `INFO` in a service = flying blind.

## Message writing rules

| Do | Don't |
|---|---|
| Verb first, past tense: `Approved invoice` | `invoice approval processing completed successfully` |
| State the fact, not the mechanism: `Order shipped` | `OrderShippedHandler.handle() finished` |
| Add the identifier(s) as fields, not in the message string | `Order 123 for user 45 shipped at 2024-...` all crammed in the text |
| One sentence, under ~80 chars | Multi-line stories |
| Business terms | Framework / layer jargon (`DAO`, `Mapper`, `Impl`) |

### Example

```java
// Bad — unreadable, no identifier, no fields
log.info("processing done successfully in service layer");

// Good — readable + fields with identifiers
log.info("Approved invoice",
    kv("invoiceId", invoice.getId()),
    kv("tenantId", invoice.getTenantId()),
    kv("amount", invoice.getAmount()));
```

## Identifiers — always present

Every log line in a request path **must** carry:

| Field | Source |
|---|---|
| `traceId` / `correlationId` | MDC — set once at the edge (filter / interceptor), propagated downstream |
| `tenantId` (if multi-tenant) | Authenticated context |
| **Primary business identifier** of the operation | e.g., `orderId`, `invoiceId`, `userId` — whichever the operation is about |

Set via MDC so they appear automatically; don't rely on authors remembering to add them per line.

```java
// At the request edge:
MDC.put("traceId", traceId);
MDC.put("tenantId", tenantId);
try { chain.doFilter(req, res); } finally { MDC.clear(); }
```

## What to log (minimum)

Log a line at every point where state meaningfully changes:

- **Request received** (edge) — `method`, `path`, `userId`, `tenantId`
- **Request completed** — `status`, `durationMs`, `bytesOut`
- **Business event** — one line per domain event (`Order placed`, `Payment captured`, `Subscription cancelled`)
- **External call made** — `service`, `endpoint`, `durationMs`, `status` (don't log request/response bodies)
- **Decision taken** that would surprise a reader (feature flag toggled on, retry triggered, cache miss on a hot key)

Everything else: ask "would this help a 3am debugger?" If no → don't log.

## What to NEVER log

- Passwords, tokens, auth headers, session cookies.
- API keys, JWTs, secrets.
- Full card numbers, CVV, bank account numbers.
- Full email, phone, address — mask (`u***@example.com`) if you must.
- Request/response bodies that may contain PII.

Enforce masking **at the appender**, not at every call site. Don't trust authors to remember.

## Error logging — the strict rules

Errors are the logs that matter most during incidents. Treat them with care.

| Rule | Why |
|---|---|
| **Log the full stack trace** — pass the exception as the last arg, don't `e.getMessage()` | `getMessage()` loses the entire cause chain |
| **Preserve the cause chain** when wrapping — `throw new DomainException("…", cause)` | Lets the final log line walk back to the root cause |
| **Include the business identifier(s)** at log time | So you can find THIS error in a sea of errors |
| **Log once, at the boundary** that handles the exception | Logging and rethrowing produces duplicates and lies to dashboards |
| **One line per error** — the stack trace is attached to that line | Don't `log.error("A"); log.error(e.getMessage()); log.error("B");` |
| **Add a stable error code** when known | `code=INVOICE_ALREADY_APPROVED` is queryable, English isn't |

### Error log — the correct shape

```java
// Bad — loses stack trace, no identifier
catch (Exception e) {
    log.error("something went wrong: " + e.getMessage());
}

// Good — identifier + full stack trace + error code
catch (DomainException e) {
    log.error("Invoice approval failed",
        kv("invoiceId", id),
        kv("tenantId", tenantId),
        kv("code", e.getCode()),
        e);                               // <- exception LAST, prints full stack
}
```

See [exception-handling](../exception-handling/SKILL.md) for how exceptions should flow before they reach the log.

## Rules

- **Structured JSON logs only.** Humans grep in dev; machines query in prod — only JSON serves both.
- **One line per event.** Don't split one event across three lines.
- **Correlation ID on every single line.** Via MDC. No exceptions.
- **Business identifier on every meaningful line.** `orderId`, `invoiceId`, `userId` — whichever is the primary noun of the operation.
- **Fields for data, message for meaning.** Numbers, IDs, durations go in fields. The message describes the event.
- **Level discipline.** Don't log an exception at `INFO`. Don't log a happy path at `WARN`. Mis-levelled logs destroy dashboards.
- **Dynamic log levels in prod.** Config change raises `DEBUG` on one package without a redeploy.
- **Rate-limit noisy loggers.** A loop that logs 10k times per second is a reliability bug, not a logging bug — fix the loop; also add a per-logger rate limit as a safety net.
- **Delete dead logs.** `log.info("here")` and friends get removed before merge.

## Red flags

- `log.error("error", e.getMessage())` — stack trace lost.
- `log.info("entering method X")` / `log.info("exiting method X")` — tracing isn't logging.
- Log messages that are only readable if you know which file emitted them.
- Any line in a request path with no `traceId`.
- The same exception logged 3 times as it bubbles up.
- Secrets, tokens, or raw request bodies in logs.
- An `ERROR` with no business identifier — un-findable in prod.
- 50 lines of `DEBUG` left in prod code.
