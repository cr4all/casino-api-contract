# Platform Events Contract

RabbitMQ message schemas for integration between the Casino Platform and external systems (e.g. AFS).

**Source of truth:** [`asyncapi/platform-events-v1.yaml`](../asyncapi/platform-events-v1.yaml)

## Quick reference

| event_type | Routing key | Required `data` fields |
|------------|-------------|------------------------|
| `wallet.bet` | `wallet.bet` | `player_id`, `amount`, `ledger_id`, `reference_type`, `reference_id` |
| `wallet.win` | `wallet.win` | same as bet |
| `wallet.rollback` | `wallet.rollback` | `player_id`, `amount`, `ledger_id`, `original_ledger_id`, `reference_type`, `reference_id` |
| `payment.deposit` | `payment.deposit` | `player_id`, `amount`, `deposit_id`, `ledger_id`, `currency` |
| `payment.withdraw` | `payment.withdraw` | `player_id`, `amount`, `withdrawal_id`, `ledger_id`, `currency` |
| `bonus.created` | `bonus.created` | `player_id`, `bonus_id`, `amount` (+ optional `ledger_id`, `type`, `metadata`) |
| `affiliate.commission` | `affiliate.commission` | `affiliate_id`, `player_id`, `commission_id`, `amount`, `type` |
| `notification.send` | `notification.send` | `channel`, `template` (+ optional `player_id`, `data`) |

Every message uses the same envelope:

```json
{
  "event_id": "uuid",
  "event_type": "wallet.bet",
  "occurred_at": "2026-06-21T10:30:00+00:00",
  "version": "1.0",
  "data": { }
}
```

## AFS subscription

Recommended queue setup on the shared broker:

```
Exchange: casino.events (topic)
Queue:    casino.afs
Binding:  #   (all platform events)
```

Or bind only the namespaces you need:

```
wallet.#, payment.#, bonus.#, affiliate.#, notification.#
```

## Change workflow

Event contract changes follow the same PR-first process as the REST API:

1. Open a PR in **casino-api-contract** updating `asyncapi/platform-events-v1.yaml` and `CHANGELOG.md`.
2. Bump `info.version` in the AsyncAPI file (semver).
3. Get review from platform backend and AFS owners.
4. Merge and tag (e.g. `platform-events-v1.1.0`).
5. Implement payload changes in `casino-backend` (`EventPublisher::publish` call sites).
6. Update AFS consumers to match the tagged contract.

### When to bump version

| Change | Semver |
|--------|--------|
| New optional `data` field | MINOR |
| New `event_type` | MINOR |
| Remove/rename field or `event_type` | MAJOR |
| Doc/example fix only | PATCH |

### Backend sync checklist

When adding or changing an event in the backend:

- [ ] Update `asyncapi/platform-events-v1.yaml` (channel + schema + examples)
- [ ] Update `CHANGELOG.md` under **Platform Events**
- [ ] Update publisher table in AsyncAPI `info.description` if a new Action publishes the event
- [ ] Run `npm run lint:events`

## Tooling

```bash
npm install
npm run lint:events
npm run preview:events
```

## Related

- Backend architecture: `casino-backend/docs/10-event-driven.md`
- REST API contract: `openapi/player-v1.yaml`
