# Spawnpoint Build Guide
Step-by-step pseudocode. Complete each file, then move to the next.

---

## PHASE 1 — Shared Utilities ✅

### 1. `apps/api/src/shared/errors.ts` ✅
```
class AppError extends Error
  constructor(code, message, statusCode = 400)

const ErrorCodes = { ERR_UNAUTHORIZED, ERR_WALLET_INSUFFICIENT, ... }
```

### 2. `apps/api/src/shared/response.ts` ✅
```
sendSuccess(res, data, statusCode = 200)
  → res.json({ success: true, data, meta: { request_id, timestamp } })

sendError(res, code, message, statusCode)
  → res.json({ success: false, error: { code, message }, meta })
```

### 3. `apps/api/src/shared/logger.ts` ✅
```
logger = pino({ level, redact: [...sensitive fields], transport: pino-pretty in dev })
requestLogger = pinoHttp({ logger })
export { logger, requestLogger }
```

---

## PHASE 2 — Express App Bootstrap

### 4. `apps/api/src/app.ts`
```
create express app
app.use(requestId())           ← attaches req.id
app.use(requestLogger)         ← pino-http
app.use(helmet())              ← security headers
app.use(cors())                ← allow origins from env
app.use(express.json())        ← parse body

register all route modules:
  app.use('/api/v1/auth',         authRoutes)
  app.use('/api/v1/users',        userRoutes)
  app.use('/api/v1/wallet',       walletRoutes)
  app.use('/api/v1/bookings',     bookingRoutes)
  app.use('/api/v1/food',         foodRoutes)
  app.use('/api/v1/coupons',      couponRoutes)
  app.use('/api/v1/leaderboard',  leaderboardRoutes)
  app.use('/api/v1/tournaments',  tournamentRoutes)
  app.use('/api/v1/venues',       venueRoutes)
  app.use('/api/v1/seller',       sellerRoutes)
  app.use('/api/v1/device',       deviceRoutes)
  app.use('/api/v1/admin',        adminRoutes)
  app.use('/api/v1/notifications',notificationRoutes)

global error handler (last middleware):
  if err instanceof AppError → sendError(res, err.code, err.message, err.statusCode)
  else → sendError(res, ERR_INTERNAL, 'Internal server error', 500)

export app
```

### 5. `apps/api/src/server.ts`
```
import app
connectDB(MONGODB_URI)
connectRedis(REDIS_URL)
app.listen(PORT, () => logger.info(`Server running on ${PORT}`))
```

---

## PHASE 3 — Middleware (update existing for Express)

### 6. `apps/api/src/middleware/jwt.middleware.ts` (update)
```
jwtMiddleware: RequestHandler
  get Bearer token from req.headers.authorization
  jwt.verify(token, JWT_PUBLIC_KEY, { algorithms: ['RS256'] })
  attach payload to req.user
  on TokenExpiredError → throw AppError(ERR_TOKEN_EXPIRED, 401)
  on error → throw AppError(ERR_TOKEN_INVALID, 401)
```

### 7. `apps/api/src/middleware/rbac.middleware.ts` (update)
```
requireRole(...roles): RequestHandler
  if !req.user → throw AppError(ERR_UNAUTHORIZED, 401)
  if role not in roles → throw AppError(ERR_FORBIDDEN, 403)
```

### 8. `apps/api/src/middleware/api-key.middleware.ts` (update)
```
apiKeyMiddleware: RequestHandler
  get x-api-key and x-device-id from headers
  HMAC-SHA256(apiKey, DEVICE_API_KEY_SALT) → keyHash
  lookup device in DB, compare keyHash
  check Redis device:rate:{deviceId} → if > 10/min → 429
  attach req.deviceId
```

---

## PHASE 4 — Database Models

### 9. `packages/db/src/models/venue.model.ts`
```
IVenue:
  name, address, city, state, pincode
  phone, email
  manager_id → ref User
  is_active: boolean
  open_time, close_time: string ("10:00")
  amenities: string[]
  images: string[]
  timestamps

indexes: is_active, manager_id
```

