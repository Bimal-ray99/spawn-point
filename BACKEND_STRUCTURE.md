# Backend Structure — Complete Codebase Map

```
spawn-point-backend/
│
├── apps/
│   ├── api/                                   # Express REST + WebSocket server
│   │   └── src/
│   │       ├── modules/                       # 13 feature modules (bounded contexts)
│   │       │   ├── auth/
│   │       │   │   ├── auth.routes.ts         # POST /auth/signup|login|refresh|logout
│   │       │   │   ├── auth.service.ts        # bcrypt, JWT issue/verify, Redis blacklist
│   │       │   │   └── auth.schema.ts         # Zod request validation
│   │       │   ├── users/
│   │       │   ├── booking/
│   │       │   │   ├── booking.routes.ts      # CRUD + /lease endpoint
│   │       │   │   ├── booking.service.ts     # Conflict detection, slot lease, saga runner
│   │       │   │   └── booking.saga.ts        # 4-step Saga definition
│   │       │   ├── sessions/
│   │       │   │   ├── session.routes.ts      # POST /start, POST /stop
│   │       │   │   └── session.service.ts     # Redis lock, BullMQ enqueue, OTel context
│   │       │   ├── wallet/
│   │       │   │   └── wallet.service.ts      # Atomic $inc, circuit breaker on payment gw
│   │       │   ├── food/
│   │       │   │   └── food.service.ts        # Active session guard (Redis + DB fallback)
│   │       │   ├── leaderboard/
│   │       │   │   └── leaderboard.service.ts # Redis ZSET read → XFetch → MongoDB fallback
│   │       │   ├── venue/
│   │       │   ├── tournament/
│   │       │   ├── notification/
│   │       │   ├── admin/
│   │       │   │   └── admin.service.ts       # DLQ list/replay, ledger verify
│   │       │   ├── cards/                     # RFID card management
│   │       │   └── support/
│   │       │
│   │       ├── middleware/
│   │       │   ├── auth.middleware.ts         # RS256 JWT verify, attach req.user
│   │       │   ├── role.middleware.ts         # requireRole('admin'|'staff'|'member')
│   │       │   ├── idempotency.middleware.ts  # X-Idempotency-Key → Redis cache
│   │       │   ├── rate-limit.middleware.ts   # Token bucket (Redis Lua) + device rate limit
│   │       │   ├── hmac.middleware.ts         # ESP32 webhook: X-Signature verify
│   │       │   └── etag.middleware.ts         # SHA256 ETag + 304 on If-None-Match
│   │       │
│   │       ├── shared/
│   │       │   ├── errors.ts                  # AppError class + ErrorCodes enum
│   │       │   ├── response.ts                # sendSuccess() / sendError() envelope
│   │       │   ├── logger.ts                  # Pino + correlation ID child logger
│   │       │   ├── metrics.ts                 # prom-client: histogram, gauge, counter
│   │       │   ├── tracing.ts                 # OTel SDK init, getTracer() export
│   │       │   ├── websocket.ts               # WS server + Redis pub/sub fan-out
│   │       │   ├── circuit-breaker.ts         # Redis-backed CLOSED/OPEN/HALF-OPEN FSM
│   │       │   ├── saga.ts                    # Generic Saga<TContext> orchestrator
│   │       │   └── change-streams.ts          # MongoDB change streams → WS emit
│   │       │
│   │       ├── queues/
│   │       │   └── index.ts                   # BullMQ queue instances (producer side)
│   │       │
│   │       ├── cron/
│   │       │   └── session.cron.ts            # Abandoned session cleanup (withCronLock)
│   │       │
│   │       └── server.ts                      # Bootstrap: tracing → express → routes → WS → change streams
│   │
│   └── workers/                               # BullMQ consumers (separate Node process)
│       └── src/
│           ├── workers/
│           │   ├── session-events.worker.ts   # Billing, leaderboard, streak, referral
│           │   ├── notification.worker.ts     # FCM push + Resend email (circuit breaker)
│           │   └── dlq.worker.ts              # Dead Letter Queue — log + alert + persist
│           │
│           ├── jobs/
│           │   ├── cron.jobs.ts               # scheduleRepeating + withCronLock pattern
│           │   └── outbox.poller.ts           # Poll OutboxEvents every 5s → BullMQ publish
│           │
│           ├── queues/
│           │   └── index.ts                   # Same queue names, consumer side
│           │
│           └── shared/
│               ├── logger.ts                  # Pino with worker context
│               └── metrics.ts                 # Queue depth + job duration metrics
│
├── packages/                                  # Shared packages — built before apps
│   │
│   ├── types/                                 # Shared TypeScript interfaces + enums
│   │   └── src/
│   │       └── index.ts                       # BookingStatus, TierEnum, WalletTxType, etc.
│   │
│   ├── db/                                    # Mongoose models + connection management
│   │   └── src/
│   │       ├── models/                        # 34 collections, one file each
│   │       │   ├── user.model.ts
│   │       │   ├── booking.model.ts           # version field for OCC
│   │       │   ├── wallet.model.ts
│   │       │   ├── transaction.model.ts       # account_type for double-entry bookkeeping
│   │       │   ├── session.model.ts
│   │       │   ├── loyalty-streak.model.ts    # last_processed_session_id (idempotency guard)
│   │       │   ├── outbox-event.model.ts      # TTL index on publishedAt
│   │       │   ├── saga-log.model.ts
│   │       │   └── ... (26 more models)
│   │       ├── seeds/
│   │       │   └── index.ts                   # runSeeds() — venues, gaming systems, tiers
│   │       ├── logger.ts
│   │       └── index.ts                       # connectDB() with 3 pools + export * from models
│   │
│   ├── redis/                                 # Redis client + distributed primitives
│   │   └── src/
│   │       ├── index.ts                       # createClient, acquireLock, releaseLock, withCronLock
│   │       ├── pubsub.ts                      # Separate publish/subscribe clients
│   │       └── cache.ts                       # getWithXFetch<T>() — stampede prevention
│   │
│   └── config/                                # Environment-aware configuration
│       └── src/
│           └── index.ts                       # resolveMongoUri(), resolveDbName(), LOG_LEVEL
│
├── tests/
│   ├── integration/                           # supertest + mongodb-memory-server
│   │   ├── auth.test.ts                       # register → login → refresh → logout
│   │   ├── booking.test.ts                    # create → double-book rejected → cancel → refund
│   │   └── saga.test.ts                       # compensation on step 3 failure
│   └── stress/                                # k6 load tests
│       ├── booking-thundering-herd.k6.ts      # 100 VUs, same slot — assert 1 winner
│       ├── wallet-concurrent-debit.k6.ts      # 50 debits on balance=100, never negative
│       ├── websocket-storm.k6.ts              # 1000 WS connections + broadcast, 0 drops
│       ├── session-burst.k6.ts                # 50 sessions end simultaneously
│       └── rfid-flood.k6.ts                   # 500 scans/min, Redis hit rate > 95%
│
├── infra/
│   ├── nginx/
│   │   └── nginx.conf                         # TLS, rate limit zones, WS upgrade
│   └── prometheus.yml                         # Scrape config for /metrics
│
├── docker/
│   └── docker-compose.yml                     # Profiles: local | dev | obs | full
│                                              # Services: api, workers, mongo, redis,
│                                              #           prometheus, grafana
├── .github/
│   └── workflows/
│       └── ci.yml                             # lint → typecheck → unit → integration → build
│
├── package.json                               # workspaces: ["apps/*", "packages/*"]
├── tsconfig.base.json
├── README.md
├── CLAUDE.md                                  # AI assistant instructions
└── docs/
    ├── SYSTEM_DESIGN.md                       # Architecture deep-dive
    ├── ADR.md                                 # 10 architecture decision records
    ├── PATTERNS.md                            # 14 distributed systems patterns
    ├── BACKEND_STRUCTURE.md                   # This file
    ├── api-endpoints.md                       # All 60+ endpoints documented
    └── build-guide.md                         # Step-by-step implementation pseudocode
```

