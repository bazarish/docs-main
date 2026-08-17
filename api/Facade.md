# Facade Contract

Status: draft skeleton.

The facade is the "last mile" between the client and the server infrastructure. It is deliberately a thin, replaceable layer: the client side of it may be anything (HTTP today, DNS later, exotic transports in principle), while the backend side speaks the standard internal API.

## First implementation: HTTP facade (implemented)

- **The facade's own I2P destination is the way in.** With `--i2p` (plus a
  `--state` directory for the keys and `--i2p-dests <n>` for how many) the facade
  publishes the same API on one or more encrypted-LeaseSet destinations, and the
  client prefers those over clearnet as soon as it has a transport of its own.
  The addresses are stable across restarts because they travel in the server
  descriptor beside the clearnet URLs. The I2P front parses no HTTP: an accepted
  stream is pumped to the facade's own listener, so routing, the secret prefix
  and streaming behave identically on both fronts. The stream is torn down only
  after the client has read the reply (or a bounded drain elapses): closing it
  straight after the last write truncates the reply, and the client then sees an
  empty stream rather than a response. The log says when each destination's
  leaseset has actually published ("i2p facade ready"), because until then a dial
  fails for reasons that have nothing to do with the facade.
- **Clearnet stays for the bootstrap.** A client with no netDb cannot use an I2P
  facade yet: it fetches the private reseed (`GET /v1/messaging/reseed`) over a
  clearnet facade, starts its embedded router with it, and only then switches to
  the I2P front. That request is pinned to clearnet in the client - routing it
  over I2P would ask the router to start before it had a netDb.
- **A profile that has reached its server over I2P refuses clearnet from then
  on** ("sticky I2P", persisted per profile). Falling back silently would bind
  the account's keys to the device's address at the server precisely when the
  network is being interfered with - the moment the user chose I2P for. The
  private reseed stays permitted, because it carries no identity and is what
  makes I2P reachable at all; everything else fails closed until the user allows
  clearnet again deliberately.
- **i2pd's built-in reseed hosts are off** in the client. They are allowed in
  exactly one case: this server's descriptor carries no clearnet facade, or none
  of them answered - the only situation with nobody to ask. Otherwise the
  bootstrap stays between the user and their own server.
- This is the **only** I2P address the server side publishes. Federation runs on
  the users' own destinations (`InviteAnonymity.md`), never a shared one, so a
  facade address says nothing about which users a server hosts.
- Published under a **secret URI** known only to clients of this server, and that secrecy is **mandatory, not a deployment option**: `bazarish-facade-http` refuses to start without `--base-path <secret>`, which it requires on every request and strips before forwarding. The single exception is an explicit `--public`, and it means one thing only - a fronting reverse proxy (Caddy, nginx, ...) already publishes this facade under a secret URI of its own. The secret reaches a client solely inside the facade URL of the server descriptor (below); it is never advertised, never derived from the server fingerprint, and never sent anywhere else.
- The whole clearnet API therefore lives under the secret prefix, **including the unauthenticated `GET /v1/messaging/reseed`** (`ClientApi.md`), whose only gate this is: a client fetching its first netDb has no I2P transport and no delivery token yet, so the prefix is what keeps the request - and the fact that this host serves Bazarish at all - out of a scan.
- A single catch-all handler relays every request **transparently** - method, headers, body and query string verbatim - to backends over the **operator private channel** (localhost/unix socket/VPN), routing by path prefix once the secret prefix is stripped:
  - `/v1/messaging/*` -> messaging server
  - `/v1/account/*` -> service node
  - anything else -> 404 (never forwarded)
- Everything outside the secret prefix answers the same bare **404** - the API root, an unknown path, and the operator liveness endpoint alike - so a probe of the host learns nothing. `/healthz` is not on the client-facing socket at all: it is served on a separate private **ops listener** (`--ops-listen <host> <port>`) that is never published to clients (`AdminPanel.md`).
- Nothing streams through the facade: the only large payload there ever was - a file upload - now goes directly between clients (see `FileTransfer.md`).
- Holds no keys, parses no content (the auth signature is verified by the backends, not the facade), stores nothing. An unreachable backend yields a `FACADE_UPSTREAM` 502 - the facade is a dumb pipe.
- Implemented in the `facade-http` repository (`Facade`; covered by tests and live-verified relaying to a running server-core).

## Client addressing

A client reaches a facade through a single **facade URL**:

```
http[s]://host[:port]/secret-path
```

- **scheme** - `http` or `https`. `https` is for facades fronted by a
  TLS-terminating reverse proxy; the client does **not** verify the TLS
  certificate (a self-signed or proxy cert is fine), because the facade is the
  untrusted last mile - all security is end-to-end and anchored in the server
  fingerprint, not in TLS PKI.
