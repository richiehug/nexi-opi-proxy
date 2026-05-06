# Security And TLS

## Transport Policy

The proxy follows this runtime policy:

- localhost may use HTTP
- non-local traffic should use TLS
- insecure remote HTTP is only allowed if explicitly enabled

## Terminal OPI Peers

The proxy also checks the peer IP address on OPI terminal communication:

- pay-channel responses must come from the configured terminal `ip`
- device-channel callbacks are accepted only from the configured terminal `ip` for the active terminal context
- unexpected device-channel peers are rejected without an OPI response

This is a network-level guardrail in addition to bearer-token authentication on the public HTTP API. Keep `config/terminals.json` terminal IPs accurate, and avoid exposing the device callback port outside the trusted terminal network.

In Docker, the standard posture is:

- `HOST=0.0.0.0`
- `PUBLISHED_HOST=127.0.0.1` for local-only exposure
- `TLS_MODE=required` when the service is remotely reachable

## TLS Modes

- `auto`: HTTP on localhost, TLS required for non-local bind
- `required`: always use TLS
- `disabled`: HTTP only

Remote HTTP with `TLS_MODE=disabled` is blocked unless `ALLOW_INSECURE_REMOTE_HTTP=true` is set intentionally.

## Certificate Files

The bundle expects file-based certificates so Docker can mount them directly:

- `TLS_CERT_FILE`
- `TLS_KEY_FILE`

These paths are resolved from the bundle root and mounted into `/app/certs` inside the container.

## Development TLS

For a quick self-signed certificate during testing:

```bash
docker compose run --rm tls-init-dev
```

That creates:

- `certs/dev.crt`
- `certs/dev.key`

This is convenient for development, but clients will warn until the certificate is trusted locally.
