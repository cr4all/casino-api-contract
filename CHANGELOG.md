# Changelog

All notable changes to the Player API contract are documented here.

Format follows [Semantic Versioning](https://semver.org/).

## [1.2.0] - 2026-06-16

### Added

- `sort_order` field on `GameVendor` schema returned by `GET /games/vendors`

## [1.1.0] - 2026-06-12

### Added

- Required `phone` field on `POST /auth/register` request body (E.164 format)

## [1.0.0] - 2026-06-09

### Added

- Initial OpenAPI 3.1 spec for `/api/v1` Player API
- Auth (register, login, refresh, logout)
- Player profile endpoints
- Wallet balance and transactions
- Payment methods, deposits, withdrawals
- Bonus available, active, claim
- Notifications messages
- Game catalog (vendors, types, collections, list, detail, launch)

Derived from `routes/api.php` and Feature Tests in the backend monorepo.
