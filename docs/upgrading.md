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
docker load -i image/nexi-opi-proxy-<version>.tar
```

Restart:

```bash
docker compose up -d
```
