# Installation

## Prerequisites

Install:

- Docker Engine
- Docker Compose

This public distribution is Docker-only. Node.js usage stays in the private source repo for development work.

## Fast Start

1. Copy `.env.example` to `.env`.
2. Copy `config/terminal-profiles.json.example` to `config/terminal-profiles.json`.
3. Copy `config/terminals.json.example` to `config/terminals.json`.
4. Review the example files in `config/`.
5. Pull the release image from GHCR or load the bundled tar archive.
6. Run `docker compose run --rm auth-init`.
7. Start the proxy with `docker compose up -d`.
8. Check logs and call `/health`.

For real terminal transactions, the Docker host must also expose the OPI device callback port, not just the HTTP API port. The default bundle publishes `4102` for that purpose. If you change `deviceChannel1` in terminal config, keep the published port aligned.

## Image Source

Online from GHCR:

```bash
docker compose pull
```

Offline from a separately distributed archive:

```bash
docker load -i image/nexi-opi-proxy-<version>.tar
```

After `docker load`, you can keep the same image reference in `.env` if the archive was built from the matching release tag.

The public repo itself stays GHCR-first. Large offline archives are kept out of git and should be attached separately when needed.

## Start The Service

```bash
docker compose up -d
```

Check logs:

```bash
docker compose logs -f proxy
```

For LAN-connected payment terminals, make sure:

- `DEVICE_CHANNEL1_PUBLIC_PORT` matches the terminal `deviceChannel1` port
- the host firewall allows inbound TCP on that port
- the terminal can reach the Docker host by IP

Default localhost health check:

```bash
curl http://127.0.0.1:3001/health
```

The default `.env.example` uses `TLS_MODE=auto`, so localhost can start without a certificate.

If you later switch to `TLS_MODE=required`, use `https://` instead and trust the certificate appropriately.

## First-Time Setup Helpers

Generate `config/auth.json` and the bootstrap admin token:

```bash
docker compose run --rm auth-init
```

The bootstrap admin bearer is printed directly in the terminal when the file is created.

Generate a self-signed development certificate for quick HTTPS testing:

```bash
docker compose run --rm tls-init-dev
```

That certificate is for testing only. Shared environments should use a proper certificate and key mounted into `certs/`.