### 10. `packages/db/src/models/tier.model.ts`
```
ITier:
  name: "Bronze"|"Silver"|"Gold"|"Platinum"
  level: 1|2|3|4
  min_lifetime_points: number
  discount_percentage: number
  hourly_rate_multiplier: number
  color: string
  benefits: string[]

index: level (unique)
```

### 11. `packages/db/src/models/gaming-system.model.ts`
```
IPricingRule (subdoc, no _id):
  tier_level: number
  hourly_rate: number
  discount_percentage: number

IGamingSystem:
  venue_id → ref Venue
  name: string          ("PC-01")
  type: 'pc'|'ps'|'vr'|'cricket'|'racing'
  specifications: string
  is_active: boolean
  is_maintenance: boolean
  base_hourly_rate: number
  pricing_rules: IPricingRule[]   ← embedded, max 4
  rfid_tag?: string
  timestamps

indexes: venue_id + type, is_active
```

### 12. `packages/db/src/models/group-booking.model.ts`
```
IGroupBooking:
  venue_id → ref Venue
  created_by → ref User
  session_ids: ObjectId[]  → ref Session
  total_cost: number
  status: 'active'|'completed'|'cancelled'
  timestamps
```

### 13. `packages/db/src/models/waitlist-entry.model.ts`
```
IWaitlistEntry:
  user_id → ref User
  system_id → ref GamingSystem
  venue_id → ref Venue
  slot_iso: string
  notified_at?: Date
  expires_at: Date

TTL index: expires_at (expireAfterSeconds: 0)
unique index: user_id + system_id + slot_iso
```

### 14. `packages/db/src/models/food-item.model.ts`
```
IFoodItem:
  venue_id → ref Venue
  name, description?, category
  price_points: number
  image_url?: string
  is_available: boolean
  prep_time_minutes: number
  timestamps

index: venue_id + is_available
```

### 15. `packages/db/src/models/food-order.model.ts`
```
IFoodOrderItem (subdoc, no _id):
  food_item_id, quantity
  unit_price_points, subtotal_points   ← SNAPSHOT at order time
  item_name_snapshot                   ← SNAPSHOT at order time

IFoodOrder:
  user_id → ref User
  venue_id → ref Venue
  session_id → ref Session   ← must have active session
  items: IFoodOrderItem[]    ← embedded
  total_points: number
  status: 'pending'|'preparing'|'delivered'|'cancelled'
  ordered_at: Date
  delivered_at?: Date
  createdAt only (no updatedAt)

indexes: user_id + status, session_id
```

### 16. `packages/db/src/models/point-pack.model.ts`
```
IPointPack:
  name, points, price_inr
  bonus_points: number
  is_active: boolean
  valid_from?, valid_until?
  timestamps
```

### 17. `packages/db/src/models/pack-purchase.model.ts`
```
IPackPurchase:
  user_id → ref User
  pack_id → ref PointPack
  points_credited: number
  amount_paid_inr: number
  payment_gateway: string
  payment_id: string
  status: 'pending'|'success'|'failed'|'refunded'
  createdAt only

index: user_id, payment_id (unique)
```

### 18. `packages/db/src/models/tournament.model.ts`
```
IBracketMatch (subdoc):
  round, match_number
  player1_id?, player2_id?
  winner_id?, score?

ITournament:
  venue_id → ref Venue
  name, description?
  game_type: 'pc'|'ps'|'vr'|'cricket'|'racing'
  status: 'draft'|'registration_open'|'registration_closed'|'ongoing'|'completed'|'cancelled'
  entry_fee_points, max_participants, prize_pool_points
  prize_distribution: [{ position, points }]
  registration_start, registration_end
  tournament_start?, tournament_end?
  brackets: IBracketMatch[]   ← embedded
  winner_id?
  created_by → ref User
  timestamps

index: status, venue_id + status
```

### 19. `packages/db/src/models/tournament-participant.model.ts`
```
ITournamentParticipant:
  tournament_id → ref Tournament
  user_id → ref User
  registration_fee_paid: number
  status: 'registered'|'eliminated'|'winner'|'refunded'
  final_position?, prize_awarded?
  registered_at: Date

unique index: tournament_id + user_id
index: tournament_id + status
```

