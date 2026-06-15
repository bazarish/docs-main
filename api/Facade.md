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

## Client addressing

A client reaches a facade through a single **facade URL**:

```
http[s]://host[:port][/secret-path]
```

- **scheme** — `http` or `https`. `https` is for facades fronted by a
  TLS-terminating reverse proxy; the client does **not** verify the TLS
  certificate (a self-signed or proxy cert is fine), because the facade is the
  untrusted last mile — all security is end-to-end and anchored in the server
  fingerprint, not in TLS PKI.
- **host[:port]** — the facade's address. The port defaults to `443` (https) /
  `80` (http) when omitted.
- **/secret-path** — the optional secret URI prefix. The client prepends it to
  every request path; the facade (or the fronting proxy) strips it before
  forwarding. It is excluded from the signed canonical path, so the backend
  verifies the stripped, query-less path.

### Multiple facades and failover

A server may publish **several** facade URLs (different hosts, proxies, or
secret paths). The client holds them as an ordered list and **fails over** across
it: it sends to the current facade, and on an *unreachable* facade (transport
failure) advances to the next, cycling without end across attempts; an
HTTP error *response* is final (the facade is reachable, the request simply
failed) and does not trigger failover. The client remembers the last facade that
worked and starts there next time, so a healthy facade is preferred. Facades can
be added or edited at any time without touching the server.

### Configuration links

So a user can add a server with no manual entry, its whole configuration rides in
one link (also rendered as a QR):

```
bazarish://server/<base64url(JSON)>
```

where the JSON payload is `{ "v": 1, "fp": <server fingerprint>, "facades":
[<facade URL>, …] }`. The fingerprint is the trust anchor; everything the server
signs is verified against it as usual. (This is distinct from
`bazarish://invite/…`, which carries a self-verifying **contact** chain — a
subscription certificate plus a server card — see `Identity.md` / `Contacts.md`.)

## Requirements

- **Compromise containment**: a compromised facade must yield nothing beyond the private-channel backend addresses and in-flight traffic (which is E2E-encrypted at the content level). It must never learn server root keys or storage contents.
- **No logging** of client-identifying data; access logs are disabled by design.
- Multiple facades per server are supported; each gets its own secret URI. Rotation/revocation of a secret URI must not require touching the server.

## Future facades

- **DNS facade** — low-bandwidth transport over the DNS protocol for aggressive network environments.
- The contract stays the same: any transport that can carry the request/response exchange to the backend API qualifies.

## Open questions

- Facade registration/rotation flow: how a server **operator** issues and revokes secret URIs. (Clients already learn a server's facades from the configuration link above, and can add/edit the list locally; what remains is the operator-side issuance/revocation and whether updated facades should also propagate via the server card.)
- Whether the facade enforces any rate limiting or leaves it entirely to the reverse proxy.
- Streaming/long-poll support requirements imposed by the incoming-delivery mechanism chosen in `ClientApi.md`.
