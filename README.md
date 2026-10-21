# TakeSlot Partner API

Connect your business to the **TakeSlot** marketplace. TakeSlot is a last-minute
fitness & wellness booking marketplace; this API lets a business that runs its own
booking software publish its catalog and schedule to TakeSlot and receive the
bookings TakeSlot customers make — while TakeSlot handles payment.

This repository is the **source of truth** for the public API contract: the
OpenAPI document in [`openapi/partner-api.v1.yaml`](openapi/partner-api.v1.yaml)
and the reference docs generated from it. It evolves in parallel with the TakeSlot
backend, which implements and is tested against this contract.

- **Base URL:** `https://api.takeslot.app/v1`
- **Spec:** OpenAPI 3.1 — `openapi/partner-api.v1.yaml`
- **Status:** v1 (additive changes only; breaking changes ship as `/v2`)

## What you can do

| Area | Capability |
| --- | --- |
| **Services** | Publish and update your catalog (classes/activities). |
| **Schedule** | Publish dated, timed slots and the number of seats you release to TakeSlot. |
| **Bookings** | Receive bookings via webhooks, reconcile them by polling, and cancel them. |

Setup — creating the business, locations, API keys and webhook endpoints — is done
in the TakeSlot **dashboard**, not through this API.

## Core concepts

- **Location** — your business. In v1 a location maps 1:1 to the business your API
  key belongs to.
- **Service** — a catalog item (e.g. "Vinyasa Yoga 60 min").
- **Slot** — a dated, timed instance of a service. `released_seats` is how many
  seats you offer on TakeSlot; it can be lower than the slot's real capacity.
- **Booking** — a confirmed reservation made by a TakeSlot customer. Bookings are
  confirmed instantly against your released seats (inventory model) — there is no
  accept/reject step.
- **Webhook** — a signed HTTP callback TakeSlot sends when a booking is created or
  cancelled.

TakeSlot collects payment from the customer (via Stripe) and pays you out through
your connected account; the booking carries the amount paid. You never receive the
customer's full contact details — only their first name and a booking confirmation
code.

## Authentication

Create an API key in the TakeSlot dashboard and send it as a bearer token:

```
Authorization: Bearer sk_live_xxxxxxxxxxxxxxxxxxxxxxxx
```

The key is shown once at creation. Every request is scoped to the business the key
belongs to. Keys carry scopes (`services:read`, `services:write`, `schedule:read`,
`schedule:write`, `bookings:read`, `bookings:write`); a request that needs a scope
the key lacks returns `403 insufficient_scope`.

## Quickstart

```bash
BASE=https://api.takeslot.app/v1
KEY=sk_live_xxx

# 1. Confirm the key
curl -H "Authorization: Bearer $KEY" $BASE/me

# 2. Publish a service (upsert on your external_id)
curl -X POST $BASE/services -H "Authorization: Bearer $KEY" -H 'Content-Type: application/json' -d '{
  "external_id": "sku-101",
  "name": "Vinyasa Yoga",
  "format": "group",
  "category": "yoga",
  "duration_min": 60,
  "capacity": 16,
  "full_price": 4900,
  "default_discount_pct": 50
}'

# 3. Publish a slot releasing 4 seats to TakeSlot
curl -X POST $BASE/schedule -H "Authorization: Bearer $KEY" -H 'Content-Type: application/json' -d '{
  "external_id": "evt-55012",
  "service_external_id": "sku-101",
  "date": "2026-10-25",
  "start_time": "18:30",
  "capacity": 16,
  "released_seats": 4
}'

# 4. Register a webhook endpoint in the dashboard, then poll as a fallback
curl -H "Authorization: Bearer $KEY" "$BASE/bookings?since=2026-10-21T00:00:00Z"
```

## Conventions

- **JSON** in and out. Money is in **minor units (grosze)**, currency `PLN`
  (`4900` = 49.00 PLN). Slot `date` is `YYYY-MM-DD` and `start_time` is `HH:mm` in
  **Europe/Warsaw**. Other timestamps are ISO 8601 UTC.
- **Upserts.** `POST /services` and `POST /schedule` upsert on your `external_id`,
  so re-sending the same id updates the record instead of duplicating it.
- **Idempotency.** `POST` requests accept an `Idempotency-Key` header.
- **Pagination.** List endpoints return `{ "data": [...], "meta": { "next_cursor",
  "has_more" } }`; pass `?limit=` (default 50, max 200) and `?cursor=`.
- **Errors.** `{ "error": { "code", "message", "details" } }` with a stable `code`.
- **Rate limits.** Per key; responses carry `RateLimit-*` headers, `429` with
  `Retry-After` when exceeded.

## Webhooks

TakeSlot `POST`s signed events to the URL you register in the dashboard:

| Event | When |
| --- | --- |
| `booking.created` | A customer booking is confirmed against your released seats. |
| `booking.cancelled` | A booking is cancelled (by the customer, you, or a slot deletion). |

Payload:

```json
{
  "id": "booking.created.ckxx…",
  "type": "booking.created",
  "created_at": "2026-10-21T17:03:00.100Z",
  "data": {
    "id": "bk_789",
    "confirmation_code": "A1B2C3",
    "status": "confirmed",
    "customer_first_name": "Marta",
    "slot": { "id": "slot_xyz", "external_id": "evt-55012", "service_id": "svc_abc", "date": "2026-10-25", "start_time": "18:30" },
    "amount": 2450,
    "currency": "PLN"
  }
}
```

### Verifying the signature

Each request carries a header:

```
TakeSlot-Signature: t=1766336580,v1=<hex HMAC-SHA256>
```

`v1` is `HMAC-SHA256(signing_secret, "{t}.{raw_request_body}")`. To verify:

1. Read `t` and `v1` from the header.
2. Recompute `HMAC-SHA256(secret, t + "." + rawBody)` over the exact raw body.
3. Compare in constant time, and reject if `t` is older than a few minutes.

```js
const crypto = require('crypto');
function verify(rawBody, header, secret) {
  const parts = Object.fromEntries(header.split(',').map((p) => p.split('=')));
  const expected = crypto.createHmac('sha256', secret).update(`${parts.t}.${rawBody}`).digest('hex');
  return crypto.timingSafeEqual(Buffer.from(expected), Buffer.from(parts.v1));
}
```

Delivery is **at-least-once** — treat the event `id` as idempotent and ignore
duplicates. Whether or not a webhook succeeds, every booking is also retrievable
from `GET /bookings?since=`, so you can rely on webhooks, polling, or both.

## Repository layout

```
openapi/partner-api.v1.yaml   # the contract (OpenAPI 3.1)
openapi/components/           # shared schemas
examples/                     # request/response and webhook samples
docs/                         # generated reference site
CHANGELOG.md                  # contract changes
```

## Versioning & support

`/v1` changes are additive. Breaking changes ship under a new major version. See
[`CHANGELOG.md`](CHANGELOG.md) for the history.
