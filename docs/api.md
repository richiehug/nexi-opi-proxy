# API Overview

The proxy exposes a pragmatic HTTP and JSON interface on top of OPI.

OpenAPI description:

- [`../openapi.yaml`](../openapi.yaml)

For this project, adding OpenAPI now is useful because the public HTTP surface is intentionally smaller than raw OPI, and integrators benefit from a single machine-readable contract for client generation, testing, and release validation.

## Terminal Routes

| Route | Purpose |
| --- | --- |
| `GET /health` | simple health probe for the proxy runtime |
| `GET /terminals/:terminalAlias/info` | reads terminal identity, software, network, acquirer, and card-circuit information |
| `GET /terminals/:terminalAlias/status` | reads live terminal status plus cached proxy state such as reachability and last status change |
| `GET /terminals/:terminalAlias/receipts/latest` | returns the latest stored receipt bundle for that terminal, from either a transaction or a service flow |
| `GET /terminals/:terminalAlias/receipts?limit=5` | lists recent stored receipt bundles for that terminal |
| `POST /terminals/:terminalAlias/payment` | starts a sale or pre-authorisation |
| `POST /terminals/:terminalAlias/refund` | starts a terminal-side refund |
| `POST /terminals/:terminalAlias/transmit` | triggers `TransmitTrx` to submit terminal-side transactions |
| `POST /terminals/:terminalAlias/config` | triggers `ContactTMS` to refresh the terminal OS configuration from ep2 |
| `POST /terminals/:terminalAlias/init` | triggers `ContactAcq` to refresh merchant and acquirer initialisation data from ep2 |
| `POST /terminals/:terminalAlias/reprint` | asks the terminal to reprint the last receipt |
| `POST /terminals/:terminalAlias/activate` | opens the terminal session |
| `POST /terminals/:terminalAlias/deactivate` | closes the terminal session |
| `POST /terminals/:terminalAlias/abort` | forwards `AbortRequest` to cancel an active terminal flow |
| `POST /terminals/:terminalAlias/close` | triggers `CloseDay` to perform final balance and logoff |
| `POST /terminals/:terminalAlias/reset` | restarts the terminal application |

`/reprint` remains available, even though it should still be treated as a feature that benefits from further production validation.

## Transaction Routes

- `GET /transactions/:transactionId`
- `GET /transactions/:transactionId/receipts`
- `POST /transactions/:transactionId/capture`
- `POST /transactions/:transactionId/cancel`
- `POST /transactions/:transactionId/refund`
- `PATCH /transactions/:transactionId`

The `PATCH` route is an admin recovery tool for manual transaction correction. It is not the same thing as terminal-side reconciliation.

## Payment Request Example

```json
{
  "reference": "order-100045",
  "amount": 24.90,
  "currency": "CHF",
  "paymentMethods": ["visa", "mastercard", "twint"],
  "receiptHandling": {
    "customer": "external",
    "merchant": "external"
  }
}
```

Important notes:

- `workstationId` is the OPI `WorkstationID`. Configure it as a stable, unique POS/ECR identity per checkout or terminal context. If omitted, the proxy uses the terminal alias.
- `ClerkId` and `ShiftNumber` are optional. If you do not send them, the proxy omits them from the OPI request.
- `paymentMethods` limits which brands can be processed. It does not change terminal UI branding.
- `receiptHandling.customer=external` and `receiptHandling.merchant=external` send receipt output to the proxy for storage under `data/receipts`. In this mode, receipts are stored by the proxy and the ECR is responsible for deciding whether the customer receipt should be printed, shown, sent, or skipped.
- `receiptHandling.customer=local` and `receiptHandling.merchant=local` let terminals with local printers handle those copies on the terminal.
- Interactive payment-style flows use the configured device callback port. Multiple terminals can share the same callback port when they have distinct `workstationId` values. If another interactive request is still active for the same workstation, the proxy responds with a `409` busy-style error.
- tips and DCC are not controlled through the public JSON request model.
- `POST /terminals/:terminalAlias/transmit` maps to OPI `TransmitTrx`.
- `POST /terminals/:terminalAlias/close` maps to OPI `CloseDay` and performs the final balance with logoff at the end.
- `POST /terminals/:terminalAlias/config` maps to OPI `ContactTMS`.
- `POST /terminals/:terminalAlias/init` maps to OPI `ContactAcq`.
- `POST /terminals/:terminalAlias/activate` and `/deactivate` normally return early when the proxy cache already says the terminal is active or inactive. Send `{ "force": true }` to bypass the cached state and send the OPI Login/LogOff request anyway. `AlreadyActivated` and `AlreadyDeactivated` responses are treated as successful state confirmations.
- `/config` and `/init` return the parsed terminal service payload, including status details and any supported brand data returned by the terminal.
- Service routes that print a slip, such as `/activate`, `/deactivate`, `/config`, `/init`, `/transmit`, and `/close`, store those receipts separately from transaction receipts under the terminal receipt storage.
- `/config`, `/init`, `/activate`, `/deactivate`, `/transmit`, and `/close` return receipt metadata and receipt file locations, not the full receipt body.
- `GET /terminals/:terminalAlias/receipts/latest` returns the latest stored receipt bundle for the terminal, whether it came from a transaction or from a terminal service operation.
- `GET /terminals/:terminalAlias/receipts?limit=5` lists the most recent stored receipt bundles for the terminal across both transaction and service flows.
- `POST /terminals/:terminalAlias/abort` forwards OPI `AbortRequest`. If the terminal no longer has an active flow to cancel, the terminal may answer with `Failure`.
