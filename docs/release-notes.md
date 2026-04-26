# Release Notes

## 1.0.0

This is the first public Docker-first release line of Nexi OPI Proxy.

### Official Scope

Public `1.0.0` is officially positioned for:

- Ingenico RX5000
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
