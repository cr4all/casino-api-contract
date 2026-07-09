# Marketing CRM Backend Integration Guide

Your **casino-backend** publishes platform events to the CRM broker and exposes internal REST endpoints so **casino-crm** can execute notifications and bonus grants.

**Contracts (single source of truth):**

- REST: `openapi/crm-integration-v1.yaml`
- Events: `asyncapi/crm-events-v1.yaml`
- Platform events consumed by CRM: `asyncapi/platform-events-v1.yaml`

When CRM runs via Docker on your machine, backend connects over **localhost ports**. Backend does **not** connect to CRM Postgres directly.

---

## Connection points (Docker)

| Service | Host port | Purpose |
|---------|-----------|---------|
| CRM API | 8002 | Sync REST (`/v1/*`, `/health`) |
| CRM Admin | 5174 | Standalone React admin |
| CRM PostgreSQL | 5433 | CRM OLTP (internal) |
| CRM RabbitMQ AMQP | 5673 | Platform event fan-out |
| CRM RabbitMQ UI | 15673 | Management console |
| CRM Redis | 6380 | Segment cache / locks |

---

## Backend responsibilities

1. **Publish platform events** to CRM RabbitMQ (`casino.events` topic exchange) after commit.
2. **Expose internal CRM API** at `/api/internal/crm/*` with `X-Service-Key` auth.
3. **Optional:** consume `crm.actions.execute` queue for async CRM actions.

### Environment (backend `.env`)

```env
CRM_ENABLED=true
CRM_BASE_URL=http://127.0.0.1:8002
CRM_API_KEY=crm-dev-service-key
CRM_RABBITMQ_HOST=127.0.0.1
CRM_RABBITMQ_PORT=5673
CRM_RABBITMQ_USER=crm
CRM_RABBITMQ_PASSWORD=secret
CRM_RABBITMQ_PUBLISH_ENABLED=true
CRM_RABBITMQ_CONSUME_ENABLED=true
CRM_SERVICE_KEY=platform-dev-service-key
```

### Environment (CRM `.env`)

```env
CRM_SERVICE_KEY=crm-dev-service-key
PLATFORM_BASE_URL=http://127.0.0.1:8000
PLATFORM_SERVICE_KEY=platform-dev-service-key
DATABASE_URL=postgresql://crm:secret@127.0.0.1:5433/crm?schema=public
RABBITMQ_HOST=127.0.0.1
RABBITMQ_PORT=5673
```

---

## Consumed platform events (MVP)

| event_type | CRM use |
|------------|---------|
| `auth.signup` | Create player profile, lifecycle journeys |
| `login_success` | Activity / dormant segment updates |
| `payment.deposit` | Update deposit metrics, first-deposit campaigns |
| `payment.withdraw` | Update withdraw metrics |
| `wallet.bet` | Bet count / turnover segments |
| `wallet.win` | Win metrics |
| `bonus.created` | Campaign conversion tracking |
| `player.vip.promoted` | Sync `platform_vip_level` on player profile |

Full envelope: see `platform-events-v1.yaml`. Idempotency key: `event_id`.

Backend publishes only the events listed above to CRM RabbitMQ when `CRM_RABBITMQ_PUBLISH_ENABLED=true` (see `CrmEventTypes` whitelist).

---

## Internal API (CRM → Platform)

| Method | Path | Purpose |
|--------|------|---------|
| GET | `/api/internal/crm/health` | Connection test |
| GET | `/api/internal/crm/vip-levels` | VIP level catalog (0–6) |
| POST | `/api/internal/crm/vip/promote` | Promote player VIP level |
| POST | `/api/internal/crm/notifications/send` | Send internal/email message |
| POST | `/api/internal/crm/bonus/grant` | Grant bonus by policy ID |
| GET | `/api/internal/crm/players/{id}/summary` | Player snapshot |

Header: `X-Service-Key: {CRM_SERVICE_KEY}` (backend validates).

---

## Sync API (Platform → CRM)

| Method | Path | Purpose |
|--------|------|---------|
| GET | `/health` | CRM health |
| GET | `/v1/players/{id}/segments` | Segment membership lookup |
| POST | `/v1/segments/evaluate` | Recompute segments |

Header: `X-Service-Key: {CRM_API_KEY}` (CRM validates).

---

## Local dev workflow

```bash
# 1. CRM infrastructure
cd casino-crm
docker compose -f docker/docker-compose.yml up -d
cp .env.example .env
npm install
npm run db:push
npm run db:seed
npm run dev

# 2. Platform backend (separate terminal)
cd casino-backend
# set CRM_* env vars in .env
php artisan crm:consume-actions
composer dev
```

Default CRM admin: `admin@crm.local` / `admin123`

---

## VIP levels (backend source of truth)

| Level | Name |
|-------|------|
| 0 | Regular |
| 1 | Bronze |
| 2 | Silver |
| 3 | Gold |
| 4 | Platinum |
| 5 | Diamond |
| 6 | VIP |

CRM promotes via Segment rules + Campaign action `vip.promote` with `{ "targetLevel": N }`.

### Segment rule metrics for VIP promotion

| Rule field | Meaning |
|------------|---------|
| `total_deposit` | Lifetime deposit sum (USD) |
| `lifetime_turnover` | Lifetime bet turnover (`wallet.bet` amount sum) |
| `monthly_turnover` | Current UTC month bet turnover (resets on 1st) |
| `platform_vip_level` | Current VIP level (use `< targetLevel` to avoid re-promotion) |

### Segment rule fields (CRM internal DSL)

Rules use AND logic. Operators: `eq`, `neq`, `gt`, `gte`, `lt`, `lte`, `contains`.

| Category | Fields |
|----------|--------|
| monetary | `total_deposit`, `total_withdraw`, `withdraw_count`, `total_win`, `win_count`, `net_deposit`*, `net_player_result`*, `avg_bet_size`* |
| turnover | `lifetime_turnover`, `monthly_turnover`, `cash_turnover`, `bonus_turnover`, `monthly_cash_turnover`, `monthly_bonus_turnover`, `total_bet`, `bet_count` |
| lifecycle | `deposit_count`, `days_since_registration`*, `days_since_last_activity`*, `days_since_last_deposit`* |
| engagement | `bonus_count`, `status` |
| vip | `platform_vip_level` |

\*Derived at evaluation time from stored `PlayerProfile` metrics.

Event sources for metrics: `payment.deposit`, `payment.withdraw`, `wallet.bet`, `wallet.win`, `bonus.created`, `player.vip.promoted`, `auth.signup`.

---

## Defaults

- Currency: **USD**
- Language: **en**
