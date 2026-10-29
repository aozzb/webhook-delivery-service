# Interview questions & answers — defend this project

50 questions covering the webhook delivery service in depth, each with a short,
plain-English answer. They lean on "what happens when…" scenarios, because that's
where interviewers probe. Defaults referenced: 6 attempts, 8s POST timeout,
backoff base 1s capped at 5min, breaker trips at 5 consecutive failures / 30s
cooldown, sweeper every 10s, signature tolerance 300s.

## A. Architecture & data flow

**1. Walk through the lifecycle of an event from `POST /events` to a delivered webhook.**
The API authenticates the key, validates the body (zod), inserts the **event** in
Postgres, finds the application's active endpoints **subscribed** to that event
type ("fan-out"), creates one **delivery** row per endpoint, and enqueues a
BullMQ job per new delivery. The **worker** picks up each job, loads the fresh
rows from Postgres, signs the request, POSTs it to the endpoint with a timeout,
records the attempt, and marks the delivery succeeded/failed/retrying/dead. The
**scheduler** sweeps in the background to re-enqueue anything that slipped through.

**2. Why separate api / worker / scheduler processes?**
Separation of concerns and independent scaling. The **api** is latency-sensitive
and just accepts + enqueues; the **worker** does the slow, failure-prone outbound
HTTP and can be scaled to many replicas; the **scheduler** is a single instance
that reconciles. A crash or overload in one doesn't take down the others.

**3. Why is Postgres the source of truth and Redis disposable?**
Postgres durably holds the real state — events, deliveries, attempts, statuses.
Redis/BullMQ only holds the *work queue*, which is derived from Postgres and can
be rebuilt from it. So if Redis is lost, we lose no data, only transient queue
state, which the sweeper reconstructs.

**4. What are the delivery semantics and why?**
**At-least-once.** Across a network you can't guarantee exactly-once (you can
always crash between "receiver got it" and "we recorded it"). We prevent
*duplicate rows/jobs* with a unique constraint and queue dedupe, and we send a
stable `X-Webhook-Id` so receivers can dedupe on their side. Receivers must be
idempotent — that's the contract.

**5. The single most important design principle?**
Postgres is the source of truth and everything else is recoverable from it. That
one rule drives the unique constraints, the sweeper, and treating Redis as a
cache of work rather than a system of record.

## B. Schema & data model

**6. The five tables and their relationships.**
`applications` (tenant, holds the hashed API key) → has many `endpoints` (url,
HMAC secret, subscribed event types) and `events` (type, jsonb payload,
idempotency key). An event + endpoint produce a `delivery` (status, attempts,
next_attempt_at). Each delivery has many `delivery_attempts` (append-only log of
each POST). Foreign keys cascade on delete.

**7. Why `UNIQUE(event_id, endpoint_id)` on deliveries?**
It's the backbone of "at-least-once without duplicate deliveries": the database
physically allows only one delivery obligation per (event, endpoint) pair. Even
if two processes or a retried enqueue race, you can't create two. Without it,
races and retries would create duplicate deliveries that each get sent.

**8. Why scope `idempotency_key` to `(application_id, idempotency_key)`?**
Keys are per-customer. If it were globally unique, two different customers both
using `"order-1"` would collide — one customer's event would be silently treated
as a duplicate of another's. The composite unique fixes that; NULLs stay distinct
so keyless events are never deduped.

**9. Why UUIDs for most tables but `bigserial` for `delivery_attempts`?**
UUIDs are used where the id is exposed to the outside world (event ids ride in
headers/logs) — they're non-enumerable so one customer can't guess another's.
`delivery_attempts` is a high-volume internal log that's never exposed, so a cheap
monotonic `bigserial` is a better fit.

**10. Why the partial index `(next_attempt_at) WHERE status='pending'`?**
The worker/sweeper constantly ask "which deliveries are due now?" A partial index
only indexes rows still in flight (`pending`), so it stays tiny even as millions
of `succeeded`/`dead` rows pile up — a plain index would keep growing and slow
those lookups.

## C. Publishing & idempotency

**11. What happens when the same event is published 100× with one idempotency key?**
The first publish inserts the event; the other 99 find the existing row (or hit
the unique constraint) and return the *same* event id with `idempotent_replay:
true` — no new event, no new deliveries. The duplicate is stopped at the
`UNIQUE(application_id, idempotency_key)` constraint on `events`.