---

## Module Boundaries

```
packages/types     ← no imports from other packages
packages/config    ← imports: types
packages/redis     ← imports: types, config
packages/db        ← imports: types, config
apps/api           ← imports: types, config, redis, db
apps/workers       ← imports: types, config, redis, db
```

Strict DAG — no circular dependencies. Build order enforced in `package.json`:
```
types → redis → db → api/workers (parallel)
```

---

## Key File Roles

| File | Role |
|---|---|
| `packages/db/src/index.ts` | Bulkhead pools, global query timeout, model exports |
| `apps/api/src/shared/saga.ts` | Generic Saga orchestrator |
| `apps/api/src/shared/circuit-breaker.ts` | Redis FSM: CLOSED → OPEN → HALF-OPEN |
| `apps/api/src/shared/websocket.ts` | WS server + Redis pub/sub fan-out |
| `apps/api/src/shared/change-streams.ts` | MongoDB → WS (decoupled from services) |
| `packages/redis/src/cache.ts` | XFetch stampede prevention |
| `packages/redis/src/pubsub.ts` | Separate pub/sub Redis connections |
| `apps/workers/src/jobs/outbox.poller.ts` | Guaranteed event delivery |
| `apps/api/src/middleware/idempotency.middleware.ts` | Duplicate request suppression |
| `apps/api/src/middleware/hmac.middleware.ts` | ESP32 webhook authentication |
