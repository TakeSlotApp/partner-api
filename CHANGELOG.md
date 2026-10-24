# Changelog

All notable changes to the TakeSlot Partner API contract are documented here.
The API follows `/vN` versioning: within a major version changes are additive
only; anything breaking ships under a new major version.

## [Unreleased]

### v1 — business-level keys & multiple studios (additive)

- API keys and webhook endpoints now belong to the **business** and cover every
  one of its studios. Keys are created self-serve in the dashboard
  (Business → Partner API). Existing keys keep working and now reach all studios
  of their business.
- New `GET /v1/studios` lists the studios a key can act on.
- `studio_id` on `Service`, `Slot` and `Booking` (and so on webhook payloads).
- `studio_id` on `POST /v1/services`, `POST /v1/schedule` and bulk slots. Optional
  while the business has one studio; otherwise `400 studio_required`. A slot
  created with `service_id` follows its service's studio (`400 studio_mismatch`
  on conflict). A foreign studio returns `404 studio_not_found`.
- `?studio_id=` filter on `GET /v1/services`, `/v1/schedule` and `/v1/bookings`.
- `external_id` is unique **per studio**.
- `GET /v1/me` returns `business { id, name }` and `studios`; `studio` is
  deprecated (the only studio, or `null` when there are several).
- `openapi/partner-api.v1.yaml` is now generated from the backend's served
  contract, bringing it back in sync (fuller descriptions and examples).

### v1 — initial

- Meta: `GET /v1/ping`, `GET /v1/me`, `GET /v1/reference/categories`,
  `GET /v1/reference/formats`.
- Services: `GET/POST/PATCH/DELETE /v1/services` (upsert on `external_id`).
- Schedule: `GET/POST/PATCH/DELETE /v1/schedule`, `POST /v1/schedule/bulk`
  (upsert on `external_id`, `released_seats` inventory).
- Bookings: `GET /v1/bookings` (poll via `since`), `GET /v1/bookings/{id}`,
  `POST /v1/bookings/{id}/cancel` (privacy-safe projection).
- Webhooks: `booking.created`, `booking.cancelled`, HMAC-SHA256 signed
  (`TakeSlot-Signature`).
