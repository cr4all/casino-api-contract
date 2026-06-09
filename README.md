# Casino API Contract

OpenAPI specification for the Casino Platform **Player API** (`/api/v1`).

This repository is the **single source of truth** for frontend/backend integration.

## Files

| File | Description |
|------|-------------|
| `openapi/player-v1.yaml` | Player-facing REST API (OpenAPI 3.1) |
| `CHANGELOG.md` | Contract version history |

## Versioning

- **MAJOR**: breaking changes (field removal, type change, auth change)
- **MINOR**: new endpoints or optional fields
- **PATCH**: documentation / example fixes only

Tag releases: `v1.0.0`, `v1.1.0`, ...

## Workflow

1. Open a PR in **this repo** first when changing the API
2. Get review from backend + frontend developers
3. Merge and tag a new version
4. Backend and frontend implement against the tagged contract

## Local commands

```bash
npm install
npm run lint
npm run bundle
npm run preview
```

## Consumers

- **Backend**: [casino-backend](https://github.com/cr4all/casino-backend) — implement + Feature Tests
- **Frontend**: [casino-frontend](https://github.com/cr4all/casino-frontend) — `openapi-typescript` type generation

## Related

Human-readable architecture docs live in the backend repo under `docs/`.
Markdown `06-api-specification.md` is deprecated in favor of this OpenAPI file.
