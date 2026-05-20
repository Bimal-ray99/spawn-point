# Distributed Systems Patterns — Implementation Reference

Each pattern: problem → solution → where implemented → trade-off.

---

## 1. Saga Orchestrator

**Problem:** Booking spans multiple steps (wallet deduct → coupon apply → DB write → event emit).
If step 3 fails, steps 1 and 2 have already executed. Data is inconsistent.

**Solution:** Define each step with an `execute` and `compensate` function. On failure, run
compensations in reverse. No two-phase commit required.

```typescript
// apps/api/src/shared/saga.ts
const bookingSaga = new Saga<BookingContext>()
  .step(deductWallet,     refundWallet)
  .step(applyCoupon,      revertCoupon)
  .step(createBooking,    cancelBooking)
  .step(writeOutboxEvent, deleteOutboxEvent);

await bookingSaga.run(ctx);
```

**Where:** `apps/api/src/modules/booking/booking.saga.ts`

**Trade-off:** Compensations must be idempotent. Step 3 failure runs compensations for steps 1 and 2.
If compensation itself fails, saga log captures state for manual recovery.

---

## 2. Outbox Pattern

**Problem:** Service writes to DB, then crashes before calling `queue.add()`. Event lost.
Booking created, billing job never runs. User gets free time.

**Solution:** Write `OutboxEvent` in same Mongoose operation as the business document.
Background poller reads pending events and publishes to BullMQ with `jobId` deduplication.

```
DB write (atomic):
  BookingDocument { status: "confirmed" }
  OutboxEvent     { aggregate_type: "booking", event_type: "booking.created", status: "pending" }

Poller (every 5s):
  find OutboxEvents where status="pending"
  → queue.add(eventType, payload, { jobId: outboxEvent._id })  ← dedup key
  → OutboxEvent.status = "published"
```

**Where:** `packages/db/src/models/outbox-event.model.ts`, `apps/workers/src/jobs/outbox.poller.ts`

**Trade-off:** 5s delivery lag. Acceptable for this domain. Provides at-least-once delivery guarantee.

---

## 3. Circuit Breaker

**Problem:** Payment gateway goes down. Every booking attempt waits 30s for timeout, threads pile up,
entire API slows down. One external dependency takes down the whole system.

**Solution:** Redis-backed state machine. After N failures in T seconds, open the circuit.
Fail fast with fallback. After cooldown, allow one probe. Close on success.

```
CLOSED → (5 failures in 60s) → OPEN → (30s cooldown) → HALF-OPEN → (probe succeeds) → CLOSED
                                  ↓                         ↓
                             fail fast               (probe fails) → OPEN
```

**Where:** `apps/api/src/shared/circuit-breaker.ts`
Applied to: payment gateway, FCM push notifications, Resend email.

**Trade-off:** False positives possible — circuit opens on transient spike. Tune threshold per dependency.

---

## 4. Bulkhead (Connection Pool Isolation)

**Problem:** Leaderboard aggregation query takes 3s. Single connection pool: all 10 connections busy
with analytics. Booking query queues. p99 spikes to 15s.

**Solution:** Three isolated connection pools. Slow analytics queries cannot consume transactional connections.

```typescript
// packages/db/src/index.ts
connections.transactional = mongoose.connection;           // 10 conn, primary
connections.analytics     = mongoose.createConnection(uri, { maxPoolSize: 3, readPreference: "secondaryPreferred" });
connections.background    = mongoose.createConnection(uri, { maxPoolSize: 2 });
```

**Where:** `packages/db/src/index.ts`

**Trade-off:** 3× connection overhead. At MongoDB Atlas free tier (500 connections max), negligible.

---

## 5. Distributed Cron Lock

**Problem:** Two API instances both run the leaderboard rebuild cron at midnight.
Double writes, duplicate notifications, incorrect ranks.

**Solution:** Redis `SET key token EX ttl NX` — atomic lock. Only one instance gets the lock.
Others see null return and skip. Lock auto-expires if instance crashes mid-job.