**12. Two requests with the same key in the same millisecond — the race?**
Both may pass the initial "does it exist?" check and try to insert. The unique
constraint lets exactly one win; the loser gets a `23505` unique-violation, which
the code catches and turns into the same idempotent-replay response. The database
is the arbiter, so the race is safe.

**13. What is fan-out, and what decides the target endpoints?**
Fan-out is turning one event into one delivery per interested endpoint. The API
selects the application's **active** endpoints whose `event_types` array contains
the event's type (`event_types @> ARRAY[type]`, backed by a GIN index).

**14. Event type with no subscribed endpoint?**
The event is still stored, but fan-out finds zero endpoints, so **zero
deliveries** are created and the response says `deliveries_created: 0`. Nothing is
sent — correct behavior.

## D. Jobs & workers (what happens to a job / a worker)

**15. What is a "job" and why only the delivery id?**
A job is a BullMQ queue entry telling a worker "process this delivery." It carries
only `{ deliveryId }`. Because Postgres is the source of truth, the worker reloads
the current delivery/event/endpoint by id — so it always acts on fresh data and
the job can't carry stale payloads.

**16. What happens to a job after success? After permanent failure?**
On success the job completes and is removed (`removeOnComplete: true`). On a
permanent/exhausted failure the job is kept in BullMQ's failed set
(`removeOnFail: false`) for inspection, while the delivery row records the real
terminal state (`failed`/`dead`) in Postgres.

**17. Worker POSTs, gets 200, then crashes before recording success — then?**
The receiver already got it, but our DB still says `pending`, so the job is
retried and **the same webhook is sent again**. That's expected under
at-least-once. Two things keep it safe: before sending, the processor skips any
delivery already in a terminal state (catches the case where we *did* record
success); and the receiver dedupes on the stable `X-Webhook-Id`.

**18. A worker is killed mid-delivery — how does the job come back?**
The job was "active" with a lock held by the dead worker. BullMQ's **stalled-job
check** (≈30s interval) notices the lock wasn't renewed and returns the job to the
queue, where another worker re-runs it. So recovery takes tens of seconds, not
instantly — and nothing is lost. (Measured ~31s in the experiments.)

**19. Three worker replicas — how do they avoid double-delivering one job?**
BullMQ gives each job to exactly one worker and locks it while in progress; other
workers can't pick up a locked job. Combined with `jobId = deliveryId` dedupe and
the DB unique constraint, the same delivery isn't sent twice concurrently.

**20. What does graceful SIGTERM shutdown do to in-flight jobs?**
`worker.close()` stops accepting new jobs and waits for in-flight ones to finish
(or releases their locks), then closes DB/Redis. This avoids leaving half-done
work or corrupt state; a hard timeout guards against a job that never drains.

**21. Why is `jobId` the delivery id?**
Queue-level dedupe. Re-enqueuing the same delivery (a retried publish, or the
sweeper) collapses to one job instead of creating duplicates — belt-and-braces on
top of the row-level unique constraint.

**22. Redis is wiped while jobs are queued — what happens?**
The queued jobs vanish, but the deliveries are still `pending` in Postgres. On its
next pass the **sweeper** finds those due-pending deliveries and re-enqueues them,
rebuilding the queue from the source of truth. (Measured recovery ~1.3s.)

## E. Retries, backoff, timeouts

**23. Which failures do you retry, and why is 400 different from 500?**
Retry 5xx, timeouts, and network errors — plus the two transient 4xx, 429 and
408. Don't retry other 4xx: a 400/401/404 means the request itself is wrong and
will fail identically forever, so retrying just wastes attempts and hammers the
receiver.

**24. Why throw on retryable but return on permanent?**
BullMQ retries a job only when the processor **throws**. So throwing = "try
again," returning = "done, stop." Throw on a 400 and you'd burn all 6 attempts on
something that can never succeed; return on a 500 and you'd silently drop a
delivery that would've worked next time.

**25. Describe the backoff schedule.**
`delay = min(5min, 1s × 2^(attempt-1))`, then full jitter applied — so roughly
1s, 2s, 4s, 8s, 16s… capped at 5 minutes, over 6 total attempts. A delivery
reaches `dead` within a few minutes.

**26. Why jitter?**
Without it, when a downed receiver recovers, thousands of deliveries that failed
together would all retry at the *same instant* — a thundering herd that knocks it
back down. Full jitter (`random() × delay`) spreads retries evenly across the
window so recovery is smooth.