### 20. `packages/db/src/models/coupon.model.ts`
```
ICoupon:
  code: string (unique, uppercase)
  description?
  discount_type: 'percentage'|'fixed'
  discount_value: number
  min_booking_value?, max_discount_points?
  applicable_to: 'booking'|'food'|'both'
  max_uses, used_count (default 0), per_user_limit
  valid_from, valid_until
  is_active: boolean
  created_by → ref User
  timestamps

index: code, is_active + valid_until
```

### 21. `packages/db/src/models/coupon-usage.model.ts`
```
ICouponUsage:
  coupon_id → ref Coupon
  user_id → ref User
  reference_id → ObjectId
  reference_type: 'session'|'food_order'
  discount_applied: number
  createdAt only

index: coupon_id + user_id
```

### 22. `packages/db/src/models/flash-sale.model.ts`
```
IFlashSale:
  name, description?
  venue_id? → ref Venue   (null = all venues)
  system_types?: string[]
  discount_percentage: number
  starts_at, ends_at: Date
  is_active: boolean
  created_by → ref User
  timestamps

index: is_active + starts_at + ends_at
```

### 23. `packages/db/src/models/leaderboard-entry.model.ts`
```
ILeaderboardEntry:
  user_id → ref User
  venue_id? → ref Venue
  period: 'weekly'|'monthly'|'all_time'
  period_key: string   ("2024-W01", "2024-01", "all")
  points: number
  rank: number
  sessions_count: number
  updatedAt only

unique index: user_id + period + period_key
index: period + period_key + rank
```

### 24. `packages/db/src/models/loyalty-streak.model.ts`
```
ILoyaltyStreak:
  user_id → ref User (unique)
  current_streak: number
  longest_streak: number
  last_activity_date: Date
  streak_broken_at?: Date
  milestone_rewards_claimed: number[]
  updatedAt only
```

### 25. `packages/db/src/models/referral-reward.model.ts`
```
IReferralReward:
  referrer_id → ref User
  referred_id → ref User (unique)
  reward_points: number
  status: 'pending'|'credited'|'expired'
  trigger: 'signup'|'first_booking'
  credited_at?: Date
  createdAt only

index: referrer_id, status
```

### 26. `packages/db/src/models/announcement.model.ts`
```
IAnnouncement:
  venue_id? → ref Venue   (null = global)
  title, body
  type: 'info'|'warning'|'promotion'|'event'
  starts_at, ends_at?
  is_active: boolean
  created_by → ref User
  timestamps

index: venue_id + is_active, starts_at
```

### 27. `packages/db/src/models/announcement-view.model.ts`
```
IAnnouncementView:
  announcement_id → ref Announcement
  user_id → ref User
  viewed_at: Date

unique index: announcement_id + user_id
```

### 28. `packages/db/src/models/notification.model.ts`
```
INotification:
  user_id → ref User
  title, body
  channel: 'push'|'email'|'sms'|'in_app'
  type: string   ('booking_confirmed', 'streak_broken', etc.)
  data?: object
  is_read: boolean (default false)
  read_at?: Date
  sent_at: Date
  createdAt only

index: user_id + is_read, user_id + createdAt desc
```

### 29. `packages/db/src/models/support-ticket.model.ts`
```
ITicketMessage (subdoc):
  sender_id → ref User
  sender_role: 'client'|'seller'|'admin'
  body: string
  sent_at: Date

ISupportTicket:
  user_id → ref User
  venue_id? → ref Venue
  subject, category: 'billing'|'booking'|'food'|'technical'|'other'
  status: 'open'|'in_progress'|'resolved'|'closed'
  priority: 'low'|'medium'|'high'
  messages: ITicketMessage[]   ← embedded
  assigned_to? → ref User
  resolved_at?
  timestamps

index: user_id + status, assigned_to + status
```

### 30. `packages/db/src/models/user-card.model.ts`
```
IUserCard:
  user_id → ref User
  rfid_uid: string (unique)
  status: 'active'|'blocked'|'lost'|'replaced'|'deactivated'
  issued_at: Date
  blocked_at?, block_reason?
  replaced_by? → ref UserCard
  issued_by → ref User
  timestamps

index: rfid_uid (unique), user_id + status
```

