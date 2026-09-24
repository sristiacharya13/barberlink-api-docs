# Barber API

Base path: `/api/v1/barbers`

This document covers every endpoint the Barber side of the app needs:
account & profile, shop location & discovery, subscription/payments, daily
slot management, and booking actions (view/approve/reject/complete).

---

## 1. Account & Profile

### Register a Barber

Triggered when a Barber scans the **universal QR** and submits the
registration form. Creates the Barber record and starts the free-trial
subscription clock.

```
POST /api/v1/barbers
```

**Auth:** none

**Request body**
```json
{ "name": "Ravi Kumar", "contact_number": "+919876543210" }
```

**Response `201 Created`**
```json
{
  "barber_id": 12,
  "name": "Ravi Kumar",
  "contact_number": "+919876543210",
  "qr_code": "https://api.barberapp.com/qr/b12x9f.png",
  "account_created_at": "2026-09-23T10:15:00Z",
  "location": null,
  "subscription_status": "trial",
  "subscription_expiry_date": "2027-09-23",
  "token": "eyJhbGciOi..."
}
```

`qr_code` is generated server-side after the record is created, since it
encodes the new `barber_id`. `location` starts `null` set separately
below. `token` is the JWT the Barber's app stores and sends as
`Authorization: Bearer <token>` on every subsequent authenticated call.

**Errors**
| Status | Code | When |
|---|---|---|
| 400 | `VALIDATION_ERROR` | Missing `name` or `contact_number` |
| 409 | `DUPLICATE_CONTACT_NUMBER` | A Barber with this contact number already exists |

**Open item:** this document assumes a Barber stays logged in on one device
after registration (token stored locally). A separate login/re-auth
endpoint (e.g. OTP-to-phone) would be needed to support signing in on a new
device. (Implemeting it later)

---

### Get profile / dashboard

```
GET /api/v1/barbers/{barber_id}
```

**Auth:** Barber (own profile) or none

**Response `200 OK` — Barber's own view**
```json
{
  "barber_id": 12,
  "name": "Ravi Kumar",
  "contact_number": "+919876543210",
  "qr_code": "https://api.barberapp.com/qr/b12x9f.png",
  "location": { "lat": 12.9716, "lng": 77.5946 },
  "subscription_status": "active",
  "subscription_expiry_date": "2027-09-23"
}
```

**Response `200 OK` — public view (Customer via QR/nearby search)**
```json
{ "barber_id": 12, "name": "Ravi Kumar", "location": { "lat": 12.9716, "lng": 77.5946 } }
```

The server picks the response shape based on whether a valid Barber token
is present. `contact_number`, `qr_code`, and subscription fields are never
exposed publicly.

**Errors:** `404 NOT_FOUND` — no Barber with this id.

---

### Update profile (name / contact number)

```
PATCH /api/v1/barbers/{barber_id}
```

**Auth:** Barber (must match `{barber_id}`)

**Request body** (any subset)
```json
{ "contact_number": "+919876500000" }
```

**Response `200 OK`** — updated Barber object. `location` is intentionally
**not** accepted here — see the dedicated location endpoint next, so a
routine name/number edit can never accidentally overwrite shop coordinates.

**Errors**
| Status | Code | When |
|---|---|---|
| 400 | `VALIDATION_ERROR` | Invalid field value |
| 403 | `FORBIDDEN` | Token doesn't match `{barber_id}` |
| 409 | `DUPLICATE_CONTACT_NUMBER` | New contact number already in use |

---

## 2. Shop Location & Discovery

### Set / update shop location

```
PATCH /api/v1/barbers/{barber_id}/location
```

**Auth:** Barber (must match `{barber_id}`)

**Request body**
```json
{ "lat": 12.9716, "lng": 77.5946 }
```

**Response `200 OK`**
```json
{ "barber_id": 12, "location": { "lat": 12.9716, "lng": 77.5946 } }
```

A **one-time (or rarely-edited) shop location** 
Captured once via the browser's Geolocation API while the Barber is in his
shop (typically right after registration), with a manual map-pin fallback
if he denies permission. No background job updates this; it only changes
if he deliberately edits it (e.g. relocating).

**Errors:** `400 VALIDATION_ERROR` (bad lat/lng), `403 FORBIDDEN`.

---

### Nearby search (how Customers find this Barber without a QR)

```
GET /api/v1/barbers/nearby?lat={lat}&lng={lng}&radius_km={radius_km}
```

