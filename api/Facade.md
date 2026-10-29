# Facade Contract

The facade is the "last mile" between the client and the server infrastructure. It is deliberately a thin, replaceable layer: the client side is HTTP over the facade's own I2P destination, while the backend side speaks the standard internal API. Any transport that can carry the request/response exchange to the backend API would satisfy the same contract.

## HTTP facade

- **The facade's own I2P destination is the way in, and the only one.** A
  facade publishes its API on exactly one encrypted-LeaseSet destination, whose
  key it keeps in `state`; there is no switch that turns it off and no count of
  them, because a facade without it serves nobody and a client would not dial
  it (below). Several addresses for one server are several facades. The
  address is stable across restarts because it travels in the server
  descriptor. It carries 16 tunnels unless `i2p.tunnelQuantity` says
  otherwise. The I2P front parses no HTTP: an accepted stream is relayed to the
  facade's plain listener (`listen`, on loopback), which applies the routing. The
  stream is torn down only after the client has read the reply (or a bounded
  drain elapses): closing it immediately after the last write truncates the
  reply, and the client then sees an empty stream rather than a response. The log
  records when the destination's leaseset has published ("i2p facade ready"),
  because until then a dial fails for reasons that have nothing to do with the
  facade.
- **A facade is an I2P address, and the API is spoken over nothing else.** A
  clearnet facade is not tried at all, so there is no first connection in the
  clear either - which is when a client would otherwise show its address to the
  server it is about to register with. Where a stand on a LAN has no I2P at all,
  the client is started with `--allow-facade-without-i2p-for-dev-purposes` and
  the node's configuration key of the same name; both state their consequences, and
  both are disabled by default.
- **Clearnet is the bootstrap, and only the bootstrap.** A client with no netDb
  cannot dial anything on I2P, so the descriptor carries **reseed** addresses
  beside the facades: clearnet, fetched once, and never queried again once the
  router is up. They are a separate list because they serve a different
  purpose; a reseed serves no API and a facade is never a reseed.
- **A reseed address is an https base URL and nothing else, and the archive
  under it is `i2pseeds.su3`** - the form every I2P reseed already has, so a
  router reads it with no special case anywhere. It is not a facade and speaks no
  Bazarish API: the client's I2P engine appends that name, fetches the archive,
  unpacks it and loads the routers, each of which carries its own signature. The
  archive's own signature is not required - a reseed an operator runs privately
  is signed by nobody a stock client trusts, so requiring it would mean no
  private reseeds - which is why the address must be **https**: the transport is
  what stands between the archive and the network. The router refuses a reseed
  entry of any other form.
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
- **The address is the secret.** The destination is published as an encrypted
  LeaseSet (b33): the network database keeps it under a blinded key that changes
  daily, so the address is learned only from someone who holds it. It reaches a
  client solely inside the server descriptor (below); it is never advertised,
  never derived from the server fingerprint, and never sent anywhere else. The
  facade answers at `/`, with no path prefix of its own.
- **The facade knows its own address and says it.** It prints the address at
  start and reports it in `GET /settings` on its ops socket. The admin panel reads
  it from there and writes the facades' addresses into the server descriptor
  itself (`AdminPanel.md`), so an operator never copies an address by hand.
- **Two paths exist, and the facade reads neither.** Both are relayed over the
  **operator private channel** to the
  messaging server, which is the facade's only backend. That channel is TLS
  pinned by the key the messaging server prints, so it need not be a loopback
  address or a tunnel:
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
    them: the messaging server passes it to the portal over its private channel
    (`/internal/i2pseeds.su3`) and the portal serves it publicly
    (`ClientApi.md`); the facade has no path for it.
- **Account management has no route of its own.** Client calls to the
  service node arrive inside the same tunnel and are forwarded to the node by the
  messaging server under its own signature; the node is not reachable from
  the facade at all (`Architecture.md`). One path carries both kinds of request,
  so nothing outside the seal says which of the two a request is.
