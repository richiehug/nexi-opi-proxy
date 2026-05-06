# Authentication

## How Auth Works

The proxy protects all terminal, transaction, and receipt routes with bearer-token authentication.

The only public route is:

- `GET /health`

All other routes require an `Authorization: Bearer ...` header.

## Initialization

Generate the auth file and bootstrap admin token:

```bash
docker compose run --rm auth-init
```

This creates `config/auth.json`.

The bootstrap admin bearer is printed directly in the terminal when the file is created.

## What `auth.json` Contains

`config/auth.json` stores:

- bearer token hashes, not raw token secrets
- enabled scopes for each token
- terminal access groups and explicit terminal access

The bootstrap admin token is intended to get the environment running. After that, create narrower tokens for day-to-day integrations.

## Scopes

The key public scope groups are:

- `terminals:read`
- `terminals:write`
- `transactions:read`
- `transactions:write`
- `receipts:read`
- `admin:manage`

## Example Header

```http
Authorization: Bearer nexi_opi_bridge_xxxxxxxxxxxxxxxxxxxxxxxx
```