### 31. `packages/db/src/models/card-issuance-log.model.ts`
```
ICardIssuanceLog:
  card_id → ref UserCard
  user_id → ref User
  action: 'issued'|'blocked'|'unblocked'|'lost'|'replaced'
  performed_by → ref User
  reason?
  createdAt only

index: card_id, user_id
```

### 32. `packages/db/src/models/venue-checkin.model.ts`
```
IVenueCheckin:
  user_id → ref User
  venue_id → ref Venue
  method: 'qr'|'rfid'|'manual'
  checked_in_at: Date
  checked_out_at?
  session_id? → ref Session
  device_id?: string
  createdAt only

index: user_id + checked_in_at, venue_id + checked_in_at
```

### 33. `packages/db/src/models/seller-credit-log.model.ts`
```
ISellerCreditLog:
  seller_id → ref User
  user_id → ref User
  card_id? → ref UserCard
  rfid_uid?: string
  points_credited: number
  reason: string
  venue_id → ref Venue
  createdAt only

index: seller_id, user_id, venue_id + createdAt
```

### 34. `packages/db/src/models/maintenance-request.model.ts`
```
IMaintenanceRequest:
  system_id → ref GamingSystem
  venue_id → ref Venue
  reported_by → ref User
  issue: string
  status: 'open'|'in_progress'|'resolved'
  resolved_by? → ref User
  resolved_at?
  timestamps

index: venue_id + status, system_id
```

### 35. `packages/db/src/models/user-session.model.ts`
```
IUserSession:  (auth/device session tracking — not gaming session)
  user_id → ref User
  device_info?: string
  ip_address?: string
  user_agent?: string
  last_active_at: Date
  is_active: boolean
  createdAt only

TTL index: last_active_at (expireAfterSeconds: 30 days in seconds)
index: user_id + is_active
```

### 36. `packages/db/src/models/app-config.model.ts`
```
IAppConfig:
  key: string (unique)    ('referral_config', 'streak_milestones', etc.)
  value: object (Mixed)
  description?
  updatedAt only

index: key (unique)
```

---

## PHASE 5 — Update DB Index

### 37. `packages/db/src/index.ts` (add 3 exports)
```
add:
  export * from './models/user-session.model'
  export * from './models/maintenance-request.model'
  export * from './models/announcement-view.model'
```

---

## PHASE 6 — Seeds

### 38. `packages/db/src/seeds/index.ts`
```
connectDB()

seed Tiers (4 docs):
  { level:1, name:'Bronze', min_lifetime_points:0,    discount:0,  multiplier:1.0 }
  { level:2, name:'Silver', min_lifetime_points:1000,  discount:10, multiplier:0.9 }
  { level:3, name:'Gold',   min_lifetime_points:5000,  discount:20, multiplier:0.8 }
  { level:4, name:'Platinum',min_lifetime_points:15000,discount:30, multiplier:0.7 }

seed AppConfig (2 docs):
  { key: 'referral_config',   value: { referrer_points: 100, referred_points: 50, trigger: 'first_booking' } }
  { key: 'streak_milestones', value: { milestones: [7,30,100], bonus_points: [50,200,500] } }

use upsert (updateOne + upsert:true) so seeds are idempotent

disconnect()
```

---

## PHASE 7 — Workers Bootstrap

### 39. `apps/workers/src/shared/logger.ts`
```
same as api logger.ts but prefix '[WORKER]'
export logger
```

### 40. `apps/workers/src/index.ts`
```
connectDB(MONGODB_URI)
connectRedis(REDIS_URL)

import and start all workers:
  notificationWorker
  walletEventsWorker
  sessionEventsWorker
  tournamentWorker
  reportExportWorker

startCronJobs()

logger.info('All workers running')

handle process SIGTERM → gracefully close all workers
```

---

## PHASE 8 — Missing Workers