**27. Outbound POST timeout and a hanging receiver?**
Every POST uses an 8s hard timeout (`AbortSignal.timeout`). If the receiver hangs
past it, the request aborts, the attempt is recorded with a null status code and a
timeout error, it's classified retryable, and the worker moves on — a hanging
receiver can never block a worker.

**28. How does a delivery reach `dead`, and after how many attempts?**
After the final allowed attempt (6th by default) fails with a retryable error, the
processor marks it `dead` and throws, so BullMQ also moves the job to its failed
set. `dead` is the dead-letter state.

**29. Who owns retry scheduling — you or BullMQ?**
BullMQ, via `attempts` + a custom jitter backoff strategy. Pro: reuses
battle-tested scheduling and locking. Con: the processor must be disciplined
about throw-vs-return. Postgres still records the truth (status, attempts,
next_attempt_at).

## F. Signing & security

**30. How is an outgoing webhook signed?**
HMAC-SHA256 over the string `"{timestamp}.{body}"` using the endpoint's secret.
We send `X-Webhook-Id` (the event id), `X-Webhook-Timestamp` (unix seconds), and
`X-Webhook-Signature: sha256=<hex>`.

**31. Why sign timestamp + body, not just body?**
Binding the timestamp into the signature lets the receiver reject stale/replayed
requests *and* stops an attacker from changing the timestamp without invalidating
the signature.

**32. How does a receiver verify, and why constant-time compare?**
It recomputes the HMAC with its copy of the secret over `"{timestamp}.{body}"`
and compares. A constant-time comparison (`timingSafeEqual`) avoids leaking, via
response timing, how many characters of the signature matched — which a naive
`==` could.

**33. Replay-attack protection?**
The receiver rejects any request whose timestamp is outside a tolerance window
(default 300s), so a captured-and-resent request fails once it's stale.

**34. How are API keys stored, and why SHA-256 not bcrypt?**
Keys are high-entropy random tokens; we store only their SHA-256 hash and look up
by hash. A fast hash is correct here because there's nothing to brute-force, and
it gives an O(1) exact-match lookup on every request. Bcrypt/argon2 are
*deliberately slow* to defend **low-entropy human passwords** — overkill and a
latency cost for random tokens.

## G. Circuit breaker, rate limiting, concurrency

**35. Circuit breaker states and transitions.**
Per endpoint, state in Redis. **Closed** = normal; 5 consecutive failures trip it
**Open**. **Open** = for 30s we skip the POST and fast-fail. After the cooldown it
goes **Half-open** and allows one trial: success → Closed (reset), failure →
Open again.

