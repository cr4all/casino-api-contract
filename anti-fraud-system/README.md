# AFS Backend Integration Guide

Your **casino / iGaming backend** sits between the player and AFS. AFS never talks to players directly — your API is the integration layer.

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

**Yes — your backend should always send the same `CanonicalEvent` shape** for both sync (`POST /evaluate`) and async (`casino.events`). That keeps scoring consistent and avoids validation errors.

### Schema sources (single contract)

| Format | Location |
|--------|----------|
| **Live API** (authoritative) | `GET http://localhost:8001/integration/event-schema` |
| **JSON Schema** (request) | `http://localhost:8001/schemas/canonical-event.schema.json` |
| **JSON Schema** (sync response) | `http://localhost:8001/schemas/evaluate-response.schema.json` |
| **JSON Schema** (async action) | `http://localhost:8001/schemas/risk-action.schema.json` |
| **TypeScript** | `schemas/afs-types.d.ts` (also at `/schemas/afs-types.d.ts`) |
| **Python TypedDict** | `schemas/canonical_event.py` |
| **OpenAPI** | `http://localhost:8001/docs` |

Validate in your backend **before** calling AFS:

- **Node/TypeScript:** `ajv` + `canonical-event.schema.json`, or use `afs-types.d.ts`
- **Python:** `pydantic.TypeAdapter(CanonicalEvent)` using `schemas/canonical_event.py`, or copy the Risk `CanonicalEvent` model

