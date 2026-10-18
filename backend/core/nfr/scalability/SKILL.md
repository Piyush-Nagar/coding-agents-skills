# Scalability

Scalability is the ability to serve 10× the load without rewriting the system. It's a design-time property — bolting it on later is the single most expensive form of tech debt.

## Rules

- **Services must be stateless.** Session, cache, locks, counters — all externalized to a datastore. Any pod should be interchangeable at any moment.
- **Scale horizontally, not vertically.** A bigger box has a ceiling and no redundancy; more boxes have neither limitation.
- **Move spiky or slow work off the request path.** Queues (SQS, Kafka, Rabbit) decouple producer from consumer and absorb bursts without timing out callers.
- **Backpressure, not unbounded queues.** An unbounded queue is a memory leak in disguise. Reject or shed load when saturated.
- **The database is almost always the bottleneck.** Plan read replicas, caching, and sharding keys before you need them, not after.
- **Partition hot data.** Pick a shard key with even distribution and no hotspots. A single celebrity tenant on one shard defeats sharding entirely.
- **Cache aggressively, invalidate deliberately.** Cache-aside is the safe default. Decide write-through vs TTL-only *before* adding it.
- **Design for 10×, load-test for 2×.** The gap buys you time to react; the test keeps you honest.
- **Fan out, join at the top.** Parallelize independent calls — don't call three services sequentially when they could run concurrently.
- **Make hot paths async where possible.** Non-blocking I/O, reactive streams, or event-driven handlers free threads to serve more requests.

## Red flags

- Sticky sessions. (The service isn't actually stateless.)
- A singleton cron that only one pod can run. (The service isn't actually horizontally scalable.)
- Load test that only exercises the happy path at nominal load.
- "We'll shard it later." (Later is during an outage.)
- Queue depth with no alert and no consumer autoscaling.