**36. When open, does a delivery still consume a retry attempt? Why?**
Yes. An open-circuit skip is recorded as a fast retryable failure that consumes an
attempt, so a permanently-dead endpoint still marches to the DLQ instead of
sitting pending forever. The breaker's value is saving the wasted network call and
the 8s timeout wait, not changing the budget. (A permanent 4xx does **not** trip
the breaker — it's a client error, not an availability problem.)

**37. You replay a dead delivery and it won't deliver for ~30s — why?**
Sustained failures left that endpoint's breaker **open**. Replaying immediately
just fast-fails until the ~30s cooldown elapses; then the breaker half-opens, the
trial hits the now-healthy receiver, and it succeeds. So you wait out the cooldown
before replaying.

**38. How does per-endpoint rate limiting work and where?**
In the worker, before each POST: a fixed 1-second window counter in Redis per
endpoint. If the endpoint has already had its allowed number of sends this second,
the delivery is throttled.

**39. Why does throttling *not* consume an attempt but an open circuit *does*?**
Throttling is **us** choosing to slow down — the receiver is fine — so it would be
unfair to spend a retry; the job is rescheduled with `moveToDelayed` and the
attempt count is untouched. An open circuit means the receiver looks **broken** —
that's a real failure, so it counts, which keeps a dead endpoint moving toward the
DLQ.

**40. One slow receiver is hammered — what protects it and the workers?**
The 8s timeout (a slow receiver can't tie up a worker), per-endpoint rate limit
and concurrency cap (we don't flood it), and the circuit breaker (after repeated
failures we stop calling it entirely for a cooldown). Together they isolate one
bad endpoint from the rest.

## H. Dead-letter queue & replay

**41. What is your DLQ, concretely?**
Not a separate Redis queue — it's simply `deliveries` with `status = 'dead'` in
Postgres (the retained BullMQ failed jobs are secondary). `GET
/deliveries?status=dead` lists them. DB-backed means it's durable and queryable.

**42. What happens on replay, and how do you avoid dedupe against the old job?**
Replay resets the row to `pending`, `attempts_count = 0`, `next_attempt_at = now`,
clears `last_error`, then **removes any existing job with that jobId** before
enqueuing a fresh one — otherwise `jobId` dedupe would collapse the new job into
the old dead one. Only `dead`/`failed` deliveries are replayable.

**43. Receiver down 10 minutes then recovers — honestly, what happens?**
With the default 6-attempt / 5-min-cap schedule, a persistently-down receiver
reaches the **DLQ in a few minutes, not 10** — so the real recovery is **replay
after it comes back**, not silent auto-healing. A *short* outage (within the retry
budget) is absorbed automatically; a long one needs replay.

## I. Reliability: the dual-write gap & the sweeper

**44. Describe the "DB write succeeded but enqueue failed" gap.**
In publish, we commit the delivery row and *then* enqueue the job. If the process
dies in between, the delivery is left `pending` with no job — a silently lost
webhook. It's the two-lines-apart `INSERT … deliveries` then `enqueueDelivery` in
the events route.

**45. Why can't a DB transaction fix it, and what are the two fixes?**
Because it's a write to **two different systems** (Postgres and Redis); a DB
transaction only makes the Postgres write atomic, not the Redis enqueue. The two
standard fixes are a **transactional outbox** (write an outbox row in the same DB
transaction, a relay publishes it) or a **reconciling sweeper**. We chose the
sweeper — simpler, and it doubles as Redis recovery.

**46. How does the sweeper work, and why is it safe + recovery-capable?**
A single scheduler process every 10s selects due `pending` deliveries and
enqueues them. It's safe to run repeatedly because `jobId = deliveryId` dedupes
anything already queued (a no-op), so it can only ever add the *missing* ones.
Since "what should exist" always lives in Postgres, it rebuilds the entire queue
after a Redis wipe.

## J. SSRF & input safety

**47. How do you stop a customer URL from hitting internal services, and why check twice?**
`assertUrlAllowed` rejects non-http(s) URLs, embedded credentials, and any host
that resolves to a loopback/private/link-local/CGNAT/multicast address (including
the cloud metadata IP 169.254.169.254). We check at **endpoint creation** and
again in the worker **before each POST**, because DNS can rebind — a host that
looked public at creation can later point at an internal IP. (A dev-only
`SSRF_ALLOW_PRIVATE` flag lets the in-cluster test receiver work.)

## K. Observability

**48. How are `/metrics` computed, and why DB-derived?**
`/metrics` (prom-client) exposes default process metrics, one in-process counter
for events published by the API, and gauges (deliveries by status, total events,
attempts by result) that are **queried from Postgres on each scrape**. They're
DB-derived because deliveries happen in the *worker* process, which this endpoint
doesn't scrape — and Postgres is the source of truth, so the counts stay correct
across processes and restarts.

## L. Testing & CI

**49. How do you test retry/DLQ deterministically with real timing and a real queue?**
Integration tests run against **real Postgres + Redis** (isolated test DB and
Redis index) and start a **real in-process BullMQ worker** pointed at a
controllable local receiver (succeed / 500 / 400 / hang / succeed-after-N). We
only "mock" the receiver's behavior, never the queue or DB. Determinism comes from
a shrunk clock (tiny backoff/timeout), polling for state instead of sleeping, and
running files serially. CI (GitHub Actions) spins up Postgres + Redis service
containers and runs lint → typecheck → build → tests; the slow, timing-sensitive
chaos script stays out of that lane.

## M. Trade-offs, scaling & honesty

**50. Current bottlenecks, how to scale 10x, and which numbers are trustworthy?**
On the dev box the ceiling is CPU/single-machine contention — everything (API,
worker, scheduler, Postgres, Redis, receiver, load generator) runs on one 6-core
laptop; raising the DB pool didn't help, which we measured and concluded the pool
wasn't the limit. To scale: run components on separate hosts, use managed/remote
Postgres + Redis, add worker replicas, and cut per-delivery round-trips. On
honesty: the reported numbers (≈300+ publish req/s, p99 ~110–150ms, ~86
deliveries/s, recovery times) are **real measurements with full context**, but
they're **dev-box, localhost, in-container** — a comparison ceiling, not a
production figure — and the raw output is committed so anyone can check.
