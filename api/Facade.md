# Facade Contract

Status: draft skeleton.

The facade is the "last mile" between the client and the server infrastructure. It is deliberately a thin, replaceable layer: the client side of it may be anything (HTTP today, DNS later, exotic transports in principle), while the backend side speaks the standard internal API.

## First implementation: HTTP facade (implemented)

- **The facade's own I2P destination is the way in.** With `i2p.enabled` (plus
  `state` for the keys and `i2p.destinations` for how many) the facade
  publishes the same API on one or more encrypted-LeaseSet destinations, and the
  client prefers those over clearnet as soon as it has a transport of its own.
  The addresses are stable across restarts because they travel in the server
  descriptor beside the clearnet URLs. The I2P front parses no HTTP: an accepted
  stream is pumped to the facade's own listener, so routing, the secret prefix
  and streaming behave identically on both fronts. The stream is torn down only
  after the client has read the reply (or a bounded drain elapses): closing it
  straight after the last write truncates the reply, and the client then sees an
  empty stream rather than a response. The log records when each destination's
  leaseset has actually published ("i2p facade ready"), because until then a dial
  fails for reasons that have nothing to do with the facade.
- **A facade is an I2P address, and the API is spoken over nothing else.** A
  clearnet facade is not tried at all, so there is no first connection in the
  clear either - which is when a client would otherwise show its address to the
  server it is about to register with. Where a stand on a LAN has no I2P at all,
  the client is started with `--allow-facade-without-i2p-for-dev-purposes` and
  the node's configuration key of the same name; both state their consequences, and
  both are disabled by default.
- **Clearnet is the bootstrap, and only the bootstrap.** A client with no netDb
  cannot dial anything on I2P, so the descriptor carries **reseed** addresses
  beside the facades: clearnet, fetched once, and never asked again once the
  router is up. They are a separate list because they are a different job; a
  reseed serves no API and a facade is never a reseed.
- **A reseed address is an https base URL, and the archive under it is
  `i2pseeds.su3`** - the shape every I2P reseed already has, so a router reads it
  with no special case anywhere. It is not a facade and speaks no Bazarish API:
  the client's I2P engine appends that name, fetches the archive, unpacks it and
  loads the routers, each of which carries its own signature. The archive's own
  signature is not required - a reseed an operator runs privately is signed by
  nobody a stock client trusts, so requiring it would mean no private reseeds -
  which is why the address must be **https**: the transport is what stands
  between the archive and the network.
- **A server may pack its own**, and its **portal** publishes it at
  `i2pseeds.su3` - never a facade. A facade is an I2P address and this is the one
  file a client needs before it has I2P; the portal is on the clearnet in any
  case. An operator who wants their own reseed names the portal's address (or
  whatever fronts it) in the descriptor.
- **The reseed need not be the server.** Because it is an ordinary su3 URL, an
  operator may publish it anywhere - a page they already serve, a host with no
  other relationship to the server - so where a user bootstraps says nothing
  about where their server is. A server that names reseed addresses this way
  needs no clearnet address of its own.
- **A third-party reseed host is a last resort.** With no reseed address in the
  descriptor the client's router falls back to the public hosts it carries - the
  only case in which the bootstrap leaves the network the user chose.
- This is the **only** I2P address the server side publishes. Federation runs on
  the users' own destinations (`InviteAnonymity.md`), never a shared one, so a
  facade address discloses nothing about which users a server hosts.
- Published under a **secret URI** known only to clients of this server, and that secrecy is **mandatory, not a deployment option**: `bazarish-facade-http` refuses to start without `basePath <secret>`, which it requires on every request and strips before forwarding. The single exception is an explicit `public`, and it means one thing only - a fronting reverse proxy (Caddy, nginx, ...) already publishes this facade under a secret URI of its own. The secret reaches a client solely inside the facade URL of the server descriptor (below); it is never advertised, never derived from the server fingerprint, and never sent anywhere else.
- The whole clearnet API therefore lives under the secret prefix. Nothing is left outside it: the reseed is no longer part of this API at all (`ClientApi.md`), so a facade answers only requests from a client that already holds the secret.
- **Two paths exist, and the facade reads neither.** Once the secret prefix is
  stripped, both are relayed over the **operator private channel**
  (localhost/unix socket/VPN) to the messaging server, which is the facade's only
  backend:
  - `POST /t` - the client tunnel. One opaque body in, one opaque body out; the
    answer is always `200`, because a status code outside the seal would be a
    channel of its own. What is inside - the method, the path, the headers, the
    device the request speaks for, the body, and the real status of the answer -
    is sealed and unreadable here. The facade cannot even tell mail being fetched
    from an account being managed. See `ClientApi.md`.
  - `GET /v1/server/card` - the signed server card, which is what a client seals
    its tunnel to. Public, identical for every caller, and signed by the
    fingerprint the user already had, so a facade may withhold it but cannot put
    one of its own in its place.
  - anything else -> 404 (never forwarded). The reseed archive is not among
    them: the server serves it on its ops socket and the portal serves it
    publicly (`ClientApi.md`), and the path the facade once whitelisted for it is
    gone from the list as well as from the answer.
- **Account management is not a separate route any more.** Client calls to the
  service node arrive inside the same tunnel and are forwarded to the node by the
  messaging server under its own signature; the node is no longer reachable from
  the facade at all (`Architecture.md`). That is what leaves one path where there
  used to be two prefixes, and with it goes the one bit that said which of the
  two kinds a request was.

### What the facade still knows

