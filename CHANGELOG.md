# Changelog

All notable changes to the Player API contract are documented here.

Format follows [Semantic Versioning](https://semver.org/).

## [1.4.2] - 2026-06-16

### Added

- `PUT /player/password` — change password for authenticated player

## [1.4.1] - 2026-06-16

### Added

- `phone`, `country_name`, and `language` on `GET /player/me` and `PATCH /player/profile` (`PlayerMe` schema)

### Changed

- `PATCH /player/profile` response now returns the full `PlayerMe` shape (not only nickname/language)

## [1.4.0] - 2026-06-16

### Added

- Phone-based password recovery via `phone` field on `POST /auth/password-recovery/request` and `POST /auth/password-recovery/reset` (E.164, Twilio SMS)

### Changed

- `PasswordRecoveryRequestBody` and `PasswordRecoveryResetBody` accept exactly one of `email` or `phone` (email-only `required` removed)

## [1.3.0] - 2026-06-16

### Added

- `POST /auth/password-recovery/request` — request email verification code for password recovery
- `POST /auth/password-recovery/reset` — reset password with email verification code
- Schemas `PasswordRecoveryRequestBody`, `PasswordRecoveryResetBody`

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