- Everything else answers the same bare **404** - an unknown path and the operator liveness endpoint alike. `/healthz` is not on the client-facing socket at all: it is served on a separate private **ops listener** (`opsListen`) that is never published to clients (`AdminPanel.md`).
- The ops listener also carries the facade's **own configuration**, operator-signed
  like `/healthz`: `GET /settings` and `PUT /settings` (the backend, the three
  sockets, the I2P side, the log level; `GET` adds the facade's address),
  `POST /new-address` and `POST /restart`. They are on the ops socket with
  `/healthz`: every client of the server holds the public address, and none of
  them needs the operator's endpoints. Only the log level applies at once; the rest is written into the
  facade's config file and read at its next start, which is what
  `POST /restart` is for.
- **`POST /new-address` replaces the key**, answers with the new address and the
  previous one, and restarts the facade to publish the new destination. **It
  invalidates every server descriptor that carries the old address**: a client keeps
  the facade list it was given and does not learn a new one by itself, so a
  client that knows only this facade loses the server until it is given the new
  descriptor. A server with several facades loses only the one.
- A third socket, `directListen`, is optional and serves **exactly what the I2P
  destination serves**, over TLS with the facade's own certificate. It exists
  for a gateway the same operator runs: given a route to it, that gateway dials
  this address instead of the destination and the flow does not enter I2P
  (`bazarish-server/gateway/PROTOCOL.md`). Left out, the destination is the only way
  in. It changes nothing a client can observe: the paths, the sealed tunnel and
  the uniform 404 are the same on both.
- Nothing streams through the facade: the only large payload there ever was - a file upload - now goes directly between clients (see `FileTransfer.md`).
- Holds no key of the server or of its users (only its own I2P destination and a self-signed TLS credential), parses no content (there is nothing left to parse: authentication happens inside the seal, at the backend), stores nothing. An unreachable backend yields an `UPSTREAM_UNAVAILABLE` 502 - the facade is only a relay.
- Implemented in `facade-http/` of the `bazarish-server` repository (`Facade`; covered by tests and live-verified relaying to a running server-core).

### What the facade still knows

Sealing the traffic removes the **global identifiers** - the caller's public
keys (which are their identity), their destination, their device ids, the passes
that admit mail to their mailbox, the private transient that operates their
address. It does not remove the **flow**, and the difference has to be stated
rather than glossed over:

- a client dials from one stable outbound destination per account, which also
  holds the long poll, so every request of one device is trivially linkable to
  the others;
- when a request happens, and when a parked poll returns - which is when mail
  arrived;
- how large it was, to the nearest step of the padding ladder.

"A facade knows the flow but not whose it is, nor what is in it" is what this
design achieves. "A facade knows nothing" would be untrue, and nothing here
should be read as claiming it.

## Client addressing

A client reaches a facade through its **facade address** - the I2P host, with
no scheme and no path:

```
<b33>.b32.i2p
```

It is the facade's own I2P destination, by the b33 of its encrypted LeaseSet2.
The client dials it over I2P and speaks plain HTTP on the stream, with paths
from `/`: the stream is encrypted to the destination dialled, and the facade is
the untrusted last mile in any case - all security is end-to-end and anchored in
the server fingerprint. A descriptor whose facade carries a scheme or a path is
refused.

A clearnet facade address (`host:port`, plain HTTP) exists only for a client
started with `--allow-facade-without-i2p-for-dev-purposes` (above); without it,
a descriptor that names one is refused. A client that hosts its own
server reaches that server's facade on its own loopback, which leaves the
machine for nothing.

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

A facade address belongs to **exactly one** server and is known **only** to that
server's own clients. This is a hard invariant, not a convention:

- A client only ever connects to **its own** server's facade. It never reaches
  another server's facade, and it does not store, learn, or carry a foreign
  facade address at all.
- A facade address therefore never appears in any artifact that crosses a server
  boundary - not in an invite, not in a contact request, not in an alias
  resolution. The unit of cross-server reachability is the recipient's serving
  **destination** (its b33), carried in the user-signed **contact card**
  (`Identity.md`, `InviteAnonymity.md`) - and **no facade**.
- A facade's address is known only to its own clients. The design
  must never force revealing one in order to reach a user on that server, so a
  server operator can keep its last mile confidential to its own clientele.

Consequences for first contact (`Contacts.md`, `Federation.md`):

- **Descriptor / QR** (`bazarish://invite?v=1&fp=...&dest=...`, `fp:b33` for
  short) - carries the contact's fingerprint and serving destination. The client fetches the user-signed contact card
  for that fingerprint over I2P and verifies it against the fingerprint
  (`FederatedResolve.md`) - a direct fetch over a transient I2P destination of
  its own, never through the requester's server and never through the foreign
  facade. The contact
  request bootstrap carries the sender's own routing (sealing key + destination +
  serving sealing key), so the reply path is equally facade-free.
