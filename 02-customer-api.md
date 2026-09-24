# Customer API

Base path: `/api/v1/customers` 

A Customer must **register and log in** (phone + password) before booking  

Login behaves normally: register/login happens once, the app stores the
returned token, and the Customer stays logged in on that device
indefinitely, the login screen only reappears if they explicitly log out
or open the app on a new device. 

**Persistence note:** the nightly TTL job (documented in
[01-barber-api.md](./01-barber-api.md#4-daily-slot-management)) wipes
`Bookings` and `Slots` every midnight  but **not** `Customer` accounts. A
Customer's `name`, `phone`, `location`, and `password_hash` persist
indefinitely; only the record of *what they booked* is erased.

---

## 0. Account & Login

### Register

```
POST /api/v1/customers/register
```

**Auth:** none

**Request body**
```json
{
  "name": "Anita Sharma",
  "phone": "+919812345678",
  "password": "••••••••",
  "location": { "lat": 12.9750, "lng": 77.6010 }
}
```

**Response `201 Created`**
```json
{
  "customer_id": 88,
  "name": "Anita Sharma",
  "phone": "+919812345678",
  "token": "eyJhbGciOi..."
}
```

`phone` is `UNIQUE` on the `Customer` table, one account per phone number.
The app stores `token` and sends it as `Authorization: Bearer <token>` on
every request from here on; the Customer is not asked to log in again on
this device unless they log out.

**Errors**
| Status | Code | When |
|---|---|---|
| 400 | `VALIDATION_ERROR` | Missing `name`, `phone`, or `password` |
| 409 | `ALREADY_REGISTERED` | This phone number is already registered |

---

### Login

```
POST /api/v1/customers/login
```

**Auth:** none

**Request body**
```json
{ "phone": "+919812345678", "password": "••••••••" }
```

**Response `200 OK`**
```json
{ "customer_id": 88, "token": "eyJhbGciOi..." }
```

**Errors:** `401 INVALID_CREDENTIALS` — wrong phone/password combination.

---

## 1. Finding a Barber (entry points — documented in the Barber API)

A Customer reaches a specific Barber's page one of two ways, both already
covered in [01-barbar-api.md](./01-barbar-api.md):

- **QR scan** → the QR encodes a URL containing `barber_id` directly →
  frontend calls `GET /barbers/{barber_id}` (public subset).
- **Nearby search, no QR** → `GET /barbers/nearby?lat=..&lng=..&radius_km=6`
  → Customer taps a result → same `GET /barbers/{barber_id}` call.

---

## 2. Viewing today's slots

```
GET /api/v1/barbers/{barber_id}/slots
```

Same shared, public endpoint documented in
[01-barbar-api.md](./01-barbar-api.md#4-daily-slot-management), the
Customer's app polls this to render the slot grid (Green = `available`,
Gray = `na`, Red = `booked`, Blue = `approved`). 

---

## 3. Creating a booking

```
POST /api/v1/barbers/{barber_id}/slots/{slot_id}/bookings
```

**Auth:** required `Authorization: Bearer <token>`. If missing or
invalid, the API returns `401 Unauthorized`; the app's job is to catch this
and show the login/register screen (§0) before letting the Customer retry.
There is no request-body fallback for identifying the Customer `name`,
`phone`, and `location` are never sent here, since they're already on the
logged-in Customer's account. `customer_id` is read directly from the
token.

**Headers**
```
Idempotency-Key: <client-generated UUID>
```
Required to safely handle network retries without creating a duplicate
booking on the same slot.

**Request body:** none required.

**Response `201 Created`**
```json
{
  "booking_id": 901,
  "customer_id": 88,
  "slot_id": 502,
  "barber_id": 12,
  "booking_status": "waiting",
  "created_at": "2026-09-23T10:05:00Z"
}
```

Behind the scenes, this call atomically checks the slot's current
`status = 'available'` and flips it to `booked` in one transaction, this
is the real safeguard against two Customers booking the same slot
simultaneously.

**Customer data persistence:** the `Customer` row is untouched by the
nightly TTL job, only `Bookings`/`Slots` rows are wiped at midnight; the
account itself (and its `customer_id`) stays valid indefinitely.

**Errors**
| Status | Code | When |
|---|---|---|
| 401 | `UNAUTHORIZED` | Missing/invalid/expired token — show login screen |
| 404 | `NOT_FOUND` | `barber_id`/`slot_id` doesn't exist, or already TTL-purged |
| 409 | `SLOT_ALREADY_BOOKED` | Someone else booked this slot first (lost the race) |
| 409 | `BARBER_INACTIVE` | Barber's subscription is `expired` booking page shouldn't allow this, but enforced server-side too |

---

## 4. Checking booking status & getting notified

Its "in browser" notification, "in
browser" notification means the Customer's app **polls this endpoint** for
a status change and displays a banner/alert client-side the moment it sees
one.

```
GET /api/v1/bookings/{booking_id}
```

**Auth:** required `Authorization: Bearer <token>`. The server checks the
booking's `customer_id` matches the token before returning it.

**Response `200 OK`**
```json
{
  "booking_id": 901,
  "booking_status": "approved",
  "slot": { "start_time": "10:30", "end_time": "11:00" },
  "barber": { "name": "Ravi Kumar" }
}
```

the Customer only
ever sees a static status change (e.g. "Ravi Kumar has approved your
booking"), not a moving countdown.

**Errors**
| Status | Code | When |
|---|---|---|
| 401 | `UNAUTHORIZED` | Missing/invalid/expired token |
| 403 | `FORBIDDEN` | Token belongs to a different Customer than this booking |
| 404 | `NOT_FOUND` | Booking already TTL-purged (i.e. it's now a new day), or doesn't exist |

---

## 5. Cancelling a booking

```
PATCH /api/v1/bookings/{booking_id}/cancel
```

**Auth:** required — same ownership (token's `customer_id`
must match the booking).

**Request body:** none.

**Response `200 OK`** — `booking_status` → `cancelled`; linked slot reopens
to `available` immediately (so it can show up Green again on the next
`GET /slots` poll, and be booked by someone else the same day).

Allowed from **either** `waiting` or `approved` state, a Customer can back out even after the Barber has already
accepted.

**Errors**
| Status | Code | When |
|---|---|---|
| 401 | `UNAUTHORIZED` | Missing/invalid/expired token |
| 403 | `FORBIDDEN` | Token belongs to a different Customer than this booking |
| 404 | `NOT_FOUND` | Booking doesn't exist or already TTL-purged |
| 422 | `INVALID_STATE_TRANSITION` | Booking is already `rejected`, `cancelled`, or `completed` — nothing to cancel |

---

## Summary of the full Customer journey

0. `POST /customers/register` (first time only) or `POST /customers/login` — get a token; the app remembers it, so this step is skipped on every future visit unless the Customer logs out.
1. `GET /barbers/{id}` (via QR or `GET /barbers/nearby`) — land on a Barber's page. No login needed just to browse.
2. `GET /barbers/{id}/slots` (polled) — view live slot grid. No login needed.
3. Tap a slot → `POST /barbers/{id}/slots/{slot_id}/bookings` **with the stored token**. If no valid token exists yet, the app routes to step 0 first, then completes the booking automatically afterward.
4. `GET /bookings/{booking_id}` (polled, authenticated) — wait for `waiting` → `approved`/`rejected`; in-browser banner on change.
5. `PATCH /bookings/{booking_id}/cancel` (authenticated) — at `waiting` or `approved` stage.

Login happens once per device and is never asked for repeatedly. It does
**not** grant booking history, `Bookings` is still wiped unconditionally
every night for every Customer.
