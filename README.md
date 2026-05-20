# Spawnpoint — Production-Grade Gaming Café Backend

> TypeScript monorepo · Express · MongoDB Atlas · Redis · BullMQ · ESP32 RFID  
> Built to demonstrate distributed systems patterns at FAANG engineering depth.

![CI](https://github.com/bimalray/spawn-point-backend/actions/workflows/ci.yml/badge.svg)
![TypeScript](https://img.shields.io/badge/TypeScript-5.4-blue)
![Node](https://img.shields.io/badge/Node.js-20_LTS-green)
![License](https://img.shields.io/badge/license-Private-red)

---

## What This Is

Spawnpoint is a full-stack backend for a gaming café — real-time session billing, RFID loyalty cards,
advance bookings, food orders, tournaments, and leaderboards. Built as a portfolio project to demonstrate
that I can design and implement systems at production quality, not just tutorials.

**The goal wasn't to build a gaming café app. The goal was to answer:**  
*"Can you implement what Netflix, Uber, and Stripe do at a system level — and explain every decision?"*

The answer is 37 distributed systems patterns, 34 MongoDB collections, 60+ REST endpoints,
WebSocket real-time events, k6 stress tests that assert correctness (not just speed),
and a hardware layer (ESP32 RFID readers) talking to the API over HMAC-signed webhooks.

---

## Architecture

```
┌─────────────────────────────────────────────────────────────────────┐
│              Clients · Web App · Mobile · ESP32 RFID                │
└──────────────────────────┬──────────────────────────────────────────┘
                           │ HTTPS / WebSocket / HMAC
                           ▼
┌─────────────────────────────────────────────────────────────────────┐
│                    Nginx — Edge Layer                                │
│         TLS termination · 4-zone rate limiting · HTTP→HTTPS         │
└──────────────────────────┬──────────────────────────────────────────┘
                           │
         ┌─────────────────┴──────────────────┐
         ▼                                    ▼
┌──────────────────┐                ┌──────────────────────┐
│   apps/api       │                │   apps/workers        │
│   Express REST   │  ←─ Redis  ──→ │   BullMQ consumers   │
│   + WebSocket    │  ←─ BullMQ ──→ │   session-events     │
│                  │                │   notifications       │
│   17 modules     │                │   outbox-poller       │
│   60+ endpoints  │                │   dead-letter         │
│   Middleware:    │                │   7 cron jobs         │
│   JWT · RBAC     │                └──────────────────────┘
│   Idempotency    │
│   Circuit Breaker│
│   Rate Limiting  │
└────────┬─────────┘
         │
┌────────▼──────────────────────────────────────────────────┐
│                      Data Layer                            │
│                                                            │
│  MongoDB Atlas (34 collections)    Redis                   │
│  ┌─────────────────────────────┐   ┌─────────────────┐    │
│  │ 3 isolated connection pools │   │ 14 key patterns │    │
│  │ transactional  (10 conn)    │   │ Pub/Sub         │    │
│  │ analytics      (3 conn,     │   │ Lua scripts     │    │
│  │                secondary)   │   │ Bloom filter    │    │
│  │ background     (2 conn)     │   │ HyperLogLog     │    │
│  └─────────────────────────────┘   └─────────────────┘    │
└────────────────────────────────────────────────────────────┘
```

---

## Why These Patterns

| Pattern | Problem It Solves | Where |
|---|---|---|
| **Saga Orchestrator** | Booking spans 4 steps across DB + Redis. Step 3 fails → steps 1–2 already executed. Data corrupt without compensation. | `booking.saga.ts` |
| **Outbox Pattern** | Service writes to DB, crashes before `queue.add()`. Event lost. User billed, session worker never ran. | `outbox.poller.ts` |
| **Circuit Breaker** | FCM goes down. Every notification waits 30s to timeout. Threads pile up. API slows for everyone. | `circuit-breaker.ts` |
| **Bulkhead Pools** | Leaderboard aggregation takes 3s. Shares connection pool with bookings. p99 spikes to 15s. | `packages/db/index.ts` |
| **Idempotency Middleware** | User double-taps "Book". Two requests land. Both succeed. Double-charged. | `idempotency.middleware.ts` |
| **Slot Lease** | User sees slot free, fills form for 2 minutes, submits — slot taken. TOCTOU race on availability check. | `booking.service.ts` |
| **XFetch Cache** | Leaderboard TTL expires at peak hour. 500 users all miss. All hit MongoDB simultaneously. | `packages/redis/cache.ts` |
| **Distributed Cron Lock** | Two API instances both run midnight leaderboard rebuild. Double writes, duplicate notifications. | `packages/redis/index.ts` |
| **Self-Scheduling Pattern** | `cron.schedule("*/5 * * * *", fn)` fires at wall-clock. 6-min job overlaps next run. Race conditions. | `cron.jobs.ts` |
| **Redis Pub/Sub WS** | Session ends on Instance B. User's WebSocket is on Instance A. Event dropped silently. | `websocket.ts` |
| **Token Theft Detection** | Stolen refresh token reused. Attacker gets new access token. Original user doesn't know. | `auth.service.ts` |
| **Double-Entry Ledger** | Worker bug debits wallet without matching credit. Financial inconsistency undetected for weeks. | `wallet.service.ts` |
| **Optimistic Concurrency** | Admin cancels booking. User extends it simultaneously. Last write wins. Cancelled booking un-cancelled. | `booking.service.ts` |
| **Change Streams** | Every service called `emitToUser()` directly — business logic coupled to WebSocket infrastructure. | `change-streams.ts` |
| **HMAC Device Auth** | ESP32 on café LAN can POST to `/cards/scan`. Any device on network can forge scans to unlock doors. | `hmac.middleware.ts` |

[→ See all 37 patterns with problem/solution/tradeoff in PATTERNS.md](docs/PATTERNS.md)

---

## Technical Decisions

Selected ADRs — the ones with the most interesting tradeoffs:

**MongoDB over PostgreSQL** — Booking documents contain nested slot details, tier discounts, and coupon
metadata that would span 5 relational tables. Schema evolved 12 times during implementation with zero
migrations. Tradeoff: no foreign key enforcement (Mongoose refs + app-level validation), no free ACID
across documents (Saga handles this).

**Separate worker process** — `apps/workers` is a distinct Node.js process from `apps/api`. API crash
doesn't kill in-flight billing jobs. Workers can be scaled independently. Shared `packages/db` means
model changes propagate to both without duplication.

**RS256 over HS256** — Symmetric key means any service that verifies tokens can also issue them.
A compromised worker process could mint arbitrary access tokens. RS256: private key lives only in API,
public key distributed to workers. Compromise of worker = cannot issue tokens.

**MongoDB Change Streams over service-level emit** — Services were calling `emitToUser()` directly.
Every service test needed a WS mock. Decoupled: services write to DB only. Change streams watch
collections and fan-out WS events. Services became pure, testable, and WS-agnostic.

**Outbox + Saga together** — Outbox guarantees the event leaves (at-least-once delivery to BullMQ).
Saga guarantees the transaction is consistent (compensates on failure). Neither alone is sufficient.
Outbox without Saga: consistent event delivery but no compensation. Saga without Outbox: compensation
works but events can be lost on crash between DB write and queue enqueue.

[→ All 10 decisions with context and consequences in ADR.md](docs/ADR.md)

---

## Numbers

| Dimension | Count |
|---|---|
| REST endpoints | 60+ |
| WebSocket events | 6 |
| MongoDB collections | 34 |
| Redis key patterns | 14 |
| Distributed systems patterns | 37 |
| BullMQ queues | 5 + DLQ |
| Cron jobs | 7 |
| Middleware layers per request | 8 |
| k6 stress test scenarios | 5 |
| Architecture Decision Records | 10 |

---

## Stress Tests

All 5 k6 scenarios assert **correctness** — not just latency. CI fails if thresholds breach.

| Scenario | VUs | Key Assertion |
|---|---|---|
| Booking thundering herd | 100 concurrent, same slot | Exactly 1 booking created, ≥99 get `409 BOOKING_CONFLICT` |
| Wallet concurrent debit | 50 debits, balance=100 | Balance never goes negative (atomic MongoDB `$inc`) |
| WebSocket storm | 1000 connections + broadcast | 0 message drops, p95 latency < 200ms |
| Session burst | 50 sessions end simultaneously | All leaderboard increments applied correctly |
| RFID card flood | 500 scans/min, 10 cards | Redis Bloom filter hit rate > 95% |

---

## Module Overview

| Module | Prefix | Notable Design |
|---|---|---|
| Auth | `/api/v1/auth` | RS256 JWT, refresh theft detection, JWKS rotation |
| Wallet | `/api/v1/wallet` | Atomic `$inc`, double-entry ledger, event-sourcing audit |
| Booking | `/api/v1/bookings` | Saga-orchestrated, slot lease, OCC cancel, waitlist FIFO |
| Sessions | `/api/v1/sessions` | Redis lock, SSE live billing tick, outbox event |
| Food | `/api/v1/food` | ETag + XFetch cached menu, in-session order, kitchen queue |
| Leaderboard | `/api/v1/leaderboard` | CQRS read model, Redis ZSET primary path, XFetch |
| Cards | `/api/v1/cards` | RFID register/block, Bloom filter scan, HMAC device auth |
| Admin | `/api/v1/admin` | DLQ replay, feature flags, ledger verify, wallet audit |
| Device | `/api/v1/device` | ESP32 only — HMAC auth, no JWT, 500 scan/min rate limit |

---

## Observability

```
Pino JSON logs     → correlation ID on every log line (API + workers share req_id)
prom-client        → GET /metrics → Prometheus → Grafana dashboard
OTel tracing       → 1% baseline + 100% errors + 100% slow requests (adaptive sampler)
Sentry             → error capture + release tracking
```

SLO targets: 99.9% availability · p95 < 300ms · p99 < 800ms · error rate < 0.1%

---

## Hardware Layer

```
ESP32 + RDM6300 RFID Reader (125 kHz)
  │
  ├── Scan card → POST /api/v1/device/scan
  │   Headers: X-Device-ID, X-Signature: sha256=<HMAC-SHA256>
  │   API: timingSafeEqual verify → Redis Bloom filter check
  │        → member tier lookup → door GPIO signal
  │
  ├── Door unlock relay on GPIO pin (valid tier)
  ├── Buzzer: 1 beep = success, 3 beeps = fail/unknown card
  └── Watchdog: auto-reconnect WiFi on drop
```

---

## Tech Stack

| Layer | Technology |
|---|---|
| Runtime | Node.js 20 LTS |
| Language | TypeScript 5.4 (strict mode) |
| HTTP | Express 4.19 |
| Database | MongoDB Atlas · Mongoose 8 |
| Cache | Redis 4.6 (node-redis) |
| Queues | BullMQ 5.4 |
| Auth | JWT RS256 · jsonwebtoken 9 |
| Real-time | WebSocket (ws 8) · SSE |
| Logging | Pino 8 · pino-http |
| Metrics | prom-client → Prometheus → Grafana |
| Tracing | OpenTelemetry SDK 0.215 |
| Errors | Sentry 8 |
| Notifications | Firebase Admin (FCM) · Resend |
| Validation | Zod 3.22 |
| Tests | Vitest · supertest · mongodb-memory-server · k6 |
| Containers | Docker · Docker Compose |
| CI/CD | GitHub Actions |
| Hardware | ESP32 · RDM6300 RFID |

---

## Documentation

| Doc | Contents |
|---|---|
| [SYSTEM_DESIGN.md](docs/SYSTEM_DESIGN.md) | Full architecture: request lifecycle, DB pools, Redis keys, BullMQ, security, observability |
| [PATTERNS.md](docs/PATTERNS.md) | 14 patterns: problem → solution → code → tradeoff |
| [ADR.md](docs/ADR.md) | 10 architectural decisions with context and consequences |
| [BACKEND_STRUCTURE.md](docs/BACKEND_STRUCTURE.md) | Full file tree with inline explanation of every key file |
| [api-endpoints.md](docs/api-endpoints.md) | All 60+ endpoints documented with request/response shapes |

---

## Quick Start

```bash
git clone https://github.com/bimalray/spawn-point-backend
cd spawn-point-backend
npm install

# Start full local stack (API + workers + Mongo + Redis + Prometheus + Grafana)
docker compose -f docker/docker-compose.yml --env-file .env up --build

# API:        http://localhost:8080
# Health:     http://localhost:8080/api/v1/health
# Metrics:    http://localhost:8080/metrics
# Grafana:    http://localhost:3001  (admin / admin)
```
[LinkedIn](https://www.linkedin.com/in/bimal-ray) · [[Medium articles on the architecture decisions behind this project](https://medium.com/@bimal.ray99/i-built-a-side-project-then-real-users-broke-it-ea7449c59362)]
