# Release Notes

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
