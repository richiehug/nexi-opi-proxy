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

The official supported terminal models in the public release line are:

- Ingenico RX5000
- Ingenico DX8000
- PAX Q30
- PAX IM15
- PAX IM30

`workstationId` is the POS/ECR identity sent to the terminal as OPI `WorkstationID`. Use a stable, unique value for each checkout or terminal context, for example `lane-1`, `richie-rx`, or `mobile-dx`. If `workstationId` is omitted, the proxy uses the terminal alias as the fallback.

## Receipt Handling Defaults

Receipt routing is described through `receiptHandling`:

```json
{
  "customer": "external",
  "merchant": "external"
}
```

Allowed values:

- `external`: the ECR/proxy is available for receipt output; receipts are sent over the device callback channel and stored by the proxy under `data/receipts`
- `local`: the terminal should handle that receipt locally, for example with its built-in printer
- `unavailable`: no printer or receipt destination is available for that copy

This is the public JSON abstraction. The proxy maps it to the required OPI printer status fields internally.

For standalone terminals with built-in printers, such as DX8000, the model default is `local` for customer and merchant receipts. That sends `PrintLocal` on card-service requests and keeps `E-JournalStatus` available for the ECR. In ECR/OPI mode, the terminal may send a cardholder receipt question over the device callback channel; the proxy answers yes by default so the payment context can finalize.

Use `external` when receipts should be captured by the proxy under `data/receipts` instead of printed by the terminal. In that mode, the terminal sends `Printer`, `JournalPrinter`, and `E-Journal` callbacks when available, and the proxy stores the receipt bodies with the transaction. When an ECR chooses external receipt handling for a terminal with a built-in printer, the ECR is responsible for asking whether the customer receipt should be printed, shown, sent, or skipped.

If `ports` are omitted from `terminals.json`, the proxy uses the model defaults, normally `payChannel0=4100` and `deviceChannel1=4102`. Multiple terminals may use the same default callback port; the proxy keeps one shared listener per host/port and routes device callbacks by `WorkstationID`. Keep `workstationId` unique for terminals that share a callback port.

Service requests such as init/config do not use OPI printer status fields. Any service receipt or message sent back by the terminal is handled through the ECR/device callback channel.

## Payment Methods Filter

Request bodies can optionally include `paymentMethods` to restrict which brands are allowed for processing.

This controls processing behavior only. It does not control which brands the terminal screen shows to the customer.

Example:

```json
{
  "paymentMethods": ["visa", "mastercard", "maestro", "twint"]
}
```