- **By `<alias>`** - resolved at the **central resolver** over its I2P endpoint
  (`FederatedResolve.md`); again the client's own direct dial, never through its
  server and never through the foreign facade.
- **By bare fingerprint** - a fingerprint is an identity, not a route, so it is
  always paired with a destination (the descriptor above). Cross-server first
  contact uses a descriptor or `<alias>`.

### Multiple facades and failover

A server may publish **several** facade addresses, one per facade: a facade has
exactly one destination. The client holds them as an ordered list and **fails over** across
it: every request starts at the facade that answered last and tries the others in
list order, moving to the next only when one is *unreachable* (a transport
failure); an HTTP error *response* is final (the facade is reachable and the
request was failed) and does not trigger failover. When none answers, the request
fails. Until a facade has answered, the list's first is the one tried first. Facades can be added or edited at any time without touching the server.

### Configuration links

So that a user can add a server with no manual entry, its whole configuration is
carried in one link (also rendered as a QR):

```
bazarish://server?v=1&fp=<server fingerprint>&facade=<facade address>[&facade=<facade address>...][&reseed=<reseed address>...]
```

Every field is human-readable (no base64): the fingerprint is the trust anchor
(everything the server signs is verified against it as usual), each `facade`
parameter is one facade address, in client failover order, and each `reseed` one
reseed address (above). The service node renders this exact string in the user's
cabinet after sign-in, so a client reads back precisely what the server
published. Facade and reseed addresses are carried verbatim, so they must not
contain `&`.

This is the **only** artifact that carries facade addresses, and it has a single
purpose: configuring the client's **own** server. Its audience is the prospective
clients of *that* server (an operator publishes it, for example on the server's
site), and it is consumed solely to populate the local facade list. It is never
used to reach another user; distributing it as though it were a contact would
disclose the server's facades. The two shareable artifacts are therefore kept
distinct:

| Artifact | Carries | Audience | Facade? |
|---|---|---|---|
| `bazarish://server?...` | server fingerprint + facade addresses + reseed addresses | prospective clients of **this** server | **yes** - own server only |
| `bazarish://invite?...` (descriptor) | fingerprint + own destination (`fp:b33`) | anyone, to add the holder as a contact | no |

The descriptor carries no facade address; the contact card it points at is fetched
over I2P and is user-signed, so it is verified against the fingerprint and
intermediaries are trusted for availability only (`Identity.md` /
`FederatedResolve.md` / `Federation.md`).

## Requirements

- **Compromise containment**: a compromised facade must yield nothing beyond the private-channel backend addresses and in-flight traffic (which is E2E-encrypted at the content level). It must never learn server root keys or storage contents.
- **No logging** of client-identifying data; access logs are disabled by design.
- Multiple facades per server are supported; each has its own address. Replacing one must not require touching the server.

## Facade list, request rate and held requests

- **The facade list.** A facade mints its own address, and the admin panel
  publishes it into the server descriptor and replaces it on request
  (`AdminPanel.md`). A client learns its facade list only from a
  `bazarish://server` link: the server card names no facade, and nothing the
  server sends replaces the list a client holds. A changed list reaches an
  existing client when its user enters the new link in the client's
  server-connection settings; the portal shows the current link after sign-in.
- **No request-rate limit.** The tunnel is sealed, so the facade cannot tell one
  user's requests from another's. Its I2P front relays at most 256 streams at
  once and closes any further stream unanswered.
- **Held requests.** Incoming mail is one held-open request (`ClientApi.md`),
  which the server holds for at most 30 seconds, and there is no second mechanism
  behind it. The facade waits 60 seconds for its backend and keeps a client's
  connection for 90, so it never cuts a parked request short. A facade that
  closes a client's parked request is not slow, it is broken: the client has no
  other way to hear that mail arrived.
