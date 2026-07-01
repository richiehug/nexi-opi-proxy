# Release Notes

## 1.0.8

This patch release fixes proxy terminal-state tracking after daily close.

### Highlights

- `CloseDay` service responses now mark the cached terminal state as inactive, including async/background close flows.
- `CloseDay` responses with `DayClosed` are treated as successful close results.
- This prevents `/terminals/:terminalAlias/activate` from returning "already active" after the terminal has been logged off by daily close.

## 1.0.7

This release adds default configuration for EX8000 and A77 printerless mobile standalone terminals.

### Highlights

- added Ingenico EX8000 and PAX A77 to the default terminal model catalog
- added `ex8000-default` and `a77-default` terminal profiles
- added example terminal entries for EX8000 and A77
- EX8000 and A77 use external receipt handling by default, matching the current RX5000/Q30-style proxy behavior for printerless terminals

## 1.0.6

This release tightens Force Acceptance response handling.

### Highlights

- Payment responses now expose whether Force Acceptance was requested through `transaction.forceAcceptance`.
- Force Acceptance payment responses now expose `transaction.systemAvailable: true` when the terminal returns decline reason `0`, so merchants can switch back to normal online authorization after terminal is back online.
- Decline reasons `1`, `569`, and `579` now use public messages that explain authorization systems are unavailable.
- Terminal reversal requests can opt in to proxy-side confirmation with `autoConfirm: true`; when it is omitted or false, reversal confirmation remains terminal-side.

## 1.0.5

This release adds a minimal terminal reachability probe and improves timeout recovery for long-running card operations.

### Highlights

- added `GET /terminals/:terminalAlias/ping` for a lightweight ICMP reachability check against the configured terminal IP
- the ping route sends four packets and returns `204 No Content` when the terminal answers
- unreachable terminals return `503 Service Unavailable` with no response body
- Docker images now include the system `ping` utility required by the route
- `REQUEST_TIMEOUT_MS` now behaves as an idle timeout across both the pay channel and terminal device callbacks
- added `REQUEST_MAX_DURATION_MS`, default `300000`, as the hard cap for operations that keep producing callbacks but never return a final OPI response
- timeout cases with clear terminal display prompts such as cancellation, card removal, or card-read failure are stored as `aborted` or `failed` instead of `unknown`
- ambiguous lost-final-response timeouts attempt one automatic `TicketReprint` recovery before remaining `unknown`
- CCV/Q30 lost-final-response cases classify unsupported, declined, invalid, or card-read failure prompts as `failed`, while plain card-removal prompts alone no longer imply an abort
- CCV/Q30 final `TimedOut` responses with generic `OTHR` card circuit but no approval/reference/card evidence are stored as `failed` instead of `unknown`
- failed `TicketReprint` responses that clearly mean "no ticket/no receipt" clear the proxy's temporary terminal-unavailable cache instead of leaving the terminal blocked at the proxy level
- Q30 `AbortRequest` responses with `DeviceUnavailable` while a payment is still pending return `202` with the active transaction id and clear the temporary-unavailable cache, allowing clients to keep polling the original transaction
- synchronous transaction success/error responses keep transaction facts under `transaction`, include `transaction.terminalModel` when known, and avoid duplicating amount, currency, and status at the response top level
- SQLite transaction storage includes first-class columns for terminal model, tip, approval code, transaction reference number, and masked card number while preserving the full raw transaction JSON
- Payment failures with decline reason `1` are surfaced as `AUTHORIZATION_RESPONSE_UNKNOWN`; decline reasons `569` and `579` are surfaced as `PROVIDER_UNAVAILABLE`; force-acceptance payments that fail with terminal "function not supported" prompts are surfaced as `FORCE_ACCEPTANCE_UNSUPPORTED`. These use vendor-neutral public messages and nested `transaction.failure` details for force-acceptance aware clients
- Force acceptance is emitted as the OPI `POSdata` child element in card requests

### Integrator Notes

- Use `/ping` for network presence only; it does not open an OPI session or prove the terminal application is active.
- Use `/status` when you need the terminal's live OPI status payload.
- If a card transaction keeps producing terminal display/device callbacks, the proxy no longer cuts it off at `REQUEST_TIMEOUT_MS`; the idle timer is refreshed by that activity until `REQUEST_MAX_DURATION_MS`.

## 1.0.4

This release tightens terminal reachability handling and OPI peer validation.

### Highlights

- async terminal operations now return `202 Accepted` only after the OPI request has been written to the configured terminal socket
- async mode now waits briefly for immediate terminal rejections, so fast `DeviceUnavailable` and `PrintLastTicket` responses are returned as errors instead of new pending transactions
- fresh `DeviceUnavailable` and `PrintLastTicket` terminal responses now create a short `Retry-After` window, avoiding repeated pending transactions while the terminal is still not ready
- abort/control requests now open the OPI device callback listener, allowing receipt or display callbacks during abort handling to be acknowledged and captured
- terminal `TimedOut` results without approval or transaction reference details are stored as `failed`; ambiguous timeout responses with card outcome details remain `unknown`
- offline or black-holed terminal TCP connects now fail through `OPI_CONNECT_TIMEOUT_MS`, default `3000`
- if the terminal is offline before the request can be sent, routes now return a communication error instead of leaving a new operation as `pending`
- generic error responses now include only `error` and `requestId`; lock details are included only for active-lock conflicts
- unknown routes now return JSON `NOT_FOUND` responses instead of Express' default HTML page
- cached offline reachability no longer blocks later retries; the proxy attempts the terminal connection again and refreshes reachability from the result
- transactions that never reached the terminal are stored as `communication_error`; transactions that may have reached the terminal but lost the final response remain `unknown`
- device-channel OPI callbacks are accepted only from the configured terminal IP for the active terminal context
- pay-channel peers are validated against the configured terminal IP before processing responses

