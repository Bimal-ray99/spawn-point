# Spawnpoint API Documentation
Base URL: `/api/v1`

Auth: `Authorization: Bearer <access_token>` unless marked **Public**

---

## Auth `/auth`

| Method | Endpoint | Auth | Description |
|--------|----------|------|-------------|
| POST | `/auth/signup` | Public | Register new client |
| POST | `/auth/login` | Public | Login, get tokens |
| POST | `/auth/refresh` | Public | Rotate refresh token |
| POST | `/auth/logout` | Client | Revoke refresh token |
| GET | `/auth/sessions` | Client | List active sessions |
| POST | `/auth/change-password` | Client | Change password |

### POST `/auth/signup`
**Body**
```json
{
  "name": "Rahul Sharma",
  "email": "rahul@example.com",
  "phone": "+919876543210",
  "password": "strongpass123",
  "referral_code": "ABC123DEF456"
}
```
**Response 201**
```json
{
  "success": true,
  "data": {
    "user": { "id": "", "name": "", "email": "", "role": "client" },
    "tokens": { "access_token": "", "refresh_token": "" }
  }
}
```

### POST `/auth/login`
**Body**
```json
{
  "email": "rahul@example.com",
  "password": "strongpass123",
  "device_info": "iPhone 15 / iOS 17"
}
```
**Response 200**
```json
{
  "success": true,
  "data": {
    "tokens": { "access_token": "", "refresh_token": "" },
    "user": { "id": "", "role": "client", "is_vip": false }
  }
}
```

### POST `/auth/refresh`
**Body**
```json
{ "refresh_token": "" }
```

### POST `/auth/logout`
**Body**
```json
{ "refresh_token": "", "logout_all": false }
```

---

## Users `/users`
> All routes require Client auth unless noted

| Method | Endpoint | Auth | Description |
|--------|----------|------|-------------|
| GET | `/users/profile` | Client | Get own profile |
| PUT | `/users/profile` | Client | Update profile |
| GET | `/users/qr-token` | Client | Get current QR token |
| POST | `/users/qr-token/refresh` | Client | Rotate QR token |
| GET | `/users/notifications` | Client | List notifications |
| PUT | `/users/notification-prefs` | Client | Update notification preferences |
| GET | `/users/referral-stats` | Client | Referral earnings summary |

### GET `/users/profile`
**Response 200**
```json
{
  "success": true,
  "data": {
    "id": "",
    "name": "",
    "email": "",
    "phone": "",
    "role": "client",
    "tier": { "name": "Silver", "level": 2, "discount_percentage": 10 },
    "lifetime_points": 2400,
    "referral_code": "ABC123DEF456",
    "is_vip": false,
    "avatar_url": "",
    "preferred_system": "pc",
    "notification_preferences": { "booking_push": true, "wallet_push": true }
  }
}
```

### PUT `/users/profile`
**Body**
```json
{
  "name": "Rahul S",
  "avatar_url": "https://cdn.example.com/avatar.jpg",
  "preferred_system": "ps"
}
```

### PUT `/users/notification-prefs`
**Body** — all fields optional
```json
{
  "booking_push": true,
  "booking_email": false,
  "food_push": true,
  "wallet_push": true,
  "wallet_email": false,
  "streak_push": true,
  "tier_push": true,
  "tournament_push": true,
  "waitlist_push": true,
  "referral_push": true,
  "announcement_push": false
}
```

### GET `/users/notifications`
**Query** `?page=1&limit=20`

---

## Wallet `/wallet`

| Method | Endpoint | Auth | Description |
|--------|----------|------|-------------|
| GET | `/wallet/balance` | Client | Get wallet balance |
| GET | `/wallet/transactions` | Client | Transaction history |
| GET | `/wallet/packs` | Public | List point packs |
| POST | `/wallet/packs/purchase` | Client | Purchase a point pack |
| POST | `/wallet/gift` | Client | Gift points to another user |

### GET `/wallet/balance`
**Response 200**
```json
{
  "success": true,
  "data": {
    "balance": 1500,
    "locked_balance": 0,
    "lifetime_credited": 5000,
    "is_frozen": false
  }
}
```

### GET `/wallet/transactions`
**Query** `?page=1&limit=20&type=debit`

