# Spawnpoint — System Design

## Overview

Spawnpoint is a production-grade gaming café management platform. Backend is a TypeScript monorepo
exposing REST APIs + WebSocket events, backed by MongoDB Atlas, Redis, and BullMQ workers.
Hardware layer: ESP32 + RDM6300 RFID readers for tier-based loyalty cards and door access.

---

## High-Level Architecture

```
┌─────────────────────────────────────────────────────────────────────┐
│                          CLIENT LAYER                               │
│  Web App (React)    Mobile App    ESP32 RFID Hardware               │
└──────────┬──────────────┬──────────────┬────────────────────────────┘
           │ HTTPS        │ HTTPS        │ HTTPS + HMAC-SHA256
           ▼              ▼              ▼
┌─────────────────────────────────────────────────────────────────────┐
│                     NGINX (Edge)                                    │
│  TLS termination · Rate limiting (auth/device/global zones)         │
│  HTTP→HTTPS redirect · Security headers                             │
└──────────────────────────┬──────────────────────────────────────────┘
                           │
           ┌───────────────┴───────────────┐
           ▼                               ▼
┌────────────────────┐         ┌────────────────────┐
│   apps/api         │         │   apps/workers      │
│   Express REST     │         │   BullMQ Consumers  │
│   + WebSocket      │         │                     │
│   Port 3000        │         │   session-events    │
│                    │         │   notification      │
│   13 modules       │         │   outbox-poller     │
│   60+ endpoints    │         │   dlq               │
└────────┬───────────┘         └────────┬────────────┘
         │                              │
         ▼                              ▼
┌─────────────────────────────────────────────────────────────────────┐
│                        DATA LAYER                                   │
│                                                                     │
│  ┌──────────────────┐  ┌─────────────┐  ┌───────────────────────┐  │
│  │  MongoDB Atlas   │  │    Redis    │  │       BullMQ          │  │
│  │                  │  │             │  │                       │  │
│  │  3 conn pools:   │  │  14 key     │  │  Priority queues      │  │
│  │  - transactional │  │  patterns   │  │  DLQ + replay         │  │
│  │  - analytics     │  │  Pub/Sub    │  │  Exponential backoff  │  │
│  │  - background    │  │  Lua scripts│  │                       │  │
│  └──────────────────┘  └─────────────┘  └───────────────────────┘  │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Request Lifecycle

```
Request
  → Nginx (rate limit + TLS)
  → Express middleware chain:
      initTracing()        ← OTel span start
      requestId()          ← X-Request-ID header
      pinoHttp()           ← structured log with correlation ID
      helmet()             ← security headers
      rateLimit()          ← token bucket (Redis Lua)
      jwtMiddleware()      ← RS256 verify, attach req.user
      roleGuard()          ← RBAC check
      zodValidate()        ← input validation, strip unknown fields
  → Route handler
  → Service (business logic only)
      → Redis cache read
      → MongoDB query (lean() on reads)
      → BullMQ enqueue (with OTel context carrier)
      → OutboxEvent write (same transaction)
  → sendSuccess() / sendError()
  → OTel span end
```

---

## Database Design

### Connection Pools (Bulkhead Pattern)

| Pool | Max Connections | Read Preference | Purpose |
|---|---|---|---|
| transactional | 10 | primary | bookings, sessions, wallet |
| analytics | 3 | secondaryPreferred | reports, leaderboard reads |
| background | 2 | primary | outbox poller, cron jobs |

One slow analytics query cannot starve booking queries.

### 34 Collections

**Core:** users, refresh_tokens, user_sessions  
**Gaming:** sessions, bookings, group_bookings, waitlist_entries, gaming_systems, venues  
**Financial:** wallets, transactions, point_packs, pack_purchases, coupons, coupon_usages  
**Social:** leaderboard_entries, leaderboard_views, loyalty_streaks, referral_rewards  
**Food:** food_items, food_orders  
**Engagement:** tournaments, tournament_participants, flash_sales, announcements, announcement_views  
**Ops:** notifications, support_tickets, audit_logs, maintenance_requests  
**Infrastructure:** outbox_events, saga_logs, webhooks, app_configs, analytics_rollups  
**Hardware:** user_cards, card_issuance_logs, venue_checkins, seller_credit_logs  

---

## Redis Architecture

### 14-Key Caching Strategy

| Key Pattern | TTL | Purpose |
|---|---|---|
| `station:status:<id>` | 30s | Real-time availability |
| `session:active:<userId>` | session duration | Active session guard |
| `member:tier:<rfidCode>` | 5 min | RFID tier lookup |
| `booking:slot:<date>:<stationId>` | 1 min | Conflict check |
| `report:daily:<date>` | 1 hr | Pre-computed daily report |
| `config:booking_rules` | 1 hr | Dynamic pricing rules |
| `lb:<period>:<key>` | 90 days | Leaderboard sorted sets (ZSET) |
| `auth:refresh:<userId>` | 7 days | Refresh token store |
| `auth:blacklist:<jti>` | access TTL | Revoked token blacklist |
| `idempotency:<userId>:<path>:<key>` | 24 hr | Idempotency cache |
| `circuit:<name>` | cooldown | Circuit breaker state |
| `cron:<name>` | job TTL | Distributed cron lock |
| `slot:lease:<systemId>:<slot>` | 300s | Pre-booking reservation |
| `rate:<ip>` | 1 min | Token bucket counter |

### Pub/Sub for WebSocket Scaling

```
Instance A (user connected here)        Instance B (event originates here)
  localClients.get(userId)  ←──────── subscribe("ws:events") ←── publish("ws:events", msg)
  forEach(socket => send())
