# Architecture Decision Records

Each ADR documents a key architectural choice: context → decision → consequences.

---

## ADR-001: MongoDB over PostgreSQL

**Status:** Accepted

**Context:**
Gaming café data is document-shaped. A booking contains nested slot details, tier discounts,
and coupon metadata that would span 4-5 relational tables with complex JOINs on every read.
Schema evolves fast during early product — adding a new loyalty tier or pricing rule shouldn't
require a migration.

**Decision:**
MongoDB Atlas with Mongoose. Documents map naturally to booking/session/order shapes.
Horizontal sharding available when needed. Atlas handles backups, indexes, and geo-replication.

**Consequences:**
- ✅ Flexible schema — added 12 fields during implementation with zero migrations
- ✅ Aggregation pipeline replaces most JOIN logic
- ✅ Atlas free tier usable for development
- ❌ No multi-document ACID transactions by default (mitigated: Saga pattern handles distributed consistency)
- ❌ No foreign key enforcement (mitigated: Mongoose refs + application-level validation)

---

## ADR-002: BullMQ over direct async calls for background work

**Status:** Accepted

**Context:**
Session billing, notifications, and leaderboard updates happen after HTTP response is sent.
Doing them synchronously blocks the response. Doing them with `Promise.all` in the handler
loses work if the process crashes mid-execution.

**Decision:**
BullMQ backed by Redis. Every non-critical side effect is enqueued as a job with retry policy.
Workers run in a separate process (`apps/workers`) so API crashes don't kill in-flight jobs.

**Consequences:**
- ✅ HTTP response time decoupled from billing complexity
- ✅ Automatic retry with exponential backoff
- ✅ Dead Letter Queue for manual replay of failed jobs
- ✅ Priority queues — billing beats report generation
- ❌ Eventual consistency — leaderboard update lags session end by seconds
- ❌ Requires Redis (already a dependency for cache, so no additional infra)

---

## ADR-003: Outbox Pattern over direct BullMQ enqueue in service

**Status:** Accepted

**Context:**
A race condition exists: service writes to MongoDB, then crashes before calling `queue.add()`.
Event is lost. Booking is created but session-billing job never runs. User gets free time.

**Decision:**
Every event-producing write also inserts an `OutboxEvent` document in the same Mongoose
operation. A background poller (every 5s) reads pending outbox events and publishes them
to BullMQ using `jobId` deduplication. This guarantees at-least-once delivery.

**Consequences:**
- ✅ No event loss on process crash
- ✅ Exactly-once delivery at worker level (workers are idempotent)
- ✅ Audit trail — every event is recorded with status
- ❌ 5s delivery lag (acceptable for this domain)
- ❌ Extra collection + poller process

---

## ADR-004: Saga Orchestrator over try/catch compensation

**Status:** Accepted

**Context:**
Booking creation involves 4 steps across 2 systems (MongoDB + Redis): deduct wallet,
apply coupon, create booking document, emit event. Original code had nested try/catch
blocks with manual compensation — brittle and hard to reason about.

**Decision:**
Generic `Saga<TContext>` class. Each step declares `execute` and `compensate`. On failure,
Saga runs compensations in reverse order automatically. Steps are explicit, compensations
are co-located with their steps.

**Consequences:**
- ✅ Compensation logic explicit and testable
- ✅ New steps added without touching compensation wiring
- ✅ Saga log persisted to MongoDB for debugging
- ❌ More boilerplate per step
- ❌ No distributed Saga coordinator — all steps run in single service process (acceptable — no microservices split here)

---

## ADR-005: Redis Pub/Sub for WebSocket horizontal scaling

**Status:** Accepted

**Context:**
WebSocket connections are sticky to one server instance. When a session ends on Instance A,
it needs to emit `session.tick` to the user connected on Instance B. Without coordination,
events are silently dropped.

**Decision:**
Separate Redis pub/sub clients (cannot share connection with regular client — blocked during
subscribe). On event: `publish("ws:events", { userId, event, data })`. Every instance
subscribes and checks local `Map<userId, Set<WebSocket>>`. If user is local, send. If not, no-op.

**Consequences:**
- ✅ Any instance can deliver to any connected user
- ✅ No external coordination service needed (Redis already present)
- ✅ Fan-out within instance uses local Map (zero Redis round-trips for local delivery)
- ❌ Message lost if no instance has that user connected (acceptable — WebSocket is best-effort)
- ❌ Requires 2 additional Redis connections (pub + sub) per process

