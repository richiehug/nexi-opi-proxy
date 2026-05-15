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
- `PUBLISHED_HOST`: legacy bridge-network host-side exposure; ignored by the default public Docker bundle because it uses `network_mode: host`
- `PORT`: HTTP or HTTPS port
- `TLS_MODE`: `auto`, `required`, or `disabled`
- `TLS_CERT_FILE` and `TLS_KEY_FILE`: mounted certificate files
- `LOG_LEVEL`: `normal` or `extended`
- `STORAGE`: `json` or `db`; default is `json`
- `DB_URL`: SQLite URL for `STORAGE=db`, default `file:/app/data/opi-proxy.sqlite`
- `RESPONSE_MODE`: `sync` or `async`; runtime fallback is `sync`, while the shipped `.env.example` recommends `async`
- `ASYNC_ACCEPT_GRACE_MS`: in async mode, the short wait for immediate terminal rejections before returning `202`, default `1000`; increase it only if you prefer waiting longer to catch slower terminal rejections before returning `202`
- `RETRY_AFTER_DEVICE_UNAVAILABLE_MS`: short terminal cooldown after fresh `DeviceUnavailable` or `PrintLastTicket` responses, default `5000`
- `REQUEST_TIMEOUT_MS`: idle timeout while waiting for terminal pay-channel traffic or device-channel callbacks, default `90000`
- `REQUEST_MAX_DURATION_MS`: hard cap while waiting for a final OPI response, default `300000`
- `OPI_CONNECT_TIMEOUT_MS`: max time to establish the TCP connection to the terminal, default `3000`
- `TRANSACTION_PENDING_TIMEOUT_SECONDS`: pending-to-unknown timeout, default `180`
- `TRANSACTION_RECOVERY_GRACE_SECONDS`: startup grace for old pending transactions, default `30`

Recommended starting points:

- Raspberry Pi / terminal LAN deployments: keep `network_mode: host` in `docker-compose.yml`
- bridge networking only: use `PUBLISHED_HOST=127.0.0.1` for local-only exposure or `PUBLISHED_HOST=0.0.0.0` for LAN exposure
- easiest local first run: `TLS_MODE=auto`
- non-local traffic: `TLS_MODE=required`

## Storage And Response Mode

`STORAGE=json` keeps the legacy JSON and receipt files under `data/`. `STORAGE=db` uses SQLite and creates the database automatically. The current DB backend is SQLite only; static terminal config, auth config, terminal profiles, terminal models, and logs remain file based.

Runtime fallback stays at `RESPONSE_MODE=sync` for backward compatibility. The shipped `.env.example` recommends `RESPONSE_MODE=async` for new deployments. In sync mode, long-running terminal routes wait for the terminal result and return the final response. In async mode, the proxy returns `202 Accepted` only after it has written the OPI request to the configured terminal socket and the `ASYNC_ACCEPT_GRACE_MS` window has passed without an immediate terminal rejection. If the terminal is offline before that point, or if it answers with a final failed state such as `DeviceUnavailable` or `PrintLastTicket` during that window, the route returns an HTTP error instead of exposing a new `pending` operation. Increasing `ASYNC_ACCEPT_GRACE_MS` can catch slower terminal rejections before returning `202`, but also makes successful async starts wait longer. Fresh `DeviceUnavailable` and `PrintLastTicket` responses also put the terminal into a short retry window controlled by `RETRY_AFTER_DEVICE_UNAVAILABLE_MS`; new operations return `503` with `Retry-After` during that window instead of creating another pending transaction that is expected to fail. Payment-style routes include `Location: /transactions/{transactionId}` plus a minimal pending body. Service-style routes such as `/config`, `/init`, `/transmit`, `/close`, `/reprint`, `/abort`, and `/reset` return an empty `202` with `Preference-Applied: respond-async`. A request body `responseMode` value or `Prefer: respond-async` header can override the configured default for async-enabled routes.

Use async mode for integrations where HTTP client timeouts must not decide the terminal flow outcome. While running, `GET /transactions/:transactionId` returns `status=pending`. If the proxy never sent the request because the terminal could not be reached, it records `status=communication_error`. Cached reachability is informational only, so a terminal that comes back online can be retried without a forced activation just to clear the offline flag. If the request may have reached the terminal but the final outcome is not reliable, it records `status=unknown` with a recovery note so the integrator can reconcile operationally. If the terminal itself returns a final OPI `TimedOut` result with no card outcome details, the proxy records the operation as `failed`. If the proxy loses the final response but captured clear terminal display prompts for cancellation, card removal, or card-read failure, it records `aborted` or `failed` instead of leaving an unresolved transaction.

`REQUEST_TIMEOUT_MS` is an idle timeout while waiting for terminal activity. Pay-channel data and device-channel callbacks both refresh it, which prevents a long but still-active terminal interaction from being cut off. `REQUEST_MAX_DURATION_MS` is the hard maximum time to wait for a final OPI response even if callbacks continue. In sync mode, make sure upstream HTTP clients or reverse proxies allow enough time for the max duration when needed. Async mode avoids caller-side HTTP timeouts, but the background OPI operation still uses these timeouts to decide when the final result is no longer reliable.

The per-terminal active-operation lock expires after the largest of `ACTION_TIMEOUT_MS`, `REQUEST_TIMEOUT_MS`, and `REQUEST_MAX_DURATION_MS`, plus a 30 second safety grace. It is intentionally separate from `TRANSACTION_PENDING_TIMEOUT_SECONDS`: the lock protects an in-flight terminal socket operation, while pending/unknown transaction status protects financial reconciliation after the socket operation is no longer reliable.

Mutating routes also support the `Idempotency-Key` header. Reusing the same key with the same request method, route, async preference, and JSON body returns the same proxy response again. Reusing the same key with different request content returns HTTP `409`.

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

If `ports` are omitted from `terminals.json`, the proxy uses the model defaults, normally `payChannel0=4100` and `deviceChannel1=4102`. Multiple terminals may use the same default callback port; the proxy keeps one shared listener per host/port and routes device callbacks by `WorkstationID`. Keep `workstationId` unique for terminals that share a callback port. The listener accepts OPI device callbacks only from the configured `ip` of the terminal whose context is active on that port.

Service requests such as init/config do not use OPI printer status fields. Any service receipt or message sent back by the terminal is handled through the ECR/device callback channel.

## Payment Methods Filter

Request bodies can optionally include `paymentMethods` to restrict which explicitly filterable brands are allowed for processing.

This controls processing behavior only. It does not control which brands the terminal screen shows to the customer.

The example values reflect brands officially tested through the OPI Proxy as of today.

Other brands relevant to the Swiss market can still work through the underlying OPI integration even if they are not yet available for request-level filtering through `paymentMethods`.

Example:

```json
{
  "paymentMethods": ["visa", "mastercard", "maestro", "twint"]
}
```
