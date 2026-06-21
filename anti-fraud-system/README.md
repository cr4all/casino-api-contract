# AFS Backend Integration Guide

Your **casino / iGaming backend** sits between the player and AFS. AFS never talks to players directly — your API is the integration layer.

**Event names** follow the casino platform contract (single source of truth):

https://github.com/cr4all/casino-api-contract/blob/main/asyncapi/platform-events-v1.yaml

Pinned copy in this repo: `contracts/platform-events-v1.yaml` (v1.2.0). Check the GitHub URL before releases — new platform events are added there first.

When AFS runs via Docker on your machine, your backend connects over **localhost ports**. Your backend does **not** connect to Postgres or Redis directly — those are internal to AFS.

---

## Table of contents

1. [Connection points (Docker)](#connection-points-docker)
2. [Canonical event payload (typed schema)](#canonical-event-payload-typed-schema)
3. [Your backend's 4 jobs](#your-backends-4-jobs)
4. [Step 1 — Collect context](#step-1--collect-context)
5. [Step 2 — Sync gates (required for critical flows)](#step-2--sync-gates-required-for-critical-flows)
6. [Step 3 — Async events (RabbitMQ)](#step-3--async-events-rabbitmq)
7. [Step 4 — Consume actions (async enforcement)](#step-4--consume-actions-async-enforcement)
8. [Minimal vs full integration](#minimal-vs-full-integration)
9. [Quick test from PowerShell](#quick-test-from-powershell)

---

## Canonical event payload (typed schema)

**Yes — your backend should always use the same `event_type` strings** as the platform contract. That keeps sync, async, and validation consistent.

### Two payload shapes

| Flow | Format | When |
|------|--------|------|
| **Sync auth gates** | AFS `CanonicalEvent` JSON | `POST /evaluate` for `player.signup`, `player.login`, etc. |
| **Async platform events** | `PlatformEventEnvelope` from contract | RabbitMQ `casino.events` for `payment.deposit`, `wallet.bet`, etc. |

AFS accepts **both** on the async consumer and maps platform envelopes to its internal scoring model automatically.

### Schema sources (single contract)

| Format | Location |
|--------|----------|
| **Live API** (authoritative) | `GET http://localhost:8001/integration/event-schema` |
| **JSON Schema** (request) | `http://localhost:8001/schemas/canonical-event.schema.json` |
| **JSON Schema** (sync response) | `http://localhost:8001/schemas/evaluate-response.schema.json` |
| **JSON Schema** (async action) | `http://localhost:8001/schemas/risk-action.schema.json` |
| **TypeScript** | `schemas/afs-types.d.ts` (also at `/schemas/afs-types.d.ts`) |
| **Python TypedDict** | `schemas/canonical_event.py` |
| **Platform contract** | `contracts/platform-events-v1.yaml` |
| **OpenAPI** | `http://localhost:8001/docs` |

Validate in your backend **before** calling AFS:

- **Node/TypeScript:** `ajv` + `canonical-event.schema.json`, or use `schemas/afs-types.d.ts`
- **Python:** `pydantic.TypeAdapter(CanonicalEvent)` using `schemas/canonical_event.py`

### Event types (aligned with platform contract)

**Platform contract** — async `casino.events` (routing key **must equal** `event_type`):

| `event_type` | Scored by AFS? | Description |
|--------------|----------------|-------------|
| `wallet.bet` | yes | Wallet bet debited |
| `game.bet` | yes | Game bet (non-wallet, e.g. free spin) |
| `payment.deposit` | yes | Deposit completed |
| `payment.withdraw` | yes | Withdrawal approved |
| `bonus.created` | yes | Bonus granted |
| `wallet.win` | no (skipped) | Win credited |
| `wallet.rollback` | no (skipped) | Wallet rollback |
| `bonus.completed` | no (skipped) | Bonus completed |
| `affiliate.commission` | no (skipped) | Affiliate commission |
| `notification.send` | no (skipped) | Notification dispatch |

**Auth gates** — sync `POST /evaluate` (not in platform contract yet):

| `event_type` | Scored by AFS? | Description |
|--------------|----------------|-------------|
| `player.signup` | yes | Registration |
| `player.signup.failed` | yes | Failed registration |
| `player.login` | yes | Successful login |
| `player.login.failed` | yes | Failed login |

**Legacy aliases** (sync `/evaluate` only — migrate to new names):

| Old name | New name |
|----------|----------|
| `signup` | `player.signup` |
| `signup_failed` | `player.signup.failed` |
| `login` | `player.login` |
| `login_failed` | `player.login.failed` |
| `deposit` | `payment.deposit` |
| `withdrawal` | `payment.withdraw` |
| `bet` | `wallet.bet` |

### `PlatformEventEnvelope` — async platform events

Use this shape when publishing wallet/payment/bonus events (from contract):

```json
{
  "event_id": "550e8400-e29b-41d4-a716-446655440002",
  "event_type": "payment.deposit",
  "occurred_at": "2026-06-21T11:00:00+00:00",
  "version": "1.0",
  "data": {
    "player_id": 123,
    "amount": "500.0000",
    "deposit_id": 45,
    "ledger_id": 800,
    "currency": "USD"
  }
}
```

| Contract field | AFS mapping |
|----------------|-------------|
| `event_id` | `event_id` |
| `event_type` | `event_type` (must match routing key) |
| `occurred_at` | `timestamp` |
| `data.player_id` | `user.user_id` (string) |
| `data.amount` | `transaction.amount` (parsed from `"500.0000"`) |
| `data.currency` | `transaction.currency` |
| `data.metadata.ip` | `context.ip` (when present) |

Publish with routing key **`payment.deposit`** (same as `event_type`).

### `CanonicalEvent` — sync `/evaluate` and auth async

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `event_id` | `string` | **yes** | Unique idempotency key (UUID v4 recommended) |
| `event_type` | `string` | **yes** | Contract name — see tables above |
| `timestamp` | `string` (ISO-8601) | **yes** | UTC datetime, e.g. `2026-06-21T12:00:00Z` |
| `user` | `object` | **yes** | Player identity |
| `context` | `object` | **yes** | Device + network signals |
| `transaction` | `object` | **conditional** | Required for money event types |
| `metadata` | `object` | no | Channel, session, failure reason |

#### `user` object

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `user_id` | `string` | **yes** | Your platform player id |
| `email` | `string \| null` | no | Email address |
| `phone` | `string \| null` | no | E.164 or local format |
| `name` | `string \| null` | no | Display / legal name |

#### `context` object

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `ip` | `string \| null` | no* | Client IP (set server-side) |
| `country` | `string \| null` | no* | ISO 3166-1 alpha-2, e.g. `DE` |
| `fingerprint` | `string \| null` | no* | FingerprintJS `visitorId` (32 hex chars) |
| `fingerprint_version` | `string \| null` | no | FingerprintJS version, e.g. `v4.2.0` |
| `user_agent` | `string \| null` | no | Browser / client UA |
| `device_id` | `string \| null` | no | Native app device id (use instead of fingerprint on mobile) |
| `browser_language` | `string \| null` | no | e.g. `de-DE` |
| `timezone` | `string \| null` | no | IANA timezone, e.g. `Europe/Berlin` |
| `platform` | `string \| null` | no | e.g. `Win32`, `iPhone` |
| `screen_resolution` | `string \| null` | no | e.g. `1920x1080` |

\*Strongly recommended for `player.signup` / `player.login` — missing fingerprint on web increases risk score.

#### `transaction` object

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `amount` | `number \| null` | **yes**† | Amount in major units, e.g. `500.0` |
| `currency` | `string \| null` | **yes**† | ISO 4217, e.g. `EUR` |

†Required when `event_type` is `payment.deposit`, `payment.withdraw`, `wallet.bet`, or `game.bet`.

#### `metadata` object

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `channel` | `"web" \| "mobile" \| "api" \| null` | no | Client channel |
| `session_id` | `string \| null` | no | Your session id |
| `referrer` | `string \| null` | no | Signup referrer |
| `failure_reason` | `string \| null` | no | For `player.login.failed` / `player.signup.failed` |

### `EvaluateResponse` — sync response

| Field | Type | Description |
|-------|------|-------------|
| `event_id` | `string` | Echo of request |
| `user_id` | `string` | Echo of request |
| `event_type` | `string` | Echo of request (contract name) |
| `decision` | `"allow" \| "challenge" \| "block"` | **Enforce this** |
| `final_score` | `integer` (0–100) | Aggregated score |
| `risk_level` | `"low" \| "medium" \| "high" \| "critical"` | Score band |
| `engines` | `object` | Per-engine scores + signal names |
| `latency_ms` | `integer` | Processing time |
| `source` | `"sync" \| "async"` | Always `sync` for `/evaluate` |
| `scored_at` | `string` (ISO-8601) | When scored |

### `RiskActionMessage` — async action (queue `risk.actions`)

| Field | Type | Description |
|-------|------|-------------|
| `event_id` | `string` | Original event id |
| `user_id` | `string` | Player id |
| `event_type` | `string` | Original event type (contract name) |
| `decision` | `"allow" \| "challenge" \| "block"` | Risk decision |
| `final_score` | `integer` | Aggregated score |
| `risk_level` | `enum` | Score band |
| `action` | `string` | **Enforce this** — e.g. `require_mfa`, `hold_withdrawal` |
| `signals` | `string[]` | Triggered fraud signals |
| `processed_at` | `string` (ISO-8601) | When orchestrated |

### TypeScript example (typed builder)

```typescript
import type { CanonicalEvent } from "./schemas/afs-types";

function buildLoginEvent(userId: string, ctx: CanonicalEvent["context"]): CanonicalEvent {
  return {
    event_id: crypto.randomUUID(),
    event_type: "player.login",
    timestamp: new Date().toISOString(),
    user: { user_id: userId, email: "maria@gmail.com" },
    context: ctx,
    metadata: { channel: "web" },
  };
}
```

### Python example (typed builder)

```python
from datetime import datetime, timezone
from uuid import uuid4

def build_login_event(user_id: str, context: dict) -> dict:
    return {
        "event_id": str(uuid4()),
        "event_type": "player.login",
        "timestamp": datetime.now(timezone.utc).isoformat(),
        "user": {"user_id": user_id, "email": "maria@gmail.com"},
        "context": context,
        "metadata": {"channel": "web"},
    }
```

---

## Connection points (Docker)

| What your backend calls | URL / address |
|-------------------------|---------------|
| **Risk API (sync)** | `http://localhost:8001/evaluate` |
| **Risk health / docs** | `http://localhost:8001/health` · `http://localhost:8001/docs` |
| **Event schema** | `http://localhost:8001/integration/event-schema` |
| **Publish async events** | RabbitMQ `amqp://afs:afs@localhost:5672/` → exchange `casino.events` |
| **Consume enforcement actions** | RabbitMQ queue `risk.actions` |
| **RabbitMQ UI** (debug) | http://localhost:15672 (`afs` / `afs`) |

Your backend does **not** connect to Postgres or Redis directly — those are internal to AFS.

---

## Your backend's 4 jobs

```mermaid
flowchart LR
    P[Player] --> B[Your backend]
    B -->|POST /evaluate| R[Risk :8001]
    B -->|publish casino.events| Q[RabbitMQ]
    Q -->|risk.actions| B
    R -->|decision| B
    B --> P
```

1. **Collect** device context from the client + server data (IP, country)
2. **Send** events with contract `event_type` names (sync and/or async)
3. **Enforce** `allow` / `challenge` / `block` before the action completes (sync)
4. **Subscribe** to `risk.actions` for async enforcement (optional but recommended)

---

## Step 1 — Collect context

**Web:** frontend loads FingerprintJS, sends to your API:

```javascript
const device = await collectDeviceContext(); // fingerprint, UA, timezone, …
await fetch("/api/login", {
  method: "POST",
  body: JSON.stringify({ email, password, riskContext: device }),
});
```

**Your backend adds server-side fields:**

| Field | Source |
|-------|--------|
| `context.ip` | Request IP |
| `context.country` | Geo / user profile |
| `context.fingerprint` | From frontend (web) |
| `context.device_id` | Native app |
| `user.user_id`, email, phone | Your user record |

For web clients, serve FingerprintJS and the AFS helper from Risk:

```html
<script src="https://openfpcdn.io/fingerprintjs/v4/iife.min.js"></script>
<script src="http://localhost:8001/client/fingerprint-collector.js"></script>
```

Schema reference: `GET http://localhost:8001/integration/device-context` · full payload types in [Canonical event payload](#canonical-event-payload-typed-schema)

---

## Step 2 — Sync gates (required for critical flows)

Call Risk **before** completing signup, login, deposit, or withdrawal:

```http
POST http://localhost:8001/evaluate
Content-Type: application/json
```

Example — `player.login`:

```json
{
  "event_id": "550e8400-e29b-41d4-a716-446655440000",
  "event_type": "player.login",
  "timestamp": "2026-06-21T12:00:00Z",
  "user": {
    "user_id": "player_123",
    "email": "maria@gmail.com"
  },
  "context": {
    "ip": "203.0.113.10",
    "country": "DE",
    "fingerprint": "a1b2c3d4e5f6789012345678abcdef01",
    "user_agent": "Mozilla/5.0 ...",
    "timezone": "Europe/Berlin"
  },
  "metadata": {
    "channel": "web",
    "session_id": "sess_abc"
  }
}
```

Example — `payment.withdraw` (gate before payout):

```json
{
  "event_id": "550e8400-e29b-41d4-a716-446655440001",
  "event_type": "payment.withdraw",
  "timestamp": "2026-06-21T14:00:00Z",
  "user": { "user_id": "player_123", "email": "maria@gmail.com" },
  "context": {
    "ip": "203.0.113.10",
    "country": "DE",
    "fingerprint": "a1b2c3d4e5f6789012345678abcdef01"
  },
  "transaction": { "amount": 500.0, "currency": "EUR" },
  "metadata": { "channel": "web" }
}
```

**Enforce the response:**

```python
result = response.json()

if result["decision"] == "block":
    raise HTTPException(403, "Blocked by risk")

if result["decision"] == "challenge":
    return {"status": "mfa_required"}  # OTP, captcha, KYC step-up

# allow → issue session / process payment
```

| Player action | `event_type` | Mode |
|---------------|--------------|------|
| Registration | `player.signup` | **Sync** `/evaluate` |
| Login | `player.login` | **Sync** `/evaluate` |
| Deposit | `payment.deposit` | **Sync** `/evaluate` |
| Withdrawal | `payment.withdraw` | **Sync** `/evaluate` (always before payout) |
| Login failed | `player.login.failed` | Async `casino.events` |
| Signup failed | `player.signup.failed` | Async `casino.events` |
| Wallet bet | `wallet.bet` | Async `casino.events` (platform envelope) |
| Game bet | `game.bet` | Async `casino.events` (platform envelope) |

Use a **unique UUID** for every `event_id`.

---

## Step 3 — Async events (RabbitMQ)

### Platform events (wallet, payment, bonus)

Your casino backend already publishes `PlatformEventEnvelope` to **`casino.events`**. AFS subscribes on queue **`casino.afs`** and scores:

- `wallet.bet`, `game.bet`, `payment.deposit`, `payment.withdraw`, `bonus.created`

Non-scored events (`wallet.win`, `notification.send`, etc.) are **acked and skipped**.

**Routing key must equal `event_type`** (per contract):

```python
import json
import aio_pika

async def publish_platform_event(envelope: dict):
    routing_key = envelope["event_type"]  # e.g. payment.deposit
    conn = await aio_pika.connect_robust("amqp://afs:afs@localhost:5672/")
    async with conn:
        channel = await conn.channel()
        exchange = await channel.declare_exchange(
            "casino.events", aio_pika.ExchangeType.TOPIC, durable=True
        )
        await exchange.publish(
            aio_pika.Message(
                json.dumps(envelope).encode(),
                content_type="application/json",
                delivery_mode=aio_pika.DeliveryMode.PERSISTENT,
            ),
            routing_key=routing_key,
        )

await publish_platform_event({
    "event_id": "550e8400-e29b-41d4-a716-446655440002",
    "event_type": "payment.deposit",
    "occurred_at": "2026-06-21T11:00:00+00:00",
    "version": "1.0",
    "data": {
        "player_id": 123,
        "amount": "500.0000",
        "deposit_id": 45,
        "ledger_id": 800,
        "currency": "USD",
    },
})
```

### Auth failure events (not in platform contract yet)

Publish `CanonicalEvent` shape with routing key = `event_type`:

```python
import uuid
from datetime import datetime, timezone

await publish_platform_event({
    "event_id": str(uuid.uuid4()),
    "event_type": "player.login.failed",
    "occurred_at": datetime.now(timezone.utc).isoformat(),
    "version": "1.0",
    "data": {
        "player_id": 123,
        "failure_reason": "invalid_password",
    },
})
```

Or use flat `CanonicalEvent` JSON with routing key `player.login.failed` — AFS accepts both.

---

## Step 4 — Consume actions (async enforcement)

Orchestrator maps Risk decisions → actions on queue **`risk.actions`**:

```python
async def consume_actions():
    conn = await aio_pika.connect_robust("amqp://afs:afs@localhost:5672/")
    channel = await conn.channel()
    queue = await channel.declare_queue("risk.actions", durable=True)

    async with queue.iterator() as it:
        async for message in it:
            async with message.process():
                action = json.loads(message.body)
                match action["action"]:
                    case "require_mfa":
                        trigger_mfa(action["user_id"])
                    case "hold_withdrawal":
                        flag_withdrawal(action["event_id"])
                    case "reject_signup":
                        block_registration(action["user_id"])
                    case "rate_limit_ip":
                        throttle_ip(action["user_id"])
                    case "allow":
                        pass  # no-op
```

| Decision | `event_type` | Typical `action` |
|----------|--------------|------------------|
| `allow` | any | `allow` |
| `challenge` | `player.login` | `require_mfa` |
| `challenge` | `payment.withdraw` | `manual_review_withdrawal` |
| `challenge` | `player.signup` | `step_up_verification` |
| `block` | `payment.withdraw` | `hold_withdrawal` |
| `block` | `player.login` | `block_login` |
| `block` | `player.signup` | `reject_signup` |
| `block` | `player.login.failed` | `rate_limit_ip` |
| `block` | `payment.deposit` | `block_deposit` |
| `block` | `wallet.bet` / `game.bet` | `block_bet` |

Example action message:

```json
{
  "event_id": "550e8400-e29b-41d4-a716-446655440001",
  "user_id": "player_123",
  "event_type": "payment.withdraw",
  "decision": "block",
  "final_score": 85,
  "risk_level": "high",
  "action": "hold_withdrawal",
  "signals": ["new_device_on_withdrawal"],
  "processed_at": "2026-06-21T12:00:02Z"
}
```

---

## Minimal vs full integration

**Minimum (sync only)** — enough to gate login/signup/withdrawal:

- [ ] Frontend sends fingerprint / device context to your API
- [ ] Backend builds payloads with contract `event_type` names
- [ ] Backend validates payload before send (JSON Schema / TypedDict)
- [ ] Backend calls `POST http://localhost:8001/evaluate` with `player.*` / `payment.*` types
- [ ] Backend enforces `allow` / `challenge` / `block`

**Full (sync + async)** — audit trail + async enforcement:

- [ ] Everything above
- [ ] Platform publishes `PlatformEventEnvelope` to `casino.events` (or AFS consumes existing feed)
- [ ] Publish `player.login.failed` / `player.signup.failed` to `casino.events`
- [ ] Consumer on `risk.actions` in your backend

---

## Quick test from PowerShell

```powershell
# Auth gate — player.signup
curl -X POST http://localhost:8001/evaluate `
  -H "Content-Type: application/json" `
  -d '{"event_id":"test_001","event_type":"player.signup","timestamp":"2026-06-21T12:00:00Z","user":{"user_id":"player_123","email":"maria@gmail.com"},"context":{"ip":"203.0.113.10","country":"DE","fingerprint":"a1b2c3d4e5f6789012345678abcdef01"},"metadata":{"channel":"web"}}'

# Money gate — payment.withdraw
curl -X POST http://localhost:8001/evaluate `
  -H "Content-Type: application/json" `
  -d '{"event_id":"test_002","event_type":"payment.withdraw","timestamp":"2026-06-21T14:00:00Z","user":{"user_id":"player_123"},"context":{"ip":"203.0.113.10","country":"DE","fingerprint":"a1b2c3d4e5f6789012345678abcdef01"},"transaction":{"amount":500,"currency":"EUR"},"metadata":{"channel":"web"}}'
```

You should get JSON with `"decision": "allow"`, `"challenge"`, or `"block"`.

Check live schema (includes scored/skipped lists from contract):

```powershell
curl http://localhost:8001/integration/event-schema
```

---

## Summary

Your backend uses **contract `event_type` names** everywhere:

- **Sync gates:** `POST /evaluate` with `player.signup`, `player.login`, `payment.deposit`, `payment.withdraw`
- **Async platform:** `PlatformEventEnvelope` on `casino.events` (`wallet.bet`, `payment.deposit`, …)
- **Async auth failures:** `player.login.failed`, `player.signup.failed`
- **Enforcement:** HTTP `decision` (sync) or `risk.actions` queue (async)

AFS handles scoring, trusted devices, and audit — you never touch Postgres/Redis directly. When the platform contract adds new events, update `shared/contracts/platform_events.py` from the [GitHub YAML](https://github.com/cr4all/casino-api-contract/blob/main/asyncapi/platform-events-v1.yaml) first.
