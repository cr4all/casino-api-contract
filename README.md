# Casino API Contract

Integration contracts for the Casino Platform.

This repository is the **single source of truth** for:

- **Player REST API** (`/api/v1`) — OpenAPI
- **Platform events** (RabbitMQ / AFS) — AsyncAPI

## Files

| File | Description |
|------|-------------|
| `openapi/player-v1.yaml` | Player-facing REST API (OpenAPI 3.0) |
| `asyncapi/platform-events-v1.yaml` | RabbitMQ platform events (AsyncAPI 2.6) |
| `docs/platform-events.md` | Event contract workflow and quick reference |
| `CHANGELOG.md` | Contract version history (REST + events) |

## Versioning

### REST API (`openapi/player-v1.yaml`)

- **MAJOR**: breaking changes (field removal, type change, auth change)
- **MINOR**: new endpoints or optional fields
- **PATCH**: documentation / example fixes only

Tag releases: `v1.0.0`, `v1.1.0`, ...

### Platform events (`asyncapi/platform-events-v1.yaml`)

- **MAJOR**: remove/rename `event_type`, or breaking payload change
- **MINOR**: new event type or new optional `data` fields
- **PATCH**: documentation / example fixes only

Tag releases: `platform-events-v1.0.0`, `platform-events-v1.1.0`, ...

## Workflow

### REST API

1. Open a PR in **this repo** first when changing the API
2. Get review from backend + frontend developers
3. Merge and tag a new version
4. Backend and frontend implement against the tagged contract

### Platform events (AFS / RabbitMQ)

1. Open a PR in **this repo** first when adding, changing, or removing an event
2. Update `asyncapi/platform-events-v1.yaml` and `CHANGELOG.md`
3. Get review from backend + AFS owners
4. Merge and tag (`platform-events-v*`)
5. Implement in `casino-backend` (`EventPublisher`) and AFS consumers

See [docs/platform-events.md](./docs/platform-events.md) for details.

## Local commands

```bash
npm install
npm run lint          # lint REST + events
npm run lint:api
npm run lint:events
npm run bundle
npm run preview
npm run preview:events
```

## Consumers

- **Backend**: [casino-backend](https://github.com/cr4all/casino-backend) — REST + `EventPublisher` payloads
- **Frontend**: [casino-frontend](https://github.com/cr4all/casino-frontend) — `openapi-typescript` type generation
- **AFS**: subscribes to `casino.events` exchange per AsyncAPI contract

## Related

Human-readable architecture docs live in the backend repo under `docs/` (see `10-event-driven.md`).
Markdown `06-api-specification.md` is deprecated in favor of this OpenAPI file.
