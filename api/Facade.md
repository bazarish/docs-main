# Facade Contract

Status: draft skeleton.

The facade is the "last mile" between the client and the server infrastructure. It is deliberately a thin, replaceable layer: the client side of it may be anything (HTTP today, DNS later, exotic transports in principle), while the backend side speaks the standard internal API.

## First implementation: HTTP facade (implemented)

- Deployed behind a reverse proxy (Caddy, nginx, …) under a **secret URI** known only to clients of this server. The secret URI is part of the registration bundle (server key fingerprint + facade list). The facade also accepts an optional `--base-path <secret>` that it requires and strips before forwarding, so the secret can live at the proxy, the facade, or both.
- A single catch-all handler relays every request **transparently** — method, headers, body and query string verbatim — to backends over the **operator private channel** (localhost/unix socket/VPN), routing by path prefix:
  - `/v1/messaging/*` → messaging server
  - `/v1/account/*` → service node
  - anything else → 404 (never forwarded)
- Holds no keys, parses no content (the auth signature is verified by the backends, not the facade), stores nothing at rest. An unreachable backend yields a `FACADE_UPSTREAM` 502 — the facade is a dumb pipe.
- Implemented in the `facade-http` repository (`Facade`; covered by tests and live-verified relaying to a running server-core).

## Requirements

- **Compromise containment**: a compromised facade must yield nothing beyond the private-channel backend addresses and in-flight traffic (which is E2E-encrypted at the content level). It must never learn server root keys or storage contents.
- **No logging** of client-identifying data; access logs are disabled by design.
- Multiple facades per server are supported; each gets its own secret URI. Rotation/revocation of a secret URI must not require touching the server.

## Future facades

- **DNS facade** — low-bandwidth transport over the DNS protocol for aggressive network environments.
- The contract stays the same: any transport that can carry the request/response exchange to the backend API qualifies.

## Open questions

- Facade registration/rotation flow: how a server operator issues and revokes secret URIs and how clients learn about new facades (presumably via an updated server card).
- Whether the facade enforces any rate limiting or leaves it entirely to the reverse proxy.
- Streaming/long-poll support requirements imposed by the incoming-delivery mechanism chosen in `ClientApi.md`.
