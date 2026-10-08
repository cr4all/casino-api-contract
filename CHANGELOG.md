# Changelog

All notable changes to integration contracts in this repository are documented here.

Format follows [Semantic Versioning](https://semver.org/).

---

## Hub Internal API

### [hub-internal-1.0.0] - 2026-10-07

**Spec:** `openapi/hub-internal-v1.yaml`

**Added:**

Initial contract for the `casino-game-hub` service-to-service REST API.

**Public game catalog (no auth):**

| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/v1/games` | Paginated game list with vendor / type / collection / search filters |
| GET | `/api/v1/games/{id}` | Game detail by Hub internal ID |
| GET | `/api/v1/games/vendors` | Active vendor list (optional `?type` filter) |
| GET | `/api/v1/games/types` | Active game type list |
| GET | `/api/v1/games/collections/{slug}` | Curated collection with embedded games |

**Launch (requires `X-Hub-Service-Key`):**

| Method | Path | Description |
|--------|------|-------------|
| POST | `/api/v1/games/{id}/launch` | Launch session by Hub game ID |
| POST | `/api/v1/games/launch` | Launch session by `provider_slug` + `external_code` |

**Admin / sync (requires `X-Hub-Service-Key`):**

| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/v1/providers` | All registered providers sorted by slug |
| GET | `/api/v1/catalog/export` | Full catalog snapshot (providers, vendors, halls, games) |

**Schemas:** `Game`, `GameVendor`, `GameVendorRef`, `GameType`, `GameTypeRef`, `GameProviderRef`, `GameCollection`, `GameLaunchRequest`, `GameLaunchByCodeRequest`, `GameLaunchResult`, `GameProvider`, `CatalogExport`, `CatalogProvider`, `CatalogVendor`, `CatalogHall`, `CatalogGame`, `Pagination`, `SuccessEnvelope`, `ErrorEnvelope`

Derived from `routes/api.php`, `GameController`, `LaunchGameAction`, `FormatGameAction`, `ListGameVendorsAction`, `ListGameTypesAction`, `GetGameCollectionAction`, `ListProvidersAction`, `ExportGameCatalogAction` in `casino-game-hub`.

---

## Hub Wallet Callback API

### [hub-wallet-callback-1.0.1] - 2026-10-07

**Spec:** `openapi/hub-wallet-callback-v1.yaml`

**Changed:**

- Callback paths are the full Platform path. Admin Wallet URL is the origin; Hub appends `/api/internal/game-hub`, so each call is `{origin}/api/internal/game-hub/wallet/{operation}`.

### [hub-wallet-callback-1.0.0] - 2026-10-07

**Spec:** `openapi/hub-wallet-callback-v1.yaml`

**Added:**

Initial contract for the outgoing wallet callbacks that `casino-game-hub` sends
to the Platform (`casino-backend`) during active game sessions.

```
Game Provider ──► Hub /provider/{slug}/*
                      │  (callbacks defined here)
                      ▼
               Platform /wallet/{endpoint}
```

**Endpoints (Platform must implement, Hub calls with `X-Hub-Service-Key`):**

| Method | Path | Description |
|--------|------|-------------|
| POST | `/wallet/balance` | Get player playable balance |
| POST | `/wallet/bet` | Debit bet from player wallet |
| POST | `/wallet/win` | Credit win to player wallet |
| POST | `/wallet/refund` | Refund / rollback a previous bet |
| POST | `/wallet/transfer-in` | Recover provider balance → Platform (GameBoy only) |
| POST | `/wallet/transfer-out` | Fund provider wallet from Platform (GameBoy only) |

**Special status codes:**

- `422 INSUFFICIENT_BALANCE` — player cannot cover the bet
- `409` — duplicate `reference_id`; Hub treats as idempotent success using `data` in response
- `404` — original transaction not found (refund only)

**Schemas:** `WalletRequestBase`, `WalletTransactionBase`, `WalletBalanceRequest`, `WalletBalanceData`, `WalletBetRequest`, `WalletBetData`, `WalletWinRequest`, `WalletWinData`, `WalletRefundRequest`, `WalletRefundData`, `WalletTransferRequest`, `WalletTransferData`, `DecimalAmount`, `SuccessEnvelope`, `ErrorEnvelope`

Derived from `HttpWalletGateway`, `WalletGatewayInterface`, and `BetFundingSource` enum in `casino-game-hub`.

---

## Platform Events

### [crm-integration-1.1.0] - 2026-07-08

**Spec:** `openapi/crm-integration-v1.yaml`

**Added:**

- VIP level catalog `GET /api/internal/crm/vip-levels` (0=Regular .. 6=VIP)
- VIP promotion `POST /api/internal/crm/vip/promote`
- Player summary fields `vipLevel`, `vipLevelName`
- CRM segment rule fields: `total_deposit`, `lifetime_turnover`, `monthly_turnover`, `platform_vip_level`

### [crm-events-1.0.0] - 2026-07-08

**Spec:** `asyncapi/crm-events-v1.yaml`, `openapi/crm-integration-v1.yaml`

**Added:**

- Marketing CRM integration contract (REST internal API + CRM event topology)
- CRM consumes subset of platform events on dedicated RabbitMQ broker

### [platform-events-1.5.0] - 2026-06-21

**Spec:** `asyncapi/platform-events-v1.yaml`

**Changed (breaking for AFS security consumers):**

- Security audit events no longer use routing key / `event_type` `security.audit`
- Each audit event is published with its own routing key: `login_success`, `login_failed`, `login_lockout`, `refresh_success`, `refresh_failed`, `refresh_token_reuse`, `admin_login_success`, `admin_login_failed`, `admin_login_lockout`, `security_settings_updated`
- Envelope `event_type` and `data.audit_event` must match the routing key

### [platform-events-1.4.0] - 2026-06-21

**Spec:** `asyncapi/platform-events-v1.yaml`

**Added:**

- `auth.signup` — player registration completed (RegisterUserAction)

### [platform-events-1.3.0] - 2026-06-21

**Spec:** `asyncapi/platform-events-v1.yaml`

**Added:**

- `security.audit` — superseded in 1.5.0 by per-event routing keys (see 1.5.0 changelog)

### [platform-events-1.2.0] - 2026-06-21

**Spec:** `asyncapi/platform-events-v1.yaml`

**Added:**

- `bonus.completed` — player bonus requirements fulfilled (wagering met or free spins exhausted)

### [platform-events-1.1.0] - 2026-06-19

**Spec:** `asyncapi/platform-events-v1.yaml`

**Added:**

- `game.bet` — analytics event for bets that do not debit the wallet (e.g. free spin)
- Optional `funding_source` (`cash` | `free_spin`) on `wallet.bet` and `wallet.win` payloads

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

## [1.11.0] - 2026-07-23

### Added

- `PaymentOption.kind` value `credit_card` — SmilePayz hosted credit card deposit (USD / EUR)

## [1.10.0] - 2026-07-16

### Added

- Game favorites: `GET /games/favorites`, `POST /games/{id}/favorite`, `DELETE /games/{id}/favorite`

## [1.9.0] - 2026-07-15

### Added

- Affiliate payout endpoints: `GET /affiliate/payouts/availability`, `GET|PUT /affiliate/payout-details`, `GET|POST /affiliate/payouts`
- Schemas: `AffiliatePayoutAvailability`, `AffiliatePayoutDetails`, `UpdateAffiliatePayoutDetailsRequest`, `AffiliatePayout`, `AffiliatePayoutList`
- `AffiliateStats.available_payout`, `AffiliateStats.accruing_commission`
- `AffiliateCommission.status` value `reserved`

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
