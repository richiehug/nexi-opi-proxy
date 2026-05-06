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
| `POST /terminals/:terminalAlias/reversal` | reverses the last terminal transaction through `PaymentReversal` |
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

## Sync And Async Responses

Runtime fallback remains `RESPONSE_MODE=sync` for backward compatibility, but the shipped `.env.example` recommends `RESPONSE_MODE=async` for new deployments.

`RESPONSE_MODE=async` returns HTTP `202` after the proxy has opened the terminal connection and written the OPI request to the configured terminal socket. If the terminal is offline or the proxy cannot write the request, the route returns an error instead of exposing a new `pending` operation. Payment-style operations include `Location: /transactions/{transactionId}` and:

```json
{
  "transactionId": "trx_...",
  "status": "pending",
  "terminalAlias": "t1"
}
```

Service-style operations such as `/config`, `/init`, `/transmit`, `/close`, `/reprint`, `/abort`, and `/reset` return HTTP `202` with `Preference-Applied: respond-async` and no response body.

Per request, send `"responseMode": "async"` in the JSON body or `Prefer: respond-async` to override the environment default. Accepted values are `sync` and `async`; invalid values return HTTP `400`.

Poll `GET /transactions/:transactionId` for the source-of-truth result. `pending` means the OPI request was handed to the terminal and the terminal operation is still running. `communication_error` means the proxy could not reach the terminal before sending the OPI request. `unknown` means the request may have reached the terminal, but the proxy cannot safely determine the final outcome and the integrator must reconcile operationally.

## Idempotency

Mutating routes support the `Idempotency-Key` header.

- reuse the same key with the same request method, route, async preference, and JSON body to receive the same proxy response again
- reusing the same key with different request content returns HTTP `409`
- the proxy currently keeps idempotency records for about 24 hours

This is useful for client retries after network interruptions or uncertain caller-side timeouts.

## Payment Request Example

```json
{
  "reference": "order-100045",
  "amount": 24.90,
  "currency": "CHF",
  "responseMode": "async",
  "paymentMethods": ["visa", "mastercard", "twint"],
  "receiptHandling": {
    "customer": "external",
    "merchant": "external"
  }
}
```

Cashback is requested on the normal payment route by sending the total EFT amount as `amount` and the cash withdrawal as `cashback`. The proxy sends OPI `TotalAmount` as `amount` and adds the `CashBackAmount` attribute. `cashback` must be less than or equal to `amount`.

```json
{
  "reference": "order-100046",
  "amount": 50.00,
  "cashback": 20.00,
  "currency": "CHF"
}
```

`POST /terminals/:terminalAlias/reversal` starts OPI `PaymentReversal` to reverse the last terminal transaction. On Swiss ep2 terminals the proxy does not send the original approval code. If the proxy has a previous successful transaction for that terminal, it reuses that transaction amount; otherwise it can still forward the reversal request without a stored proxy transaction.

Follow-up transactions created from another stored transaction include `parentTransactionId`. For example, a capture, cancellation, or refund created from an earlier pre-authorisation or sale points back to the original transaction with `parentTransactionId`. The public API does not expose a separate `rootTransactionId`; if no parent exists, the transaction stands on its own.

Capability overrides are best placed inside the terminal `capabilities` block in `config/terminals.json`. Example:

```json
{
  "rx": {
    "terminalModel": "RX5000",
    "terminalProfile": "rx5000-default",
    "ip": "192.168.0.60",
    "capabilities": {
      "supportsReversal": false,
      "supportsCashback": false
    }
  }
}
```

Resolution order is terminal `capabilities` override first, then terminal profile, then terminal model.

For unattended OPI proxy integrations, the proxy automatically acknowledges the basic `DeliveryBox` callback with success. On validated IM30 proxy flows, the terminal sends `DeliveryBox`, the proxy answers success, and the terminal then continues with its own delivery-success messaging such as `Product delivery successful`. Merchants should treat delivery, partial-delivery, and reversal decisions as application concerns outside the payment flow, or use pre-authorisation where that is the better fit. IM15 and IM30 terminals do not have printers, so use proxy-stored receipts and retrieve them through the receipt endpoints.

For IM15 and IM30 in the current proxy scope:

- card payments are validated in proxy mode
- reversals are available in proxy mode
- refunds may still be unavailable depending on the unattended terminal configuration, in which case the proxy returns `UNSUPPORTED_OPERATION`

Important notes:

- `workstationId` is the OPI `WorkstationID`. Configure it as a stable, unique POS/ECR identity per checkout or terminal context. If omitted, the proxy uses the terminal alias.
- `ClerkId` and `ShiftNumber` are optional. If you do not send them, the proxy omits them from the OPI request.
- `paymentMethods` limits which explicitly filterable brands can be processed for a request. It does not change terminal UI branding.
- the example `paymentMethods` values reflect brands officially tested through the OPI Proxy as of today
- other brands relevant to the Swiss market can still work through OPI even if they are not yet available for request-level filtering through `paymentMethods`
- `receiptHandling.customer=external` and `receiptHandling.merchant=external` send receipt output to the proxy for storage under `data/receipts`. In this mode, receipts are stored by the proxy and the ECR is responsible for deciding whether the customer receipt should be printed, shown, sent, or skipped.
- `receiptHandling.customer=local` and `receiptHandling.merchant=local` let terminals with local printers handle those copies on the terminal.
- Interactive payment-style flows use the configured device callback port. Multiple terminals can share the same callback port when they have distinct `workstationId` values. Device-channel callbacks are accepted only from the configured terminal IP for the active terminal context; other peer IPs are rejected without an OPI response. If another interactive request is still active for the same workstation, the proxy responds with HTTP `409` and a `BUSY` error plus the active operation details when available.
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