**Response 200**
```json
{
  "success": true,
  "data": {
    "items": [
      {
        "id": "",
        "type": "debit",
        "amount": 200,
        "balance_before": 1700,
        "balance_after": 1500,
        "reference_type": "session",
        "reference_id": "",
        "note": "Booking PC-01",
        "createdAt": ""
      }
    ],
    "meta": { "total": 45, "page": 1, "limit": 20, "pages": 3 }
  }
}
```

### POST `/wallet/packs/purchase`
**Body**
```json
{
  "pack_id": "",
  "payment_id": "pay_xyz123",
  "payment_gateway": "razorpay"
}
```

### POST `/wallet/gift`
**Body**
```json
{
  "recipient_id": "",
  "amount": 100,
  "note": "Happy Birthday!"
}
```

---

## Bookings `/bookings`

| Method | Endpoint | Auth | Description |
|--------|----------|------|-------------|
| GET | `/bookings/availability` | Public | Get available slots |
| POST | `/bookings` | Client | Create booking |
| GET | `/bookings` | Client | My bookings |
| GET | `/bookings/:id` | Client | Booking detail |
| DELETE | `/bookings/:id` | Client | Cancel booking |
| POST | `/bookings/:id/rate` | Client | Rate completed session |

### GET `/bookings/availability`
**Query** `?venue_id=&date=2024-04-01`

**Response 200**
```json
{
  "success": true,
  "data": {
    "venue_id": "",
    "date": "2024-04-01",
    "systems": [
      {
        "system_id": "",
        "name": "PC-01",
        "type": "pc",
        "booked_slots": ["2024-04-01T10:00:00Z", "2024-04-01T11:00:00Z"],
        "is_maintenance": false
      }
    ]
  }
}
```

### POST `/bookings`
**Body**
```json
{
  "system_id": "",
  "slot_iso": "2024-04-01T14:00:00Z",
  "duration_minutes": 60,
  "coupon_code": "SAVE20"
}
```
**Response 201**
```json
{
  "success": true,
  "data": {
    "session_id": "",
    "system_name": "PC-01",
    "scheduled_start": "2024-04-01T14:00:00Z",
    "scheduled_end": "2024-04-01T15:00:00Z",
    "base_cost": 200,
    "tier_discount_applied": 20,
    "coupon_discount_applied": 36,
    "final_cost": 144,
    "wallet_balance_remaining": 1356
  }
}
```

### POST `/bookings/:id/rate`
**Body**
```json
{ "rating": 5, "comment": "Amazing setup!" }
```

---

## Food `/food`

| Method | Endpoint | Auth | Description |
|--------|----------|------|-------------|
| GET | `/food/menu` | Client | Get venue menu |
| POST | `/food/orders` | Client | Place food order |
| GET | `/food/orders` | Client | My food orders |
| GET | `/food/orders/:id` | Client | Order detail |
| POST | `/food/orders/:id/cancel` | Client | Cancel pending order |

### GET `/food/menu`
**Query** `?venue_id=`

**Response 200**
```json
{
  "success": true,
  "data": {
    "categories": ["snacks", "beverages", "meals"],
    "items": [
      {
        "id": "",
        "name": "Veg Burger",
        "category": "meals",
        "price_points": 80,
        "prep_time_minutes": 10,
        "is_available": true
      }
    ]
  }
}
```

### POST `/food/orders`
> Requires active gaming session

**Body**
```json
{
  "venue_id": "",
  "items": [
    { "food_item_id": "", "quantity": 2 },
    { "food_item_id": "", "quantity": 1 }
  ]
}
```
**Response 201**
```json
{
  "success": true,
  "data": {
    "order_id": "",
    "items": [
      { "item_name_snapshot": "Veg Burger", "quantity": 2, "unit_price_points": 80, "subtotal_points": 160 }
    ],
    "total_points": 240,
    "status": "pending",
    "estimated_prep_minutes": 15
  }
}
```

---

## Coupons `/coupons`

| Method | Endpoint | Auth | Description |
|--------|----------|------|-------------|
| POST | `/coupons/validate` | Client | Validate coupon + get discount |