---

## ADR-006: MongoDB Change Streams over service-level emitToUser calls

**Status:** Accepted

**Context:**
Originally, every service that changed booking/session/order state called `emitToUser()`
directly. This coupled business logic to WebSocket infrastructure and made services
untestable without mocking WS.

**Decision:**
Change Streams watch MongoDB collections. On document change, the stream handler emits
the WS event. Services are pure — they write to DB, nothing else. Change stream persists
resume token to Redis so it restarts from last position after crash (at-least-once delivery).

**Consequences:**
- ✅ Services are pure — no WS dependency
- ✅ Any DB write (even from admin panel, migrations) fires WS event automatically
- ✅ At-least-once delivery via resume token
- ❌ Requires MongoDB replica set (Atlas provides this)
- ❌ Additional latency: DB write → change stream event → WS emit (~10-50ms)

---

## ADR-007: RS256 JWT over HS256

**Status:** Accepted

**Context:**
HS256 uses symmetric key — any service that verifies tokens can also issue them.
If a verification-only service is compromised, attacker can mint arbitrary tokens.

**Decision:**
RS256. Private key held only by `apps/api` auth module. Public key distributed to
any service that needs to verify. Workers verify tokens without being able to issue them.

**Consequences:**
- ✅ Compromise of worker process cannot produce valid tokens
- ✅ Standard for multi-service architectures
- ❌ Key rotation requires distributing new public key (acceptable — documented in runbook)
- ❌ Slightly larger token (RSA signature is bigger than HMAC)

---

## ADR-008: TypeScript monorepo over separate services

**Status:** Accepted

**Context:**
Alternative was separate Node.js microservices communicating over HTTP. At this scale
(1 gaming café, ~200 concurrent users), that adds deployment complexity, network latency
between services, and distributed tracing overhead with minimal benefit.

**Decision:**
npm workspaces monorepo. `packages/*` for shared code. `apps/api` and `apps/workers`
are separate Node.js processes but share types, models, and Redis/DB packages.
Process boundary provides fault isolation; shared packages avoid code duplication.

**Consequences:**
- ✅ Single deploy pipeline, single repo, shared types catch bugs at compile time
- ✅ Workers can be scaled independently (separate Docker containers)
- ✅ Shared `packages/db` means model changes propagate everywhere instantly
- ❌ Harder to split into true microservices later (mitigated: module boundaries are clean)
- ❌ All packages rebuild on any change (mitigated: build order script in package.json)

---

## ADR-009: HMAC-SHA256 for ESP32 webhook authentication

**Status:** Accepted

**Context:**
ESP32 hardware posts RFID scan data to `/api/v1/cards/scan`. Without authentication,
anyone on the network can POST fake scans to unlock doors or credit loyalty points.

**Decision:**
Shared secret between ESP32 firmware and API server. ESP32 computes
`HMAC-SHA256(secret, request_body)` and sends `X-Signature: sha256=<hex>` header.
API verifies using `crypto.timingSafeEqual` to prevent timing attacks.

**Consequences:**
- ✅ Forged requests rejected at middleware layer before business logic
- ✅ Timing-safe comparison prevents signature oracle attacks
- ❌ Secret rotation requires OTA firmware update (acceptable — infrequent)
- ❌ No replay protection (mitigated: add timestamp + nonce if needed in v2)

---

## ADR-010: Bulkhead Connection Pools over single MongoDB connection

**Status:** Accepted

**Context:**
Analytics queries (leaderboard aggregation, daily reports) can take 2-5 seconds.
On a single connection pool, these queries consume connections and slow down booking
and session queries which must complete in <100ms.

**Decision:**
Three separate `mongoose.createConnection()` pools: `transactional` (10 connections, primary),
`analytics` (3 connections, secondaryPreferred), `background` (2 connections, for cron/outbox).
Named after Michael Nygard's bulkhead pattern (Release It!, 2007).

**Consequences:**
- ✅ Slow analytics queries cannot starve transactional reads/writes
- ✅ Analytics reads from secondary — reduces primary load
- ✅ Background tasks isolated — outbox poller cannot block bookings
- ❌ 3× connection overhead vs single pool
- ❌ Must pass correct connection to each model (enforced via model factory pattern)
