# Barber API

Base path: `/api/v1/barbers`

## Register a Barber

Triggered when a Barber scans the **universal QR** and submits the
registration form. Creates the Barber record and starts the free-trial
subscription clock.

```
POST /api/v1/barbers
```

**Auth:** none

**Request body**
```json
{
  "name": "Ravi Kumar",
  "contact_number": "+919876543210"
}
```

**Response `201 Created`**
```json
{
  "barbar_id": 12,
  "name": "Ravi Kumar",
  "contact_number": "+919876543210",
  "qr_code": "https://api.barberapp.com/qr/b12x9f.png",
  "account_created_at": "2026-09-23T10:15:00Z",
  "subscription_status": "trial",
  "subscription_expiry_date": "2027-09-23"
}
```

`qr_code` is generated server-side after the record is created, since it
encodes the new `barbar_id`.

---

## Get Barber profile / dashboard

```
GET /api/v1/barbars/{barbar_id}
```

**Auth:** Barber (must match `{barbar_id}`), or public subset when called
from a Customer's QR-scan landing page (see note below).

**Response `200 OK`**
```json
{
  "barbar_id": 12,
  "name": "Ravi Kumar",
  "contact_number": "+919876543210",
  "qr_code": "https://api.barberapp.com/qr/b12x9f.png",
  "location": { "lat": 12.9716, "lng": 77.5946, "updated_at": "2026-09-23T10:40:00Z" },
  "subscription_status": "active",
  "subscription_expiry_date": "2027-09-23"
}
```

**Note:** when a Customer scans the Barber's personal QR, the frontend calls
this same endpoint to render the booking page, but the API returns a
reduced public subset (`barbar_id`, `name`, `location`) with no auth — full
profile fields require the Barber's own token.

---

## Update Barber profile / location

```
PATCH /api/v1/barbars/{barbar_id}
```

**Auth:** Barber (must match `{barbar_id}`)

**Request body** (any subset)
```json
{
  "location": { "lat": 12.9720, "lng": 77.5950 }
}
```

**Response `200 OK`** — updated Barber object.

Used for the periodic (30–60s) location ping described in the product
design, and for editing name/contact number.

---

## Get subscription status

```
GET /api/v1/barbars/{barbar_id}/subscription
```

**Auth:** Barber (must match `{barbar_id}`)

**Response `200 OK`**
```json
{
  "subscription_status": "expired",
  "subscription_expiry_date": "2026-09-20",
  "renewal_amount": 499,
  "currency": "INR"
}
```

When `subscription_status` is `expired`, the Barber's frontend shows the
renewal payment screen instead of the dashboard