Sealing the traffic removes the **global identifiers** - the caller's public
keys (which are their identity), their destination, their device ids, the passes
that admit mail to their mailbox, the private transient that operates their
address. It does not remove the **flow**, and the difference has to be stated
rather than glossed over:

- a client dials from a stable outbound destination (a second one holds the long
  poll), so every request of one device is trivially linkable to the others;
- when a request happens, and when a parked poll returns - which is when mail
  arrived;
- how large it was, to the nearest step of the padding ladder.

"A facade knows the flow but not whose it is, nor what is in it" is what this
design achieves. "A facade knows nothing" would be untrue, and nothing here
should be read as claiming it.
- The ops listener also carries the facade's **own configuration**, operator-signed
  like `/healthz`: `GET /settings` and `PUT /settings` (the secret base path, the
  backend, both sockets, the I2P side, the log level) and `POST /restart`.
  They are on the ops socket for the same reason `/healthz` is - on the public one
  they would admit a facade is here. Only the log level applies at once; the rest
  is written into the facade's config file and read at its next start, which is
  what `POST /restart` is for. **Changing the base path invalidates every server
  descriptor already handed out**, so the new prefix has to reach clients (a new
  descriptor) before the old one stops answering.
- Everything outside the secret prefix answers the same bare **404** - the API root, an unknown path, and the operator liveness endpoint alike - so a probe of the host learns nothing. `/healthz` is not on the client-facing socket at all: it is served on a separate private **ops listener** (`opsListen`) that is never published to clients (`AdminPanel.md`).
- Nothing streams through the facade: the only large payload there ever was - a file upload - now goes directly between clients (see `FileTransfer.md`).
- Holds no keys, parses no content (there is nothing left to parse: authentication happens inside the seal, at the backend), stores nothing. An unreachable backend yields a `FACADE_UPSTREAM` 502 - the facade is a dumb pipe.
- Implemented in `facade-http/` of the `bazarish-server` repository (`Facade`; covered by tests and live-verified relaying to a running server-core).

## Client addressing

A client reaches a facade through a single **facade URL**:

```
http[s]://host[:port]/secret-path
```

- **scheme** - `http` or `https`. `https` is for facades fronted by a
  TLS-terminating reverse proxy; the client does **not** verify the TLS
  certificate (a self-signed or reverse-proxy certificate is acceptable), because the facade is the
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

### Connections

Over I2P a client dials its facade from **one destination per account**, shared
by everything that account sends: a destination carries many streams at once,
and every dial takes its own stream with its own receive queue. A second
destination would require a second set of tunnels and a second leaseset for the
same user, producing additional metadata to no benefit.

The connection is **kept between requests** (`Connection: keep-alive`; the
facade relays bytes and the backend's HTTP engine holds the connection open).
Measured against the stand, a request that opened its own stream took 1185 ms at
the median (n=149, quartiles 1067-1476); reusing the connection takes 506 ms
(n=39, quartiles 306-580). The dial itself measures 0 ms - I2P streaming carries
the first payload in the packet that opens the stream - so what a fresh stream
costs is its garlic session, the identity and signature in that first packet and
a fresh socket at the facade, not a round trip. A client that finds its kept
connection already closed redials and repeats the request once; request bodies
are replayable for exactly this reason.

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

- **Descriptor / QR** (`bazarish://invite?fp=...&dest=...&view=...`, rendered as
  `fp:b33`) - carries the contact's fingerprint, serving destination and the
  server's serving sealing key. The client fetches the user-signed contact card
  for that fingerprint over I2P and verifies it against the fingerprint
  (`FederatedResolve.md`) - a direct fetch over a transient I2P destination of
  its own, never through the requester's server and never through the foreign
  facade. The contact
  request bootstrap carries the sender's own routing (sealing key + destination +
  serving sealing key), so the reply path is equally facade-free.
- **By `<alias>`** - resolved at the **central resolver** over its I2P endpoint
  (`AliasResolver.md`); again the client's own direct dial, never through its
  server and never through the foreign facade.
- **By bare fingerprint** - a fingerprint is an identity, not a route, so it is
  always paired with a destination (the descriptor above). Cross-server first
  contact uses a descriptor or `<alias>`.

### Multiple facades and failover

A server may publish **several** facade URLs (different hosts, proxies, or
secret paths). The client holds them as an ordered list and **fails over** across
it: it sends to the current facade, and on an *unreachable* facade (transport
failure) advances to the next, cycling without end across attempts; an
HTTP error *response* is final (the facade is reachable and the request was
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

This is the **only** artifact that carries facade URLs, and it has a single
purpose: configuring the client's **own** server. Its audience is the prospective
clients of *that* server (an operator publishes it, for example on the server's
site), and it is consumed solely to populate the local facade list. It is never
used to reach another user; distributing it as though it were a contact would
disclose the server's facades. The two shareable artifacts are therefore kept
distinct:

| Artifact | Carries | Audience | Facade? |
|---|---|---|---|
| `bazarish://server?...` | server fingerprint + facade URLs | prospective clients of **this** server | **yes** - own server only |
| `bazarish://invite?...` (descriptor) | fingerprint + own destination + card-read capability (`fp:b32` + `view`) | anyone, to add the holder as a contact | no |

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
- ~~Streaming/long-poll support imposed by the incoming-delivery mechanism~~ -
  settled: incoming mail is one held-open request (`ClientApi.md`) and there is
  no second mechanism behind it, so a facade **must** carry a request answered
  late - up to 30 seconds - and its own upstream timeouts must outlast that. A
  facade that closes a parked request under a client is not slow, it is broken:
  the client has no other way to hear that mail arrived.