### POST `/coupons/validate`
**Body**
```json
{
  "code": "SAVE20",
  "booking_value": 200,
  "applicable_to": "booking"
}
```
**Response 200**
```json
{
  "success": true,
  "data": {
    "coupon_id": "",
    "discount_type": "percentage",
    "discount_value": 20,
    "discount_amount": 40,
    "max_discount_points": 100
  }
}
```

---

## Leaderboard `/leaderboard`

| Method | Endpoint | Auth | Description |
|--------|----------|------|-------------|
| GET | `/leaderboard` | Public | Top players |
| GET | `/leaderboard/my-rank` | Client | My rank + score |

### GET `/leaderboard`
**Query** `?period=weekly&venue_id=&page=1&limit=10`

**Response 200**
```json
{
  "success": true,
  "data": {
    "period": "weekly",
    "period_key": "2024-W14",
    "entries": [
      { "rank": 1, "user_id": "", "name": "Rahul S", "points": 1200, "avatar_url": "" }
    ]
  }
}
```

### GET `/leaderboard/my-rank`
**Query** `?period=weekly`

**Response 200**
```json
{
  "success": true,
  "data": { "rank": 12, "points": 540, "period": "weekly" }
}
```

---

## Tournaments `/tournaments`

| Method | Endpoint | Auth | Description |
|--------|----------|------|-------------|
| GET | `/tournaments` | Public | List tournaments |
| GET | `/tournaments/:id` | Public | Tournament detail + brackets |
| POST | `/tournaments/:id/register` | Client | Register for tournament |
| DELETE | `/tournaments/:id/register` | Client | Withdraw registration |

### GET `/tournaments`
**Query** `?status=registration_open&venue_id=`

### GET `/tournaments/:id`
**Response 200**
```json
{
  "success": true,
  "data": {
    "id": "",
    "name": "Friday Fight Night",
    "game_type": "ps",
    "status": "registration_open",
    "entry_fee_points": 100,
    "prize_pool_points": 2000,
    "prize_distribution": [
      { "position": 1, "points": 1200 },
      { "position": 2, "points": 500 },
      { "position": 3, "points": 300 }
    ],
    "max_participants": 16,
    "registered_count": 9,
    "registration_start": "",
    "registration_end": "",
    "brackets": []
  }
}
```

### POST `/tournaments/:id/register`
**Response 201**
```json
{
  "success": true,
  "data": { "participant_id": "", "entry_fee_paid": 100, "status": "registered" }
}
```

---

## Venues `/venues`

| Method | Endpoint | Auth | Description |
|--------|----------|------|-------------|
| POST | `/venues/checkin/qr` | Client | QR code check-in |
| GET | `/venues/announcements` | Client | Active announcements |

### POST `/venues/checkin/qr`
**Body**
```json
{ "qr_token": "", "venue_id": "" }
```

### GET `/venues/announcements`
**Query** `?venue_id=`

---

## Seller `/seller`
> All routes require Seller auth

| Method | Endpoint | Auth | Description |
|--------|----------|------|-------------|
| POST | `/seller/clients/lookup` | Seller | Find client by RFID/email/phone |
| POST | `/seller/clients/credit` | Seller | Manually credit points |
| GET | `/seller/venue/board` | Seller | Venue dashboard |
| GET | `/seller/venue/queue` | Seller | Waitlist queue |

### POST `/seller/clients/lookup`
**Body** — one of:
```json
{ "rfid_uid": "04:AB:CD:12:34:56" }
{ "email": "rahul@example.com" }
{ "phone": "+919876543210" }
```
**Response 200**
```json
{
  "success": true,
  "data": {
    "user": { "id": "", "name": "", "tier": "Silver" },
    "wallet_balance": 1500,
    "active_session": { "system_name": "PC-01", "started_at": "" },
    "card_status": "active"
  }
}
```

### POST `/seller/clients/credit`
**Body**
```json
{ "user_id": "", "points": 50, "reason": "Loyalty bonus" }
```

### GET `/seller/venue/board`
**Response 200**
```json
{
  "success": true,
  "data": {
    "active_sessions": 8,
    "total_systems": 12,
    "systems_in_maintenance": 1,
    "today_revenue_points": 4200,
    "waitlist_count": 3
  }
}
```

---

## Device `/device`
> Requires `x-api-key` + `x-device-id` headers (ESP32 HMAC auth)