### Integrator Notes

- In async mode, `pending` now means the request was handed to the terminal and is still running.
- `communication_error` means the proxy could not reach the terminal before sending the OPI request, so there is no terminal-side transaction to reconcile.
- `unknown` remains the conservative status when the request may have reached the terminal but the final outcome is not reliable.
- `REQUEST_TIMEOUT_MS` still controls how long the proxy waits for a final OPI response after request handoff; sync clients and reverse proxies must allow enough time for cardholder interaction.
- Keep each terminal `ip` in `config/terminals.json` accurate, because the proxy now rejects OPI callbacks from other peer IPs.

## 1.0.3

This release adds selectable persistence and optional async handling for long-running terminal operations.

### Highlights

- added `STORAGE=json|db`, with JSON as the backwards-compatible default
- added SQLite for `STORAGE=db` using `DB_URL=file:/app/data/opi-proxy.sqlite`
- added `RESPONSE_MODE=sync|async`, with sync as the backwards-compatible default
- added request-level `responseMode` and `Prefer: respond-async`
- added durable terminal locks and stale pending recovery to `status=unknown`
- added JSON-to-DB migration, DB-to-JSON export, and storage verification commands
- added `Idempotency-Key` support on mutating routes for safe retries
- normalized 1.0.3 examples to uppercase `.env` variable names
- simplified public transaction lineage to `parentTransactionId`
- improved terminal-busy errors with the active operation name
- improved PAX `GetInfo` and `GetStatus` compatibility and status normalization

### Integrator Notes

- Payment async responses return HTTP `202`, `Location: /transactions/{transactionId}`, and a minimal pending body.
- Service-style async responses now return an empty HTTP `202` with `Preference-Applied: respond-async`.
- `GET /transactions/:transactionId` is the source of truth for current and final status.
- `status=unknown` means the proxy cannot safely determine the final terminal result and the transaction must be reconciled operationally.

## 1.0.2

This release expands the proxy with new Swiss ep2 transaction features and validated unattended terminal support.

### Supported Terminals

This release extends the proxy scope with validated unattended proxy-mode payment flows for:

- PAX IM15
- PAX IM30

For IM15 and IM30 deployments, the default configuration keeps payment flows activated. Receipts should be stored in the proxy and handled by the merchant outside the payment flow.

### Highlights

- added terminal-side `PaymentReversal` through `POST /terminals/:terminalAlias/reversal`
- added purchase-with-cashback support on `/payment` with `cashback`
- improved reversal and refund response mapping for clearer terminal outcomes
- validated unattended proxy-mode payment flows, including `DeliveryBox` acknowledgement and proxy-stored receipts

### Integrator Notes

- Cashback requests use `amount` for the total EFT amount and `cashback` for the cash withdrawal. OPI receives `TotalAmount = amount` plus `CashBackAmount = cashback`.
- Reversal uses OPI `PaymentReversal`. For Swiss ep2 the original approval code is not sent.
- The route can reuse the proxy's last successful transaction amount for the terminal; if no matching stored proxy transaction exists, the proxy can still forward the reversal request.
- Unattended support is positioned for OPI proxy usage with IM15 and IM30. In validated proxy-mode flows, the terminal sends `DeliveryBox`, the proxy answers success, and the terminal continues its own unattended flow.
- IM15 and IM30 receipts are stored by the proxy by default; merchants should handle receipt display, delivery, or retrieval outside the payment flow.

## 1.0.1

This release adds field-tested DX8000 support for ECR-enabled OPI integrations and improves receipt, abort, and decline handling.

### Highlights

- added DX8000 support for card payments in ECR/OPI mode
- handled DX8000 device callbacks for display messages, e-journal output, receipt output, and receipt confirmation prompts
- clarified `receiptHandling` behavior for terminal-local printing versus proxy-stored receipts
- fixed e-journal receipt capture when the terminal reports the target as `E-Journal`
- mapped clear terminal declines to transaction status `declined`
- marked pending transactions as `aborted` after a successful `AbortRequest`
- added per-step timestamps to OPI trace files

### Integrator Notes

- DX8000 terminals must be enabled for ECR integration before use with the proxy.
- The terminal callback port must match the configured `deviceChannel1` port.
- Use `receiptHandling.customer=external` and `receiptHandling.merchant=external` when receipts should be stored under `data/receipts`.
- Use `local` receipt handling when receipts should be printed by the terminal.
- When the DX8000 asks whether to print the cardholder receipt during local printing, the proxy answers `true` by default in this release.

## 1.0.0

This is the first public Docker-first release line of Nexi OPI Proxy.

### Official Scope

Public `1.0.0` is officially positioned for:

- Ingenico RX5000
- Ingenico DX8000
- PAX Q30

### Highlights

- Docker-only public runtime
- GHCR-ready image publishing plus offline image archive support
- clearer transaction-linked receipt storage
- new receipt retrieval endpoints
- terminal maintenance routes for transmit, close day, terminal configuration, and initialisation refresh
- cleaner request model for receipt and printer handling
- payment method filtering in the public JSON API
- `ClerkId` and `ShiftNumber` now truly optional
- improved public docs and bundled OpenAPI description

### Removed From Public Scope

- `/print`
- `/reconciliation`
- public DCC request control
- public tip control assumptions

### Supported But Still Worth Field Validation

- `/reprint`
- `/transmit`
- `/close`
- `/config`
- `/init`
