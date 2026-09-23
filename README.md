# BarberLink API Documentation

This folder documents the REST API for the BarberLink website.

## Index

| File | Covers |
|---|---|
| [01-barbar-api.md](./01-barbar-api.md) | Barbar registration, profile, QR code, location |

## Base URL

```
https://api.barberapp.com/api/v1
```

## Versioning

URL path versioning (`/api/v1/...`). This is chosen over header/query-param
versioning because it is the most explicit and cache-friendly option, and is
easy for a small client base (Barber web app, Customer web app) to adopt.
Breaking changes ship as `/api/v2/...`; the previous version stays live
until clients migrate.

## Resource naming conventions

- Resources are **plural nouns**: `/barbars`, `/customers`, `/slots`, `/bookings`, `/payments`.
- No verbs in URLs. Actions that don't fit plain CRUD (e.g. approving a
  booking) are modeled as a `PATCH` to a sub-resource/action path, e.g.
  `PATCH /bookings/{booking_id}/approve`, not `/approveBooking`.
- Path parameters use `snake_case` (`{barbar_id}`, `{slot_id}`) to match the
  DB column names directly avoids translation bugs between API and DB layers.
- JSON body/response fields use `snake_case` for the same reason.
- Nesting reflects ownership: a slot belongs to a barbar
  (`/barbars/{barbar_id}/slots`), a booking is made against a slot
  (`/barbars/{barbar_id}/slots/{slot_id}/bookings`). Nesting is kept to a
  maximum of 2 levels to avoid unreadable deep paths.

## Response format

All successful responses return JSON with the resource(s) directly in the
body (no unnecessary envelope):

```json
{
  "barbar_id": 12,
  "name": "Ravi Kumar",
  "subscription_status": "active"
}
```