```typescript
// packages/redis/src/index.ts
async function withCronLock(name: string, ttlMs: number, fn: () => Promise<void>) {
  const token = await acquireLock(`cron:${name}`, Math.floor(ttlMs / 1000) - 5);
  if (!token) return; // another instance is running this
  try { await fn(); } finally { await releaseLock(`cron:${name}`, token); }
}
```

**Where:** `packages/redis/src/index.ts`, all cron jobs in `apps/workers/src/jobs/cron.jobs.ts`

**Trade-off:** Lock TTL must be longer than job duration. Set TTL = expected duration × 1.5.

---

## 6. Self-Scheduling Pattern (No Cron Overlap)

**Problem:** `cron.schedule("*/5 * * * *", fn)` fires at wall-clock intervals. If job takes 6 minutes,
two instances overlap. Race conditions on shared state.

**Solution:** Recursive `setTimeout`. Next run starts only after current finishes. Interval is between completions, not between starts.

```typescript
async function scheduleRepeating(fn: () => Promise<void>, intervalMs: number) {
  await fn().catch(err => logger.error({ err }, "Cron failed"));
  setTimeout(() => scheduleRepeating(fn, intervalMs), intervalMs);
}
```

**Where:** `apps/workers/src/jobs/cron.jobs.ts`

**Trade-off:** Wall-clock schedule drifts over time. Acceptable for internal jobs (leaderboard, reports).
Use real cron for user-facing scheduled events (booking reminders — use BullMQ `delay`).

---

## 7. XFetch — Cache Stampede Prevention

**Problem:** Leaderboard cache TTL expires at peak hour. 500 concurrent users all miss.
All 500 hit MongoDB simultaneously. DB overwhelmed.

**Solution:** XFetch algorithm. Probabilistically recompute before TTL expires based on remaining
time and computation cost. One request recomputes early; others serve stale cache.

```
prob = -beta * delta * log(rand()) > TTL - age
where delta = recompute time, beta = tuning parameter (default 1.0)
```

**Where:** `packages/redis/src/cache.ts`
Applied to: leaderboard top-100, menu reads, venue schedule.

