# Reliability

Reliability is the discipline of staying up when things around you fail — because they will. Design for partial failure first; happy paths take care of themselves.

## Rules

- **Timeouts on every outbound call.** No exceptions. A missing timeout is a thread-pool-exhaustion incident waiting to happen.
- **Retry only idempotent operations.** With exponential backoff and jitter. Retrying a non-idempotent POST creates duplicates, charges cards twice, sends two emails.
- **Idempotency keys on mutating public APIs.** Lets clients retry safely across network glitches without you having to guess.
- **Circuit-break external dependencies.** When a downstream is sick, failing fast protects both you and it. Blind retries turn one outage into two.
- **Bulkhead critical paths.** Isolate thread pools / connection pools per downstream so a slow one can't drown the whole service.
- **Graceful degradation beats hard failure.** Serve a stale cache, a partial response, or a reduced feature set rather than a 500.
- **No single points of failure.** Multi-AZ for every stateful component, multiple replicas for every stateless one, LB health checks that actually catch sickness.
- **Shutdown must be graceful.** Drain in-flight requests, close pools, flush buffers. SIGTERM → stop accepting → finish or cancel → exit.
- **Make it safe to restart anytime.** Any single pod should be killable at any moment with zero user-visible impact. If not, that's the bug to fix.
- **Backups are untested until you restore from them.** Run a real restore drill on a schedule.

## Red flags

- `restTemplate.exchange(...)` with no timeout configured.
- A retry loop with no cap and no backoff.
- "It works if you redeploy." (Hidden state in memory.)
- Zero-downtime deploys that aren't actually zero-downtime because requests drop on SIGTERM.
- Disaster-recovery runbook that has never been exercised.
