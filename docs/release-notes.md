# Release Notes

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
