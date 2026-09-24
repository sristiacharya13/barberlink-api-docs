# Admin API

Base path: `/api/v1/admin`

Admin is deliberately minimal, per the agreed design: **read only
the list of Barbers and their subscription status**, no create/edit/delete
actions, no access to Customer or Booking data.

---

## 1. Login

Admin accounts are created directly in the database (e.g. by whoever
deploys the app), there is no public self-registration endpoint, unlike
Barber/Customer.

```
POST /api/v1/admin/login
```

**Auth:** none

**Request body**
```json
{ "email": "admin@barberapp.com", "password": "••••••••" }
```

**Response `200 OK`**
```json
{ "admin_id": 1, "name": "Ops Admin", "token": "eyJhbGciOi..." }
```

The app stores `token` and sends it as `Authorization: Bearer <token>` on
every subsequent call — same pattern as Barber/Customer, no repeated
login prompts on the same device.

**Errors:** `401 INVALID_CREDENTIALS` — wrong email/password combination.

---

## 2. List all Barbers

The one core capability Admin has.

```
GET /api/v1/admin/barbers
```

**Auth:** Admin

**Response `200 OK`**
```json
{
  "data": [
    {
      "barber_id": 12,
      "name": "Ravi Kumar",
      "contact_number": "+919876543210",
      "account_created_at": "2026-09-23T10:15:00Z",
      "subscription_status": "active",
      "subscription_expiry_date": "2027-09-23"
    },
    {
      "barber_id": 27,
      "name": "Suresh M",
      "contact_number": "+919812340000",
      "account_created_at": "2025-06-01T08:00:00Z",
      "subscription_status": "expired",
      "subscription_expiry_date": "2026-06-01"
    }
  ],
  "pagination": { "page": 1, "limit": 20, "total_items": 47, "total_pages": 3 }
}
```

No Customer, Slot, Booking, or Payment data is included or reachable from
here, Admin's visibility stops at this list.

**Errors**
| Status | Code | When |
|---|---|---|
| 401 | `UNAUTHORIZED` | Missing/invalid/expired token |

---
