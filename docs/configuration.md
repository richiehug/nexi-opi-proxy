# Configuration

## Runtime Files

The public bundle uses one `.env` file plus a small set of JSON files in `config/`.

- `.env`: runtime host, port, TLS, logging, and timeout settings
- `config/terminal-models.json`: Nexi-owned terminal model catalog for this release
- `config/terminal-profiles.json.example`: starting point for environment-specific terminal profiles
- `config/terminals.json.example`: starting point for the environment-specific terminal list
- `config/terminal-profiles.json`: your local working profile definitions after setup
- `config/terminals.json`: your local working terminal list after setup
- `config/auth.json`: generated during initialization and then owned by the environment

## Ownership Model

Treat the files like this:

- `terminal-models.json`: release-owned and safe to refresh from a newer Nexi release after review
- `terminal-profiles.json.example`: release-owned example
- `terminals.json.example`: release-owned example
- `terminal-profiles.json`: merchant or integrator owned after first setup
- `terminals.json`: merchant or integrator owned after first setup
- `auth.json`: generated locally and always environment owned

## `.env`

The most important settings are:

- `HOST`: bind address inside the container, normally `0.0.0.0`
- `PUBLISHED_HOST`: host-side exposure, usually `127.0.0.1` or `0.0.0.0`
- `PORT`: HTTP or HTTPS port
- `TLS_MODE`: `auto`, `required`, or `disabled`
- `TLS_CERT_FILE` and `TLS_KEY_FILE`: mounted certificate files
- `LOG_LEVEL`: `normal` or `extended`

Recommended starting points:

- local machine only: `PUBLISHED_HOST=127.0.0.1`
- remotely reachable: `PUBLISHED_HOST=0.0.0.0`
- easiest local first run: `TLS_MODE=auto`
- non-local traffic: `TLS_MODE=required`

## Terminal Models, Profiles, and Terminals

The effective terminal config is merged in this order:

1. request values
2. terminal-specific values from `terminals.json`
3. profile values from `terminal-profiles.json`
4. model defaults from `terminal-models.json`
5. global defaults from `.env`

For public `1.0.0`, the official supported terminal models are:

- Ingenico RX5000
- PAX Q30

## Receipt Handling Defaults

Receipt routing is described through `receiptHandling`:

```json
{
  "customer": "external",
  "merchant": "external"
}
```

Allowed values:

- `external`: a connected external printer is available
- `local`: print locally on the terminal
- `unavailable`: printer is unavailable for that copy

This is the public JSON abstraction. The proxy maps it to the required OPI printer status fields internally.

## Payment Methods Filter

Request bodies can optionally include `paymentMethods` to restrict which brands are allowed for processing.

This controls processing behavior only. It does not control which brands the terminal screen shows to the customer.

Example:

```json
{
  "paymentMethods": ["visa", "mastercard", "maestro", "twint"]
}
```
