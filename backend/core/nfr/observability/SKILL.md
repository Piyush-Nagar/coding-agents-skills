# Observability

You can't fix what you can't see. Observability is designed-in, not bolted on after an incident. If a question about production needs a code change to answer, instrumentation is wrong.

## Rules

- **Three pillars: logs, metrics, traces.** Each answers a different question — what happened (logs), how much and how often (metrics), why it was slow (traces). You need all three.
- **Structured logs only.** JSON (or similar). Grep is not a strategy at scale.
- **Correlation / trace ID in every log line.** Propagate it through headers across service boundaries. One ID stitches an entire request journey together.
- **RED metrics on every endpoint.** Rate, Errors, Duration (with P50/P95/P99). The minimum needed to spot regressions.
- **USE metrics on every resource.** Utilization, Saturation, Errors — for CPU, memory, pools, queues, disk, network.
- **Trace every cross-service call.** Use OpenTelemetry. Capture DB queries and cache hits as spans too.
- **Alert on SLOs, not on raw metrics.** "CPU > 80%" wakes humans at 3am for nothing. "P99 latency breaches SLO for 5 min" catches real user pain.
- **Health and readiness are different endpoints.** `/health` = "I'm alive, don't restart me." `/ready` = "I can take traffic right now." Treat them as such.
- **Log at the right level, log once per event.** `INFO` for request lifecycle, `WARN` for recoverable issues, `ERROR` for things that need a human. Duplicate logs make dashboards lie.
- **Never log PII or secrets.** Mask at the appender; don't trust callers to remember.
- **Every alert points to a runbook.** An alert without a documented action is a Slack notification pretending to be a page.

## Red flags

- Debugging prod by SSH-ing into a pod.
- Alerts that fire routinely and get ignored. (Alert fatigue is a reliability problem.)
- Logs that say `"error occurred"` with no context, no ID, no stack.
- Metrics that show averages only, with no percentiles.
- A dashboard per service, zero dashboards per user journey.
