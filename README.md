# Nexi OPI Proxy

Nexi OPI Proxy gives integrators a cleaner way to work with Nexi payment terminals in Switzerland.

Instead of dealing with OPI XML, TCP sockets, receipt routing, and device messaging directly, you run one Docker service and talk to a small local HTTP and JSON API.

This release bundle is for `1.0.1`.

When this bundle is published into the public distribution repo, the repository root represents this release snapshot and older versions are tracked through Git tags and GitHub Releases.

## Why It Exists

- simpler local integration for POS, middleware, kiosk, and demo setups
- Docker-first installation and upgrades
- transaction and receipt storage outside the container
- bearer-token authentication with scoped access
- TLS-ready runtime model for non-local deployments

## Official Terminal Scope For 1.0.0

Nexi officially positions this public release for:

- Ingenico RX5000
- PAX Q30

Other terminal models may exist outside this public release scope, but they are not part of the official support statement for `1.0.0`.

## Supported Brands

<p>
  <img src="https://raw.githubusercontent.com/richiehug/payment-icons/main/icons/cards/visa-light.svg" alt="Visa" height="30" />
  <img src="https://raw.githubusercontent.com/richiehug/payment-icons/main/icons/cards/mastercard-light.svg" alt="Mastercard" height="30" />
  <img src="https://raw.githubusercontent.com/richiehug/payment-icons/main/icons/cards/maestro-light.svg" alt="Maestro" height="30" />
  <img src="https://raw.githubusercontent.com/richiehug/payment-icons/main/icons/cards/american-express-light.svg" alt="American Express" height="30" />
  <img src="https://raw.githubusercontent.com/richiehug/payment-icons/main/icons/cards/jcb-light.svg" alt="JCB" height="30" />
  <img src="https://raw.githubusercontent.com/richiehug/payment-icons/main/icons/cards/diners-light.svg" alt="Diners" height="30" />
  <img src="https://raw.githubusercontent.com/richiehug/payment-icons/main/icons/cards/discover-light.svg" alt="Discover" height="30" />
  <img src="https://raw.githubusercontent.com/richiehug/payment-icons/main/icons/other/twint-light.svg" alt="TWINT" height="30" />
</p>

Brand filtering in the API controls which brands can be processed for a request. It does not change what the terminal UI shows. The terminal display remains driven by terminal configuration.

## Quick Start

1. Copy `.env.example` to `.env`.
2. Copy `config/terminal-profiles.json.example` to `config/terminal-profiles.json`.
3. Copy `config/terminals.json.example` to `config/terminals.json`.
4. Review `config/terminal-models.json`, `config/terminal-profiles.json`, and `config/terminals.json`.
5. Choose how to get the image:
   `docker compose pull` for GHCR
   or `docker load -i image/nexi-opi-proxy-1.0.1.tar` if an offline archive was provided separately
6. Run `docker compose run --rm auth-init`.
7. Start the proxy with `docker compose up -d`.
8. Check `docker compose logs -f proxy`.

The default `.env.example` is set up for an easy localhost start with `TLS_MODE=auto`. That means:

- localhost can start on HTTP without a certificate
- non-local exposure still requires TLS unless you intentionally override it

Local health check:

```bash
curl http://127.0.0.1:3001/health
```

If you switch to HTTPS, use `https://` and trust or ignore the development certificate as needed.

## Repository Layout

- `.env.example`: runtime environment template
- `.gitignore`: protects runtime-owned local files from being tracked accidentally
- `docker-compose.yml`: Docker entrypoint for the proxy and setup tools
- `config/terminal-models.json`: Nexi-owned terminal model catalog for this release
- `config/terminal-profiles.json.example`: merchant or integrator profile examples
- `config/terminals.json.example`: merchant or integrator terminal examples
- `config/terminal-profiles.json`: your local working profile file after setup
- `config/terminals.json`: your local working terminal file after setup
- `config/auth.json`: generated during `auth-init`
- `data/`: persisted transaction and receipt data
- `logs/`: application logs and optional OPI traces
- `image/`: optional offline Docker archive location when distributed separately
- `docs/`: setup, configuration, API, storage, and upgrade guidance
- `openapi.yaml`: OpenAPI description of the public HTTP API

## API Highlights

The public API intentionally stays small:

- `GET /health`
- `GET /terminals/:terminalAlias/info`
- `GET /terminals/:terminalAlias/status`
- `POST /terminals/:terminalAlias/payment`
- `POST /terminals/:terminalAlias/refund`
- `POST /terminals/:terminalAlias/transmit`
- `POST /terminals/:terminalAlias/close`
- `POST /terminals/:terminalAlias/config`
- `POST /terminals/:terminalAlias/init`
- `POST /transactions/:transactionId/capture`
- `POST /transactions/:transactionId/cancel`
- `POST /transactions/:transactionId/refund`
- `GET /transactions/:transactionId`
- `GET /transactions/:transactionId/receipts`

Receipts are stored per transaction, so a transaction lookup can lead you directly to its receipt bundle.

## Read Next

- [Getting Started](docs/installation.md)
- [Configuration](docs/configuration.md)
- [Authentication](docs/authentication.md)
- [API Overview](docs/api.md)
- [Storage and Receipts](docs/storage-and-receipts.md)
- [Security and TLS](docs/security-tls.md)
- [Upgrading](docs/upgrading.md)
- [Release Notes](docs/release-notes.md)
