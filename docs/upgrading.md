# Upgrading

## Upgrade Flow

1. Back up `config/` and `data/`.
2. Review the release notes for the target version.
3. Pull the new image or load the new archive.
4. Keep your existing `config/auth.json`.
5. Preserve your environment-owned `terminal-profiles.json` and `terminals.json`.
6. Review whether `terminal-models.json` should be refreshed from the new release.
7. Restart with `docker compose up -d`.
8. Validate logs, health, and a real terminal flow.

## 1.0.3 Storage And Async Notes

Runtime fallbacks remain `STORAGE=json` and `RESPONSE_MODE=sync` if those values are omitted. The shipped `.env.example` now recommends `STORAGE=db` and `RESPONSE_MODE=async` for new deployments. Existing deployments can upgrade without moving data.

To use SQLite, set:

```env
STORAGE=db
DB_URL=file:/app/data/opi-proxy.sqlite
```

Then migrate existing JSON transactions and receipts:

```bash
docker compose run --rm proxy node src/index.js storage migrate --from json --to db
docker compose run --rm proxy node src/index.js storage verify
```

Async mode is recommended for integrations where client timeouts should not determine the terminal flow outcome:

```env
RESPONSE_MODE=async
```

`status=unknown` is intentionally conservative and means the proxy could not prove the final terminal result.

Mutating routes also support `Idempotency-Key` for safe client retries. Reusing the same key with the same request content returns the same proxy response. Reusing the same key with different content returns HTTP `409`.

## File Ownership During Upgrades

Preserve:

- `config/auth.json`
- `config/terminal-profiles.json`
- `config/terminals.json`
- `data/`
- `logs/`
- `certs/`

Review and optionally refresh:

- `config/terminal-models.json`
- `config/terminal-profiles.json.example`
- `config/terminals.json.example`
- `.env.example`
- `openapi.yaml`
- documentation files

## Commands

Pull the new image:

```bash
docker compose pull
```

Or load the replacement archive:

```bash
docker load -i image/nexi-opi-proxy-<version>-linux-arm64.tar
```

Restart:

```bash
docker compose up -d
```
