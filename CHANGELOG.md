# Changelog

All notable changes to integration contracts in this repository are documented here.

Format follows [Semantic Versioning](https://semver.org/).

## Platform Events

### [platform-events-1.0.0] - 2026-06-21

Initial RabbitMQ event contract for Casino Platform ↔ AFS integration.

**Spec:** `asyncapi/platform-events-v1.yaml`

**Events:**

| event_type | Description |
|------------|-------------|
| `wallet.bet` | Wallet bet debit |
| `wallet.win` | Wallet win credit |
| `wallet.rollback` | Wallet transaction rollback |
| `payment.deposit` | Deposit completed |
| `payment.withdraw` | Withdrawal completed |
| `bonus.created` | Bonus granted (cash or free spin) |
| `affiliate.commission` | Affiliate commission recorded |
| `notification.send` | Outbound notification request (consumer defined; publisher TBD) |

Derived from `casino-backend` → `EventPublisher`, domain Actions, and Consumers.

---

## Player REST API

## [1.5.0] - 2026-06-18

### Added

- `GET /payment/countries` — countries available for deposit/withdraw with default country
- `GET /payment/deposit-options` — unified deposit options per country (local, crypto, manual)
- `GET /payment/withdraw-options` — unified withdraw options per country
- `PaymentOption`, `PaymentCountry`, `PaymentDestinationField` schemas

### Removed (breaking)

- `GET /payment/methods`
- `GET /payment/crypto/currencies`
- `PaymentMethod`, `SmilePayzSupportedCountry`, `CryptoCurrency`, `CryptoCurrencyList` schemas

### Changed (breaking)

- `GET /payment/deposits/quote` — uses `option_key`, `amount`, `country` instead of `payment_method_id` / `local_country` / `pay_currency`
- `POST /payment/deposits` — uses `option_key`, `amount`, `country`
- `POST /payment/withdrawals` — uses `option_key`, `amount`, `country`, `destination`

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