- **host[:port]** - the facade's address. The port defaults to `443` (https) /
  `80` (http) when omitted.
- **/secret-path** - the secret URI prefix, e.g. `/h9fd/kshdcy/secretdkg`. The
  client prepends it to **every** request path, the first reseed included; the
  facade (or the fronting proxy) strips it before forwarding. It is excluded from
  the signed canonical path, so the backend verifies the stripped, query-less
  path, and one signature is valid across all of a server's facades. A URL
  without a path is only ever correct when the fronting proxy holds the secret
  instead.

### Facade locality (invariant)

A facade URL belongs to **exactly one** server and is known **only** to that
server's own clients. This is a hard invariant, not a convention:

- A client only ever connects to **its own** server's facade. It never reaches
  another server's facade, and it does not store, learn, or carry a foreign
  facade URL at all.
- A facade URL therefore never appears in any artifact that crosses a server
  boundary - not in an invite, not in a contact request, not in an alias
  resolution. The unit of cross-server reachability is the recipient's serving
  **destination** (its b33), carried in the user-signed **contact card**
  (`Identity.md`, `InviteAnonymity.md`) - and **no facade**.
- A facade may be secret (a private secret-path, an unlisted host). The design
  must never force revealing one in order to reach a user on that server, so a
  server operator can keep its last mile confidential to its own clientele.

Consequences for first contact (`Contacts.md`, `Federation.md`):

- **Descriptor / QR** (`bazarish://invite?fp=...&srv=...&srv_key=...`, rendered as
  `fp:b33`) - carries the contact's fingerprint, serving destination and the
  server's serving sealing key. The client fetches the user-signed contact card
  for that fingerprint over I2P and verifies it against the fingerprint
  (`FederatedResolve.md`) - a direct fetch over a transient I2P destination
  preferred, the requester's **own** server as a proxy fallback, never the foreign
  facade. The contact
  request bootstrap carries the sender's own routing (prekey + destination +
  serving sealing key), so the reply path is equally facade-free.
- **By `<alias>`** - resolved at the **central resolver** over its I2P endpoint
  (`AliasResolver.md`); again direct-dial preferred, own-server proxy fallback,
  never the foreign facade.
- **By bare fingerprint** - a fingerprint is an identity, not a route, so it is
  always paired with a destination (the descriptor above). Cross-server first
  contact uses a descriptor or `<alias>`.

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
bazarish://server?v=1&fp=<server fingerprint>&facade=<facade URL>[&facade=<facade URL>...]
```

Every field is human-readable (no base64): the fingerprint is the trust anchor
(everything the server signs is verified against it as usual) and each `facade`
parameter is one facade URL, in client failover order. The service node renders
this exact string in the user's cabinet after sign-in, so a client reads back
precisely what the server published. Facade URLs ride verbatim, so they must not
contain `&`.

This is the **only** artifact that carries facade URLs, and it has a single,
narrow purpose: configuring **your own** server. Its audience is the prospective
clients of *this* server (an operator publishes it, e.g. on the server's site),
and it is consumed solely to populate the local facade list. It is never used to
reach another user - sharing it as if it were a contact would leak your server's
facades. To avoid that confusion the two shareable artifacts are kept distinct:

| Artifact | Carries | Audience | Facade? |
|---|---|---|---|
| `bazarish://server?...` | server fingerprint + facade URLs | prospective clients of **this** server | **yes** - own server only |
| `bazarish://invite?...` (descriptor) | fingerprint + serving destination + serving sealing key (`fp:b33` + `srv_key`) | anyone, to add **you** as a contact | no |

The descriptor carries no facade URL; the contact card it points at is fetched
over I2P and is user-signed, so it is verified against the fingerprint and
intermediaries are trusted for availability only (`Identity.md` /
`FederatedResolve.md` / `Federation.md`).

## Requirements

- **Compromise containment**: a compromised facade must yield nothing beyond the private-channel backend addresses and in-flight traffic (which is E2E-encrypted at the content level). It must never learn server root keys or storage contents.
- **No logging** of client-identifying data; access logs are disabled by design.
- Multiple facades per server are supported; each gets its own secret URI. Rotation/revocation of a secret URI must not require touching the server.

## Future facades

- **DNS facade** - low-bandwidth transport over the DNS protocol for aggressive network environments.
- The contract stays the same: any transport that can carry the request/response exchange to the backend API qualifies.

## Open questions

- Facade registration/rotation flow: how a server **operator** issues and revokes secret URIs. (Clients already learn a server's facades from the configuration link above, and can add/edit the list locally; what remains is the operator-side issuance/revocation and whether updated facades should also propagate via the server card.)
- Whether the facade enforces any rate limiting or leaves it entirely to the reverse proxy.
- Streaming/long-poll support requirements imposed by the incoming-delivery mechanism chosen in `ClientApi.md`.