| Method | Endpoint | Auth | Description |
|--------|----------|------|-------------|
| POST | `/device/checkin` | API Key | RFID tap-in |
| POST | `/device/checkout` | API Key | RFID tap-out |
| GET | `/device/status` | API Key | Device health + active sessions |

### POST `/device/checkin`
**Headers**
```
x-api-key: <raw_key>
x-device-id: device_001
```
**Body**
```json
{ "rfid_uid": "04:AB:CD:12:34:56", "venue_id": "" }
```
**Response 200**
```json
{
  "success": true,
  "data": {
    "user_name": "Rahul S",
    "session_id": "",
    "system_name": "PC-01",
    "message": "Welcome! Session started."
  }
}
```

### POST `/device/checkout`
**Body**
```json
{ "rfid_uid": "04:AB:CD:12:34:56", "venue_id": "" }
```

---

## Admin `/admin`
> All routes require Admin auth

| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | `/admin/users` | List all users (paginated + filters) |
| GET | `/admin/users/:id` | Full user profile |
| POST | `/admin/users/:id/blacklist` | Blacklist user |
| DELETE | `/admin/users/:id/blacklist` | Remove blacklist |
| POST | `/admin/wallets/:userId/freeze` | Freeze wallet |
| DELETE | `/admin/wallets/:userId/freeze` | Unfreeze wallet |
| POST | `/admin/wallets/:userId/adjust` | Manual point adjustment |
| GET | `/admin/audit-logs` | Audit log (paginated + filters) |
| GET | `/admin/analytics` | Revenue + usage analytics |
| POST | `/admin/cards/issue` | Issue RFID card to user |
| POST | `/admin/cards/:id/block` | Block a card |

### GET `/admin/users`
**Query** `?role=client&is_blacklisted=false&page=1&limit=20&search=rahul`

### POST `/admin/users/:id/blacklist`
**Body**
```json
{ "reason": "Fraudulent activity" }
```

### POST `/admin/wallets/:userId/adjust`
**Body**
```json
{ "amount": -100, "note": "Correction for duplicate charge" }
```

### GET `/admin/analytics`
**Query** `?venue_id=&period=weekly`

**Response 200**
```json
{
  "success": true,
  "data": {
    "total_sessions": 320,
    "total_revenue_points": 64000,
    "new_users": 45,
    "active_users": 210,
    "top_system_type": "pc",
    "avg_session_duration_minutes": 72
  }
}
```

### POST `/admin/cards/issue`
**Body**
```json
{ "user_id": "", "rfid_uid": "04:AB:CD:12:34:56" }
```

---

## Notifications `/notifications`

| Method | Endpoint | Auth | Description |
|--------|----------|------|-------------|
| GET | `/notifications` | Client | List notifications |
| PUT | `/notifications/:id/read` | Client | Mark one as read |
| PUT | `/notifications/read-all` | Client | Mark all as read |

### GET `/notifications`
**Query** `?page=1&limit=20&is_read=false`

---

## Standard Error Responses

```json
{
  "success": false,
  "error": {
    "code": "ERR_WALLET_INSUFFICIENT",
    "message": "Insufficient wallet balance"
  },
  "meta": {
    "request_id": "550e8400-e29b-41d4-a716-446655440000",
    "timestamp": "2024-04-01T12:00:00.000Z"
  }
}
```

| HTTP | Code | Meaning |
|------|------|---------|
| 400 | ERR_VALIDATION | Invalid request body |
| 401 | ERR_UNAUTHORIZED | Missing/invalid token |
| 401 | ERR_TOKEN_EXPIRED | Access token expired |
| 403 | ERR_FORBIDDEN | Insufficient role |
| 403 | ERR_ACCOUNT_BLACKLISTED | Account blacklisted |
| 404 | ERR_NOT_FOUND | Resource not found |
| 409 | ERR_SYSTEM_UNAVAILABLE | Slot already taken |
| 409 | ERR_ALREADY_REGISTERED | Already in tournament |
| 422 | ERR_WALLET_INSUFFICIENT | Not enough points |
| 422 | ERR_WALLET_FROZEN | Wallet is frozen |
| 429 | ERR_TOO_MANY_ATTEMPTS | Rate limited |