### `CanonicalEvent` — request payload

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `event_id` | `string` | **yes** | Unique idempotency key (UUID recommended) |
| `event_type` | `enum` | **yes** | See [event types](#event-types) |
| `timestamp` | `string` (ISO-8601) | **yes** | UTC datetime, e.g. `2026-06-21T12:00:00Z` |
| `user` | `object` | **yes** | Player identity |
| `context` | `object` | **yes** | Device + network signals |
| `transaction` | `object` | **conditional** | Required for `deposit`, `withdrawal`, `bet` |
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

\*Strongly recommended — missing fingerprint on web signup/login increases risk score.

#### `transaction` object

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `amount` | `number \| null` | **yes**† | Amount in major units, e.g. `500.0` |
| `currency` | `string \| null` | **yes**† | ISO 4217, e.g. `EUR` |

†Required when `event_type` is `deposit`, `withdrawal`, or `bet`.

#### `metadata` object

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `channel` | `"web" \| "mobile" \| "api" \| null` | no | Client channel |
| `session_id` | `string \| null` | no | Your session id |
| `referrer` | `string \| null` | no | Signup referrer |
| `failure_reason` | `string \| null` | no | For `login_failed` / `signup_failed` |

#### Event types

| `event_type` | Description |
|--------------|-------------|
| `signup` | New account registration |
| `signup_failed` | Failed registration attempt |
| `login` | Successful authentication |
| `login_failed` | Failed login attempt |
| `deposit` | Player deposit |
| `withdrawal` | Player withdrawal |
| `bet` | Wager placed |

### `EvaluateResponse` — sync response

| Field | Type | Description |
|-------|------|-------------|
| `event_id` | `string` | Echo of request |
| `user_id` | `string` | Echo of request |
| `event_type` | `enum` | Echo of request |
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
| `event_type` | `string` | Original event type |
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
    event_type: "login",
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
        "event_type": "login",
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
    B -->|publish events| Q[RabbitMQ]
    Q -->|risk.actions| B
    R -->|decision| B
    B --> P
```

1. **Collect** device context from the client + server data (IP, country)
2. **Send** canonical events to AFS (sync and/or async)
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

Example payload:

```json
{
  "event_id": "550e8400-e29b-41d4-a716-446655440000",
  "event_type": "login",
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
| Registration | `signup` | **Sync** |
| Login | `login` | **Sync** |
| Deposit | `deposit` | **Sync** |
| Withdrawal | `withdrawal` | **Sync** (always before payout) |
| Login failed | `login_failed` | Async |
| Bet | `bet` | Sync or async |

Use a **unique UUID** for every `event_id`.

For `deposit`, `withdrawal`, and `bet`, include:

```json
"transaction": { "amount": 500.0, "currency": "EUR" }
```

---

## Step 3 — Async events (RabbitMQ)

Publish to exchange **`casino.events`** with routing keys like `player.login`, `player.login.failed`, `transaction.withdrawal`:

```python
import json, uuid
from datetime import datetime, timezone
import aio_pika

async def publish_event(routing_key: str, event: dict):
    conn = await aio_pika.connect_robust("amqp://afs:afs@localhost:5672/")
    async with conn:
        channel = await conn.channel()
        exchange = await channel.declare_exchange(
            "casino.events", aio_pika.ExchangeType.TOPIC, durable=True
        )
        await exchange.publish(
            aio_pika.Message(
                json.dumps(event).encode(),
                content_type="application/json",
                delivery_mode=aio_pika.DeliveryMode.PERSISTENT,
            ),
            routing_key=routing_key,
        )

# Failed login example
await publish_event("player.login.failed", {
    "event_id": str(uuid.uuid4()),
    "event_type": "login_failed",
    "timestamp": datetime.now(timezone.utc).isoformat(),
    "user": {"user_id": "player_123", "email": "maria@gmail.com"},
    "context": {"ip": "198.51.100.50", "country": "DE"},
    "metadata": {"failure_reason": "invalid_password", "channel": "web"},
})
```

AFS consumes from queue **`casino.afs`** automatically — you only publish.

Message body options:

- Flat canonical event (same as `POST /evaluate`)
- Wrapped envelope: `{ "payload": { ... } }`, `{ "data": { ... } }`, or `{ "event": { ... } }`
- `event_type` can be omitted if the routing key ends with a known suffix (`login`, `signup`, `withdrawal`, etc.)

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
                        throttle_ip(action["context"]["ip"])
                    case "allow":
                        pass  # no-op
```

| Decision | Typical action |
|----------|----------------|
| `allow` | Proceed |
| `challenge` + login | `require_mfa` |
| `challenge` + withdrawal | `manual_review_withdrawal` |
| `block` + withdrawal | `hold_withdrawal` |
| `block` + login | `block_login` |

Example action message:

```json
{
  "event_id": "withdrawal_001",
  "user_id": "player_123",
  "event_type": "withdrawal",
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
- [ ] Backend builds payloads matching `CanonicalEvent` schema
- [ ] Backend validates payload before send (JSON Schema / TypedDict / Zod)
- [ ] Backend calls `POST http://localhost:8001/evaluate`
- [ ] Backend enforces `allow` / `challenge` / `block`

**Full (sync + async)** — audit trail + async enforcement:

- [ ] Everything above
- [ ] Publish `login_failed` / `signup_failed` to `casino.events`
- [ ] Consumer on `risk.actions` in your backend

---

## Quick test from PowerShell

```powershell
curl -X POST http://localhost:8001/evaluate `
  -H "Content-Type: application/json" `
  -d '{"event_id":"test_001","event_type":"signup","timestamp":"2026-06-21T12:00:00Z","user":{"user_id":"player_123","email":"maria@gmail.com"},"context":{"ip":"203.0.113.10","country":"DE","fingerprint":"a1b2c3d4e5f6789012345678abcdef01"},"metadata":{"channel":"web"}}'
```

You should get JSON with `"decision": "allow"` or `"challenge"` or `"block"`.

---

## Summary

Your backend sits in the middle — it builds events from player + device data, calls **`localhost:8001/evaluate`** for real-time gates, optionally publishes to **`casino.events`** on RabbitMQ, and enforces decisions either from the HTTP response or from **`risk.actions`**. AFS handles all scoring; you never touch Postgres/Redis directly.
