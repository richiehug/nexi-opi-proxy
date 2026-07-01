# Storage And Receipts

## Runtime Storage

The public Docker bundle keeps state outside the container image:

- `config/`
- `data/`
- `logs/`
- `certs/`

That means configuration, auth, transactions, receipts, and logs survive container restarts and image upgrades.

## Selectable Storage

The default is still:

```env
STORAGE=json
```

This keeps the legacy JSON/file persistence under `data/`.

SQLite can be enabled with:

```env
STORAGE=db
DB_URL=file:/app/data/opi-proxy.sqlite
```

The proxy enables WAL mode, creates tables on startup, and does not move terminal configuration, auth configuration, profiles, terminal models, or logs into the database.

Migration commands:

```bash
npm run storage:migrate -- --from=json --to=db
npm run storage:export -- --from=db --to=json
npm run storage:verify
```

Docker-style equivalents:

```bash
docker compose run --rm proxy node src/index.js storage migrate --from json --to db
docker compose run --rm proxy node src/index.js storage export --from db --to json
docker compose run --rm proxy node src/index.js storage verify
```

## Transaction Storage

Transactions are stored under `data/` and can be retrieved through:

- `GET /transactions/:transactionId`

SQLite transaction rows keep first-class columns for the common reporting fields: dates (`created_at`, `updated_at`), reference, amount, total amount, tip, currency, status, payment method, approval code, terminal alias, terminal model, device terminal id/TID, proxy transaction id, and terminal transaction reference number. The full transaction object is still preserved in `raw_json` for fields that are less common or terminal-specific.

The transaction record is the anchor object for follow-up actions such as:

- capture
- cancel
- refund
- receipt lookup

`status=pending` means the OPI request was handed to the terminal and the terminal operation has not fully ended. `status=communication_error` means the proxy could not reach the terminal before sending the OPI request. `status=failed` means the terminal returned a final failed result, including OPI `TimedOut` results that contain no approval or transaction reference. `status=unknown` means the proxy cannot safely determine the final payment or terminal result, usually after a restart, TCP interruption after request handoff, missing final OPI response, or a terminal timeout response that still contains card outcome details.

## Receipt Storage

Receipts are stored per transaction, not just as a loose "last receipt" text file.

That means each stored transaction can reference its own receipt bundle directly, and receipt lookups remain stable even after later transactions happen on the same terminal.

Transaction receipts are stored when the terminal sends receipt output back over the device callback channel. For card payments, use `receiptHandling.customer=external` and/or `receiptHandling.merchant=external` when the proxy should capture and store those copies under `data/receipts`. `local` tells a terminal with a printer, such as DX8000, to handle that copy locally instead.

`POST /terminals/:terminalAlias/reprint` uses the same callback path: the terminal sends the last receipt bundle back as OPI device output (`Printer`, `JournalPrinter`, and `E-Journal` when available), and the proxy stores those copies under the created reprint transaction. The proxy sends `TicketReprint` even for printerless terminals such as RX5000, IM15, and IM30, because the terminal may still replay the last receipt bundle over the callback channel. That means reprint is not just a terminal-side physical print action; it is also the proxy-visible recovery path for `PrintLastTicket` situations and for cases where a terminal completed a payment but the ECR or proxy lost the original final response.

When a reprint succeeds while a previous card transaction is still `pending` or `unknown`, the proxy inspects the reprinted receipt evidence. If the receipt clearly matches the previous sale payment, including successful receipt result, approval/reference numbers, matching amount/currency, and a transaction time close to the original request, the proxy automatically reconciles the previous transaction to `succeeded` and records the reprint evidence in the transaction response. If that evidence is missing or does not match, the previous transaction remains `unknown` and must be reconciled manually.

When external receipt handling is used with a terminal that has a built-in printer, the ECR owns the customer-facing receipt decision. The ECR should ask whether the customer receipt should be printed, shown, sent, or skipped, then use the stored receipt content from the proxy for the selected action.

For unattended IM15 and IM30 deployments, proxy-stored receipts are the default integration pattern because the terminals do not include printers. Merchants and integrators should retrieve stored receipts from the proxy and handle customer delivery outside the payment flow.

## Receipt Endpoints

For a specific transaction:

- `GET /transactions/:transactionId/receipts`

For the most recent receipt-producing transaction on a terminal:

- `GET /terminals/:terminalAlias/receipts/latest`

For the latest few receipt bundles on a terminal:

- `GET /terminals/:terminalAlias/receipts?limit=5`

## Receipt Roles

Stored receipts are labelled with a role:

- `customer`
- `merchant`
- `journal`
- `other`

The role is derived from the underlying OPI receipt target.