### 41. `apps/workers/src/workers/wallet-events.worker.ts`
```
Worker<WalletEventJob>('wallet-events', concurrency: 5)

job types:
  'tier_upgrade':
    fetch user lifetime_points
    find matching tier (Tier.findOne { min_lifetime_points: { $lte: points } } sort level desc)
    if tier changed → update user.tier_id
    enqueue notification job (tier_push)

  'referral_reward':
    find referral_reward doc by referred_id
    if status pending → atomically credit points to referrer wallet
    create Transaction doc
    mark referral_reward as credited
    enqueue notification job

  'low_balance_alert':
    if wallet.balance < low_balance_threshold
    enqueue notification job (wallet_push)

on failed → logger.error + Sentry
```

### 42. `apps/workers/src/workers/tournament.worker.ts`
```
Worker<TournamentJob>('tournament', concurrency: 2)

job types:
  'generate_brackets':
    fetch registered participants
    Fisher-Yates shuffle participant list
    build bracket rounds (pair up players, byes for odd counts)
    save brackets[] to tournament doc
    update status → 'ongoing'

  'distribute_prizes':
    fetch final standings from brackets
    for each prize position → credit points to winner wallet
    create Transaction docs
    update participant status → 'winner'
    enqueue notification jobs

  'refund_fees':
    fetch all participants with status 'registered'
    for each → credit entry_fee_points back to wallet
    create Transaction docs
    update participant status → 'refunded'

on failed → logger.error + Sentry
```

### 43. `apps/workers/src/workers/report-export.worker.ts`
```
Worker<ReportExportJob>('report-export', concurrency: 2)

job types:
  'csv':
    fetch data based on job.data.report_type and date range
    generate CSV string
    // TODO: upload to CDN (S3/Cloudflare R2)
    enqueue notification to admin with download link

  'pdf':
    // stub — same shape as csv

on failed → logger.error
```

---

## PHASE 9 — Module Implementations

> Pattern for every module:
> `schema.ts` → Zod shapes
> `service.ts` → business logic, DB calls
> `controller.ts` → parse schema, call service, sendSuccess
> `routes.ts` → wire middleware + controller

---

### 44. `user` module

**schema.ts**
```
updateProfileSchema: { name?, avatar_url?, preferred_system? }
updateNotifPrefsSchema: { booking_push?, wallet_push?, ... all 11 booleans optional }
```

**service.ts**
```
getProfile(userId) → User.findById + populate tier
updateProfile(userId, data) → User.findByIdAndUpdate
getQrToken(userId) → User.findById → return qr_token + qr_expires_at
refreshQrToken(userId) → generate new token, update user, invalidate old Redis qr: key
getNotifications(userId, page, limit) → Notification.find paginated
updateNotifPrefs(userId, prefs) → User.findByIdAndUpdate notification_preferences
getReferralStats(userId) → count ReferralReward by referrer_id
```

**routes.ts**
```
GET  /profile              jwtMiddleware → getProfile
PUT  /profile              jwtMiddleware → updateProfile
GET  /qr-token             jwtMiddleware → getQrToken
POST /qr-token/refresh     jwtMiddleware → refreshQrToken
GET  /notifications        jwtMiddleware → getNotifications
PUT  /notification-prefs   jwtMiddleware → updateNotifPrefs
GET  /referral-stats       jwtMiddleware → getReferralStats
```

---

### 45. `wallet` module

**schema.ts**
```
purchasePackSchema: { pack_id, payment_id, payment_gateway }
giftSchema: { recipient_id, amount, note? }
```

**service.ts**
```
getBalance(userId) → Wallet.findOne { user_id }
getTransactions(userId, page, limit) → Transaction.find paginated + sort createdAt desc
getPacks() → check Redis 'packs:active' → PointPack.find { is_active: true }
purchasePack(userId, input):
  fetch pack, validate active
  start mongo session
  Wallet.findOneAndUpdate $inc balance (atomic credit)
  Transaction.create (recharge, reference: pack)
  PackPurchase.create
  commit
  enqueue wallet-events 'tier_upgrade'

giftPoints(senderId, input):
  if senderId === recipient_id → throw ERR_SELF_GIFT
  check both wallets exist + sender not frozen
  start mongo session
  Wallet.findOneAndUpdate sender $gte amount (atomic debit)
  Wallet.findOneAndUpdate recipient $inc (atomic credit)
  Transaction.create x2 (gift_sent + gift_received)
  commit
```