```

Allows horizontal scaling — any instance can emit to any connected user.

---

## BullMQ Architecture

### Queues

| Queue | Priority | Producer | Consumer |
|---|---|---|---|
| session-events | 1 (highest) | session.service | session-events.worker |
| booking-events | 5 | booking.service | booking-events.worker |
| notifications | 10 | any service | notification.worker |
| outbox-relay | 10 | outbox.poller | – (direct publish) |
| reports | 20 (lowest) | cron | report.worker |
| dead-letter | – | all workers (on maxAttempts) | dlq.worker |

### Retry Strategy

```
Attempt 1: immediate
Attempt 2: +2s
Attempt 3: +4s
→ Dead Letter Queue → admin replay or discard
```

---

## Distributed Systems Patterns

### 1. Saga Orchestrator (Booking Transaction)

```
Step 1: deductWallet      ← compensate: refundWallet
Step 2: incrementCoupon   ← compensate: decrementCoupon
Step 3: createBookingDoc  ← compensate: cancelBookingDoc
Step 4: writeOutboxEvent  ← compensate: deleteOutboxEvent

On any step failure → run compensations in reverse order
```

### 2. Outbox Pattern (Guaranteed Delivery)

```
Service: write(BookingDoc + OutboxEvent) ← same DB write, atomic
Poller (5s): find OutboxEvents where status="pending"
           → enqueue to BullMQ with jobId dedup
           → mark status="published"
```

Prevents event loss if process crashes after DB write but before queue enqueue.

### 3. Circuit Breaker (Cascading Failure Prevention)

```
States: CLOSED → OPEN → HALF-OPEN → CLOSED

CLOSED:    allow all calls
OPEN:      fail fast, return cached/fallback (Redis stores state + cooldown TTL)
HALF-OPEN: allow 1 probe call, close on success, reopen on failure
```

Applied to: payment gateway, FCM push, Resend email.

### 4. XFetch Cache Stampede Prevention

```
Traditional: all requests miss → stampede DB
XFetch: recompute probabilistically before TTL expires
        prob = -beta * delta * log(rand()) > TTL - age
```

Applied to: leaderboard top-100, menu reads, venue schedule.

### 5. Distributed Cron Lock

```
withCronLock("leaderboard-rebuild", 55_000, async () => {
  // only one instance runs this
})

Redis: SET cron:leaderboard-rebuild <token> EX 55 NX
→ returns null if another instance holds lock → skip
```

### 6. Self-Scheduling Pattern (No Cron Overlap)

```
// Instead of cron.schedule("*/5 * * * *", fn) which can overlap:
async function scheduleRepeating(fn, intervalMs) {
  await fn().catch(err => logger.error({ err }));
  setTimeout(() => scheduleRepeating(fn, intervalMs), intervalMs);
}
// Next run starts only after current finishes
```

---

## WebSocket Events

| Event | Direction | Trigger |
|---|---|---|
| `station.status_changed` | server → client | Session start/stop |
| `session.tick` | server → client | Every 60s during session |
| `notification.new` | server → client | BullMQ notification job |
| `rfid.scan` | server → client | ESP32 POST to /rfid/scan |
| `order.status_changed` | server → client | Food order state change |
| `booking.status_changed` | server → client | Booking confirmation/cancel |

Events driven by MongoDB Change Streams — services emit nothing, change streams fan-out.

---

## Security Architecture

### Auth Flow

```
Login → bcrypt verify (12 rounds) → issue RS256 JWT (15min) + refresh token (7d)
Refresh → rotate refresh token → blacklist old JTI in Redis
Logout → blacklist JTI, delete refresh token
```

### 4-Layer Rate Limiting

| Layer | Scope | Limit | Storage |
|---|---|---|---|
| Nginx global | IP | 100 req/min | Nginx shared memory |
| Nginx auth | IP | 10 req/min | Nginx shared memory |
| Token bucket | IP | configurable | Redis Lua script |
| Device | X-Device-ID | 500 req/min | Redis |

### HMAC Webhook Verification (ESP32)

```
ESP32: X-Signature: sha256=HMAC(secret, body)
API:   crypto.timingSafeEqual(computed, received)  ← timing-attack safe
```

---

## Observability Stack

```
Pino structured logs (JSON) → stdout → Loki (production)
prom-client metrics          → GET /metrics → Prometheus scrape
OTel traces (10% head + 100% error tail) → OTLP → Jaeger/Tempo
Sentry error capture         → dashboard

Correlation ID: X-Request-ID propagated → all logs → all queue jobs → all workers
```

### SLO Targets

| Metric | Target |
|---|---|
| Availability | 99.9% |
| p95 latency | < 300ms |
| p99 latency | < 800ms |
| Error rate | < 0.1% |

---

## Hardware Layer (ESP32)

```
ESP32 + RDM6300 RFID Reader
  │
  ├── Scan card → POST /api/v1/cards/scan (HMAC signed)
  │               → Redis lookup member:tier:<rfidCode>
  │               → return tier + access level
  │
  ├── GPIO27 → door unlock relay (on success)
  ├── GPIO27 → buzzer: 1 beep success, 3 beeps fail
  └── Watchdog → auto-reconnect WiFi on drop
```

---

## Deployment

```
docker compose --profile local up   # dev: mongo + redis + api + workers
docker compose --profile obs up     # + prometheus + grafana
docker compose --profile full up    # all services

Production: DigitalOcean Droplet + managed MongoDB Atlas + managed Redis
CI/CD: GitHub Actions → lint → typecheck → unit → integration → build → deploy
```
