# Changelog

All notable changes to the TakeSlot Partner API contract are documented here.
The API follows `/vN` versioning: within a major version changes are additive
only; anything breaking ships under a new major version.

## [Unreleased]

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