**routes.ts**
```
GET  /balance         jwtMiddleware + requireClient → getBalance
GET  /transactions    jwtMiddleware → getTransactions
GET  /packs           public → getPacks
POST /packs/purchase  jwtMiddleware + requireClient → purchasePack
POST /gift            jwtMiddleware + requireClient → giftPoints
```

---

### 46. `booking` module

**schema.ts**
```
createBookingSchema: { system_id, slot_iso, duration_minutes, coupon_code? }
rateSessionSchema: { rating (1-5), comment? }
```

**service.ts**
```
getAvailability(venueId, date):
  check Redis avail:{venueId}:{date} (TTL 10s)
  if miss → fetch sessions for that day, compute booked slots
  cache + return

createBooking(userId, input):
  1. acquireLock(slotLock(system_id, slot_iso), 10s) → if false throw ERR_SYSTEM_UNAVAILABLE
  2. start mongo session
  3. fetch system + pricing_rules for user tier
  4. check flash_sale (Redis flash:active)
  5. apply coupon if provided (atomic $lt increment used_count)
  6. compute final_cost
  7. Wallet.findOneAndUpdate $gte final_cost (atomic debit)  → null = ERR_WALLET_INSUFFICIENT
  8. Session.create
  9. Transaction.create
  10. invalidate Redis avail: key
  11. commit
  finally: releaseLock()
  enqueue session-events 'update_leaderboard'

getMyBookings(userId, status?, page, limit)
cancelBooking(userId, sessionId):
  fetch session, verify ownership + status upcoming
  start mongo session
  refund_points = partial refund based on notice time
  Wallet.findOneAndUpdate $inc refund_points
  Transaction.create (refund)
  Session.findByIdAndUpdate status cancelled
  commit
  enqueue session-events 'notify_waitlist'

rateSession(userId, sessionId, input)
```

**routes.ts**
```
GET  /availability     public → getAvailability
POST /                 jwtMiddleware + requireClient → createBooking
GET  /                 jwtMiddleware → getMyBookings
GET  /:id             jwtMiddleware → getSessionById
DELETE /:id           jwtMiddleware + requireClient → cancelBooking
POST /:id/rate        jwtMiddleware + requireClient → rateSession
```

---

### 47. `food` module

**schema.ts**
```
createFoodOrderSchema: {
  venue_id,
  items: [{ food_item_id, quantity }]
}
```

**service.ts**
```
getMenu(venueId):
  check Redis menu:{venueId} (TTL 300s)
  if miss → FoodItem.find { venue_id, is_available: true }
  cache + return

createFoodOrder(userId, input):
  check Redis session:{userId} → if no active session throw ERR_NO_ACTIVE_SESSION
  fetch all food items, validate is_available
  build items[] with PRICE SNAPSHOT at current time
  compute total_points
  start mongo session
  Wallet.findOneAndUpdate $gte total_points (atomic debit)
  FoodOrder.create with embedded items snapshot
  Transaction.create
  commit
  enqueue notification 'food_push'

getMyOrders(userId, page, limit)
cancelOrder(userId, orderId):
  verify ownership + status pending
  refund total_points to wallet
  update status cancelled
```

**routes.ts**
```
GET  /menu          jwtMiddleware → getMenu (query: venue_id)
POST /orders        jwtMiddleware + requireClient → createFoodOrder
GET  /orders        jwtMiddleware → getMyOrders
GET  /orders/:id    jwtMiddleware → getOrderById
POST /orders/:id/cancel  jwtMiddleware + requireClient → cancelOrder
```

---

### 48. `coupon` module

**schema.ts**
```
validateCouponSchema: { code, booking_value?, applicable_to }
```

**service.ts**
```
validateCoupon(userId, input):
  acquireLock(couponLock(couponId), 5s)
  fetch coupon by code → not found / inactive / expired → throw
  check used_count < max_uses → ERR_COUPON_EXHAUSTED
  check per_user_limit → CouponUsage.countDocuments by user
  compute discount_amount
  releaseLock()
  return { discount_amount, coupon_id }
  (note: actual $lt increment happens inside booking/food service)
```

