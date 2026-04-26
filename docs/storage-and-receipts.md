# Storage And Receipts

## Runtime Storage

The public Docker bundle keeps state outside the container image:

- `config/`
- `data/`
- `logs/`
- `certs/`

That means configuration, auth, transactions, receipts, and logs survive container restarts and image upgrades.

## Transaction Storage

Transactions are stored under `data/` and can be retrieved through:

- `GET /transactions/:transactionId`

The transaction record is the anchor object for follow-up actions such as:

- capture
- cancel
- refund
- receipt lookup

## Receipt Storage

Receipts are stored per transaction, not just as a loose "last receipt" text file.

That means each stored transaction can reference its own receipt bundle directly, and receipt lookups remain stable even after later transactions happen on the same terminal.

Transaction receipts are stored when the terminal sends receipt output back over the device callback channel. For card payments, use `receiptHandling.customer=external` and/or `receiptHandling.merchant=external` when the proxy should capture and store those copies under `data/receipts`. `local` tells a terminal with a printer, such as DX8000, to handle that copy locally instead.

When external receipt handling is used with a terminal that has a built-in printer, the ECR owns the customer-facing receipt decision. The ECR should ask whether the customer receipt should be printed, shown, sent, or skipped, then use the stored receipt content from the proxy for the selected action.

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