**Trade-off:** Serves slightly stale data. Acceptable for leaderboard (users don't notice 30s lag).
Not appropriate for wallet balance (always read from DB).

---

## 8. Idempotency Middleware

**Problem:** User clicks "Book" twice due to network lag. Two booking requests arrive. Both succeed.
Double-booked, double-charged.

**Solution:** Client sends `X-Idempotency-Key` header on POST/PATCH. Middleware caches response in Redis
for 24h keyed on `userId:path:idempotencyKey`. Duplicate request gets cached response replayed.

```
First request:  cache miss → execute → store { statusCode, body } in Redis → return response
Second request: cache hit  → return cached response immediately, no business logic executed
```

**Where:** `apps/api/src/middleware/idempotency.middleware.ts`
Applied to: booking create, food order, wallet top-up, tournament registration.

**Trade-off:** Client must generate and retry with same key. Adds 1 Redis read per mutation.

---

## 9. Optimistic Concurrency Control (OCC)

**Problem:** Admin cancels booking at same time as user extends it. Last write wins.
Extension overwrites cancellation. User keeps slot that should be cancelled.

**Solution:** `version` field on document. Read → modify → write with `findOneAndUpdate({ _id, version })`.
If version mismatch (concurrent write happened), retry up to 3×.

```typescript
// Booking has version: 0
const updated = await Booking.findOneAndUpdate(
  { _id: bookingId, version: currentVersion },
  { $set: { status: "cancelled" }, $inc: { version: 1 } },
  { new: true }
);
if (!updated) throw new ConflictError("Concurrent modification, retry");
```

**Where:** `apps/api/src/modules/booking/booking.service.ts`

**Trade-off:** Retry storms possible under high contention. Acceptable — booking conflicts are rare.
For high-contention resources (wallet balance), use MongoDB atomic `$inc` instead.

---

## 10. Dead Letter Queue + Admin Replay

**Problem:** Notification fails 3 times (FCM down). Job disappears. Admin has no visibility.
Cannot replay when FCM recovers.

**Solution:** After `maxAttempts`, route job to `dead-letter` queue. DLQ worker logs and alerts.
Admin API provides `GET /admin/dlq`, `POST /admin/dlq/:id/replay`, `DELETE /admin/dlq/:id`.

```
Worker fails × 3 → worker.on("failed") → deadLetterQueue.add(job.data)
Admin: GET /admin/dlq       → list failed jobs
Admin: POST /admin/dlq/:id/replay → re-enqueue to original queue
```

**Where:** `apps/workers/src/workers/dlq.worker.ts`, `apps/api/src/modules/admin/admin.service.ts`

**Trade-off:** DLQ grows unbounded if not monitored. Add TTL on DLQ jobs (7 days) and alert on size threshold.

---

## 11. Redis Pub/Sub for WebSocket Scaling

**Problem:** User connected to Instance A. Session ends on Instance B.
Instance B has no local WebSocket for that user. Event dropped silently.

**Solution:** Dedicated pub client and sub client (cannot share — subscribe blocks connection).
On event: publish to `ws:events` channel. Every instance subscribes and fan-outs to local sockets.

```
Instance B: publish("ws:events", { userId: "123", event: "session.tick", data: {...} })
Instance A: subscribe handler → localClients.get("123")?.forEach(ws => ws.send(msg))
```

**Where:** `packages/redis/src/pubsub.ts`, `apps/api/src/shared/websocket.ts`

**Trade-off:** Message dropped if user disconnects between publish and delivery. WebSocket is best-effort.
For guaranteed delivery, use notification model (persisted) + WebSocket as transport.

---

## 12. Slot Lease / Pre-Booking Reservation

**Problem:** User sees slot available, spends 2 minutes filling booking form, submits — slot taken.
Race condition between availability check and booking create.

**Solution:** `POST /bookings/lease` atomically reserves slot in Redis (TTL 300s). Returns `leaseToken`.
Booking create validates token exists before conflict check. Expired lease = slot released automatically.

```
POST /bookings/lease → SET slot:lease:systemId:slot leaseToken EX 300 NX
                     → return { leaseToken, expiresAt }
POST /bookings       → validate leaseToken in Redis → proceed with booking
DELETE /bookings/lease/:token → release early
```

**Where:** `apps/api/src/modules/booking/booking.routes.ts`, `booking.service.ts`

**Trade-off:** Slot held for 5 minutes even if user abandons. At gaming café scale (~20 stations), acceptable.
Scale issue at 10,000 stations — need distributed lease with shorter TTL.

---

## 13. Double-Entry Bookkeeping

**Problem:** Wallet debits accumulate. At month end, admin runs `SUM(debits)` — doesn't match
`SUM(credits)` due to a bug in a worker that debited without creating matching credit. Money lost.

**Solution:** Every debit creates matching credit entry with `account_type` field.
Admin endpoint `verifyLedgerBalance()` asserts `SUM(debits) === SUM(credits)`.

```
User pays 500 points for booking:
  Transaction { type: "debit",  account_type: "user_wallet",   amount: 500 }
  Transaction { type: "credit", account_type: "venue_revenue",  amount: 500 }
```

**Where:** `packages/db/src/models/transaction.model.ts`, `apps/api/src/modules/wallet/wallet.service.ts`

**Trade-off:** 2× transaction writes. At this scale, trivial. Provides strong financial integrity guarantee.

---

## 14. OpenTelemetry Distributed Tracing

**Problem:** Request fails. Logs show error in worker. Cannot correlate to original HTTP request.
Cannot see which step in booking saga was slow.

**Solution:** OTel context carrier injected into every BullMQ job payload. Workers extract context
and create child spans. Correlation ID propagated in all Pino log entries.

```
HTTP Request → span: "POST /api/v1/bookings"
  → queue.add({ data, _otel: propagator.inject(context) })
    → Worker: propagator.extract(job.data._otel) → childSpan: "session-events.worker"
```

Sampling: 10% head sampling (reduce overhead) + 100% tail sampling on errors (catch every failure).

**Where:** `apps/api/src/shared/tracing.ts`, `apps/workers/src/shared/tracing.ts`

**Trade-off:** 10% head sampling misses 90% of successful traces. Adjust to 100% during debugging.
Tail sampling on errors requires collector-side config (OpenTelemetry Collector).