**routes.ts**
```
POST /validate    jwtMiddleware + requireClient → validateCoupon
```

---

### 49. `leaderboard` module

**service.ts**
```
getLeaderboard(period, venueId?, page, limit):
  key = leaderboard(period, periodKey)
  top N from Redis ZRANGE key 0 N-1 REV WITHSCORES
  enrich with user names from DB
  return ranked list

getMyRank(userId, period):
  key = leaderboard(period, periodKey)
  rank = Redis ZREVRANK key userId  → O(log N)
  score = Redis ZSCORE key userId
  return { rank, score }
```

**routes.ts**
```
GET /           public → getLeaderboard (query: period, venue_id)
GET /my-rank    jwtMiddleware + requireClient → getMyRank (query: period)
```

---

### 50. `tournament` module

**schema.ts**
```
registerSchema: { tournament_id }
```

**service.ts**
```
listTournaments(status?, venueId?) → Tournament.find
getTournament(id) → Tournament.findById
registerParticipant(userId, tournamentId):
  fetch tournament → validate status registration_open
  check max_participants → count existing → ERR_TOURNAMENT_FULL
  check not already registered → ERR_ALREADY_REGISTERED
  start mongo session
  Wallet.findOneAndUpdate $gte entry_fee_points (atomic debit)
  TournamentParticipant.create
  Transaction.create
  commit
  enqueue notification

withdrawParticipant(userId, tournamentId):
  verify registration exists + tournament still registration_open
  refund entry fee
  update participant status refunded
```

**routes.ts**
```
GET  /                   public → listTournaments
GET  /:id               public → getTournament
POST /:id/register      jwtMiddleware + requireClient → registerParticipant
DELETE /:id/register    jwtMiddleware + requireClient → withdrawParticipant
```

---

### 51. `venue` module

**schema.ts**
```
qrCheckinSchema: { qr_token }
```

**service.ts**
```
checkinByQr(token, venueId):
  tokenHash = SHA256(token)
  check Redis qr:{tokenHash} → if miss → check DB User.qr_token
  validate not expired
  VenueCheckin.create { method: 'qr' }
  enqueue notification

getAnnouncements(venueId):
  Announcement.find { $or: [venue_id, null], is_active, starts_at lte now }
  sorted by starts_at desc
```

**routes.ts**
```
POST /checkin/qr        jwtMiddleware → checkinByQr
GET  /announcements     jwtMiddleware → getAnnouncements (query: venue_id)
```

---

### 52. `seller` module

**schema.ts**
```
lookupClientSchema: { rfid_uid? | email? | phone? }
creditPointsSchema: { user_id, points, reason }
```

**service.ts**
```
lookupClient(input):
  find User by rfid (UserCard) or email or phone
  return { user, wallet_balance, active_session, tier }

creditPoints(sellerId, venueId, input):
  fetch seller's venue_id from JWT, verify match
  Wallet.findOneAndUpdate $inc points (atomic credit)
  Transaction.create (credit, initiated_by: sellerId)
  SellerCreditLog.create
  enqueue wallet-events 'tier_upgrade'

getVenueBoard(venueId):
  active sessions count, systems status, today's revenue

getQueue(venueId, systemId?):
  WaitlistEntry.find { venue_id } sorted by createdAt
```

**routes.ts**
```
POST /clients/lookup        jwtMiddleware + requireSeller → lookupClient
POST /clients/credit        jwtMiddleware + requireSeller → creditPoints
GET  /venue/board           jwtMiddleware + requireSeller → getVenueBoard
GET  /venue/queue           jwtMiddleware + requireSeller → getQueue
```

---

### 53. `device` module

**schema.ts**
```
checkinSchema: { rfid_uid, venue_id }
checkoutSchema: { rfid_uid, venue_id }
```