**Auth:** none · **Called by:** Customer app, not the Barber, documented
here because it reads the Barber's `location` field set above.

**Query params:** `lat`, `lng` (required, Customer's position),
`radius_km` (optional, default `6`).

**Response `200 OK`**
```json
{ "data": [ { "barber_id": 12, "name": "Ravi Kumar", "distance_km": 1.8 } ] }
```

Only `subscription_status = active` Barbers are included. Computed in
application code (Haversine distance over all active Barbers) 
Tapping a result opens the same `GET /barbers/{barber_id}` +
`GET /barbers/{barber_id}/slots` page a QR scan would.

**Errors:** `400 VALIDATION_ERROR` — missing/invalid `lat`/`lng`.

---

## 3. Subscription & Payments

### Get subscription status

```
GET /api/v1/barbers/{barber_id}/subscription
```

**Auth:** Barber (must match `{barber_id}`)

**Response `200 OK`**
```json
{
  "subscription_status": "expired",
  "subscription_expiry_date": "2026-09-20",
  "renewal_amount": 499,
  "currency": "INR"
}
```

`subscription_status` (`trial` / `active` / `expired`) is computed
server-side from `account_created_at` and payment history, never a
manually-set flag. When `expired`, the Barber's app shows the renewal
screen below instead of the dashboard, and he drops out of nearby search.

**Errors:** `403 FORBIDDEN`.

---

### Start a renewal payment (Razorpay order)

```
POST /api/v1/barbers/{barber_id}/payments/create-order
```

**Auth:** Barber (must match `{barber_id}`)

**Request body:** none required (amount is server-determined, ₹499, not
client-supplied, prevents a tampered request paying a different amount).

**Response `201 Created`**
```json
{
  "razorpay_order_id": "order_KjX92mLp",
  "amount": 499,
  "currency": "INR",
  "razorpay_key_id": "rzp_live_xxxxx"
}
```

`amount` is in paise (Razorpay convention). The Barber's frontend feeds this
straight into the Razorpay Checkout SDK to open the payment sheet, the
API's job ends at handing over a valid order to pay against.

**Errors:** `403 FORBIDDEN`, `409 ALREADY_ACTIVE` (subscription isn't due
for renewal yet).

---

### Razorpay payment webhook

```
POST /api/v1/payments/webhook
```

**Auth:** Razorpay webhook signature (verified via `X-Razorpay-Signature`
header, not a Barber/Admin JWT), this is a server-to-server callback, not
something the Barber's app calls directly.

**Request body:** Razorpay's standard `payment.captured` event payload.

**Behavior:** on a verified, successful payment, the server:
1. Inserts a row into `Payments` (`amount`, `payment_date`,
   `payment_status = success`, `razorpay_transaction_id`).
2. Extends the Barber's `subscription_expiry_date` by 1 year and sets
   `subscription_status = active`.

**Response `200 OK`** empty body (Razorpay only checks the status code).

**Errors:** `400 INVALID_SIGNATURE` payload doesn't match the expected
webhook signature; request is discarded, no DB write occurs.

---

### Payment / renewal history

```
GET /api/v1/barbers/{barber_id}/payments
```

**Auth:** Barber (must match `{barber_id}`)

**Response `200 OK`**
```json
{
  "data": [
    { "payment_id": 5, "amount": 499, "payment_date": "2027-09-23", "payment_status": "success" }
  ]
}
```

**Errors:** `403 FORBIDDEN`.

---

## 4. Daily Slot Management

### View today's slots

```
GET /api/v1/barbers/{barber_id}/slots
```

**Auth:** none (public — the same call renders the Customer's booking page)

**Response `200 OK`**
```json
{
  "slot_date": "2026-09-23",
  "data": [
    { "slot_id": 501, "start_time": "10:00", "end_time": "10:30", "status": "available" },
    { "slot_id": 502, "start_time": "10:30", "end_time": "11:00", "status": "booked" },
    { "slot_id": 503, "start_time": "11:00", "end_time": "11:30", "status": "na" }
  ]
}
```

`status` ∈ `available`, `na`, `booked` (waiting), `approved`. Rows are
generated automatically by the daily TTL/regeneration job,
there is no `POST` to manually create a slot.

Frontends polling this endpoint for live-refresh (agreed auto-refresh
behavior) should use a **higher, dedicated rate limit** here than the
general guest limit, since this is the one endpoint expected to be polled
every 10–15 seconds

---

### Mark a slot NA (Barber busy)

```
PATCH /api/v1/barbers/{barber_id}/slots/{slot_id}
```

**Auth:** Barber (must match `{barber_id}`)

**Request body**
```json
{ "status": "na" }
```

**Response `200 OK`** updated slot object.

**Errors**
| Status | Code | When |
|---|---|---|
| 403 | `FORBIDDEN` | Token doesn't match `{barber_id}` |
| 409 | `SLOT_ALREADY_BOOKED` | Slot already has an active (waiting/approved) booking |

---

### Daily TTL / slot regeneration (system job, not a client-facing endpoint)

Runs once, just after midnight:
1. Delete **all** `Bookings` rows linked to yesterday's slots, regardless
   of `booking_status` (`waiting`, `approved`, `rejected`, `cancelled`,
   `completed` are all erased; no history is kept).
2. Delete all yesterday's `Slots` rows.
3. Generate a fresh set of `available` slots (10 AM–5 PM) for every Barber
   with `subscription_status = active`.

This is a scheduled cron/job-runner task, not a database-native TTL —
documented here since it's what makes `GET .../slots` and
`GET .../bookings` behave correctly across a day boundary.

---

## 5. Booking Management (Barber's actions on incoming bookings)

### List today's bookings

```
GET /api/v1/barbers/{barber_id}/bookings
```

**Auth:** Barber (must match `{barber_id}`)

**Response `200 OK`**
```json
{
  "data": [
    { "booking_id": 901, "slot_id": 502, "booking_status": "waiting", "created_at": "2026-09-23T10:05:00Z" },
    { "booking_id": 899, "slot_id": 498, "booking_status": "approved", "created_at": "2026-09-23T09:40:00Z" }
  ]
}
```

Customer details are **not** included in this list view, the Barber taps
into a specific booking to see who it is (next endpoint). Keeps the list
lightweight and avoids exposing customer data until it's actually needed.

**Errors:** `403 FORBIDDEN`.

---

### View a booking's customer details

```
GET /api/v1/bookings/{booking_id}/customer
```

**Auth:** Barber (must own the booking's parent Barber record)

**Response `200 OK`**
```json
{ "customer_id": 88, "name": "Anita Sharma", "phone": "+919812345678", "location": { "lat": 12.975, "lng": 77.601 } }
```

`location` replaces the earlier `address` field, the Barber's UI turns
this into a tappable map/navigation link rather than a text address.

**Errors:** `403 FORBIDDEN`, `404 NOT_FOUND` (booking already TTL-purged).

---

### Approve a booking

```
PATCH /api/v1/bookings/{booking_id}/approve
```

**Auth:** Barber (must own the booking)

**Response `200 OK`** — booking status → `approved`; linked slot status →
`approved`. Triggers an in-browser notification to the Customer.

**Errors**
| Status | Code | When |
|---|---|---|
| 403 | `FORBIDDEN` | Barber doesn't own this booking |
| 422 | `INVALID_STATE_TRANSITION` | Booking isn't currently `waiting` |

---

### Reject a booking

```
PATCH /api/v1/bookings/{booking_id}/reject
```

**Auth:** Barber (must own the booking)

**Response `200 OK`** — booking status → `rejected`; linked slot reopens to
`available`. Triggers an in-browser notification to the Customer.

**Errors:** same as approve, `422 INVALID_STATE_TRANSITION` if not
currently `waiting`.

---

### Mark a booking completed

```
PATCH /api/v1/bookings/{booking_id}/complete
```

**Auth:** Barber (must own the booking)

**Request body:** none.

**Response `200 OK`** — booking status → `completed`. This is a manual
Barber action (taps "mark as done" after finishing the haircut), there is
no automatic completion trigger (no GPS/payment confirmation in v1).

**Errors**
| Status | Code | When |
|---|---|---|
| 403 | `FORBIDDEN` | Barber doesn't own this booking |
| 422 | `INVALID_STATE_TRANSITION` | Booking isn't currently `approved` |

No booking history is retained, regardless of status. A
`completed` booking is purged at midnight exactly like `waiting`,
`rejected`, or `cancelled` ones — the TTL job (below) applies the same
blanket delete to all of yesterday's `Bookings` rows, with no status-based
exception. The Barber has no lookup of past jobs once a day rolls over;
`GET /barbers/{barber_id}/bookings` only ever reflects today.