**service.ts**
```
rfidCheckin(deviceId, input):
  find UserCard by rfid_uid → ERR_RFID_NOT_FOUND
  if card.status blocked/lost → ERR_CARD_BLOCKED
  find active session for user
  start mongo session (4-doc transaction):
    VenueCheckin.create { method: 'rfid', device_id }
    Session.findByIdAndUpdate status → 'active', actual_start = now
    update Redis session:{userId} active
    AuditLog.create
  commit

rfidCheckout(deviceId, input):
  find active VenueCheckin for user
  start mongo session:
    VenueCheckin update checked_out_at
    Session.findByIdAndUpdate status → 'completed', actual_end = now
    del Redis session:{userId}
    AuditLog.create
  commit
  enqueue session-events 'update_streak' + 'update_leaderboard'

getDeviceStatus(deviceId) → last checkin, active sessions at venue
```

**routes.ts**
```
POST /checkin    apiKeyMiddleware → rfidCheckin
POST /checkout   apiKeyMiddleware → rfidCheckout
GET  /status     apiKeyMiddleware → getDeviceStatus
```

---

### 54. `admin` module

**schema.ts**
```
blacklistSchema: { user_id, reason }
freezeWalletSchema: { user_id, reason }
adjustWalletSchema: { user_id, amount, note }
```

**service.ts**
```
getUsers(filters, page, limit) → paginated User.find
getUser(userId) → full profile
blacklistUser(adminId, input) → User update + AuditLog + enqueue notification
unblacklistUser(adminId, userId) → same
freezeWallet(adminId, input) → Wallet update + AuditLog
unfreezeWallet(adminId, userId)
adjustWallet(adminId, input):
  Wallet.findOneAndUpdate $inc amount
  Transaction.create (admin_adjustment)
  AuditLog.create
getAuditLogs(filters, page, limit)
getAnalytics(venueId?, period):
  check Redis analytics:{venue}:{period} (TTL 300s)
  aggregate sessions, revenue, top users
  cache + return
issueCard(adminId, userId, rfidUid):
  check rfid_uid not already taken → ERR_RFID_TAKEN
  UserCard.create
  CardIssuanceLog.create
blockCard(adminId, cardId, reason)
```

**routes.ts**
```
all routes: jwtMiddleware + requireAdmin

GET    /users
GET    /users/:id
POST   /users/:id/blacklist
DELETE /users/:id/blacklist
POST   /wallets/:userId/freeze
DELETE /wallets/:userId/freeze
POST   /wallets/:userId/adjust
GET    /audit-logs
GET    /analytics
POST   /cards/issue
POST   /cards/:id/block
```

---

### 55. `notification` module

**service.ts**
```
getNotifications(userId, page, limit) → Notification.find paginated
markRead(userId, notificationId) → update is_read + read_at
markAllRead(userId) → Notification.updateMany { user_id, is_read: false }
```

**routes.ts**
```
GET  /              jwtMiddleware → getNotifications
PUT  /:id/read      jwtMiddleware → markRead
PUT  /read-all      jwtMiddleware → markAllRead
```

---

## PHASE 10 — Config Files

### 56. `docker-compose.dev.yml` (root)
```
version: '3.8'
services:
  redis:
    image: redis:7-alpine
    ports: 6379:6379
    volumes: redis_data
    healthcheck: redis-cli ping
```

### 57. `migrate-mongo-config.js` (root)
```
module.exports = {
  mongodb: { url: process.env.MONGODB_URI, databaseName: 'spawnpoint' },
  migrationsDir: 'packages/db/src/migrations',
  changelogCollectionName: 'changelog',
  migrationFileExtension: '.js'
}
```

### 58. `tsconfig.check.json` (root)
```
extends ./tsconfig.json
include: all src/**/*.ts across apps + packages
paths: @spawnpoint/db, @spawnpoint/redis, @spawnpoint/types
```

---

## Completion Checklist

- [ ] Phase 1 — Shared utilities (3 files)
- [ ] Phase 2 — Express bootstrap (2 files)
- [ ] Phase 3 — Middleware updates (3 files)
- [ ] Phase 4 — Models (28 files)
- [ ] Phase 5 — DB index update
- [ ] Phase 6 — Seeds (1 file)
- [ ] Phase 7 — Workers bootstrap (2 files)
- [ ] Phase 8 — Missing workers (3 files)
- [ ] Phase 9 — Module implementations (12 modules)
- [ ] Phase 10 — Config files (3 files)