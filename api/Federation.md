# Federation API (Server <-> Server)

Status: draft skeleton.

## Transport

- Delivery is **I2P-only**: the messaging server requires `--i2p` (there is no
  non-I2P transport; the earlier plain-TCP federation was removed). I2P is provided
  by an **embedded i2pd router** (no external SAM bridge). The server has no single
  federation destination - each user's serving destination runs its
  own STREAM accept loop and is persisted so addresses are stable across restarts;
  a first-contact `op:card` is answered on those same destinations. The server
  keeps **no destination of its own**: its only published I2P addresses are the
  facade's, for its clients (`Facade.md`). Outbound discovery fetches (a
  `card`/`resolve` lookup) dial from a
  **warm pool of single-use transient destinations** (`WarmDestPool`): each lookup
  leaves from a fresh one-time dest that is **never reused** (so successive lookups
  are unlinkable, and none is tied to the server), and the pool
  keeps a few dests pre-built so the lookup pays no cold tunnel-build latency. The
  pool size adapts from demand between a floor and `--warm-pool-max` (default 2..10;
  `--warm-pool-max 0` disables the pool, and a fetch then builds its own one-time
  dest and waits for it - slower, never a shared address. Delivery
  still dials from the sender's own **serving** dest (the recipient acks back to it),
  never a one-time dest.
- Every routing/connection target is a full **`.b32.i2p`** address - a
  cryptographic destination address, never an addressbook name (poisonable) and
  never a raw base64 destination. This is enforced in code (`common`
  `isB32I2pHost`/`validateB32I2pHost`, applied wherever a destination is ingested
  or dialled). Most destinations are published as an **encrypted LeaseSet2**, so
  the address is the blinded **b33** (`<b33>.b32.i2p`); the raw destination of
  such a dest is not connectable (no plain leaseset is published). A user's own
  offline-key destination can only publish a **standard LeaseSet2**, so it is
  addressed by its standard **b32** (`<b32>.b32.i2p` = base32(sha256(destination)))
  - offline keys cannot sign a b33 (see `Identity.md`, `InviteAnonymity.md`).
- The delivery framing operates on an abstract connected stream, so a future
  overlay transport would only need to supply a byte stream; nothing above the
  transport assumes I2P specifics.
- Wire format matches the rest of the system: JSON control messages + raw binary bodies.

### Delivery framing (implemented)

Over the connected stream, one delivery is:

```
request:  {"op":"deliver","sealed":<base64>,"len":<N>}\n   then N raw payload bytes
response: {"delivered":<bool>,"errorCode":...,"errorMessage":...}\n   (best-effort)
```

The recipient runs the same delivery pipeline as a local send (dedup -> token -> size/quota -> store), so local and federated deliveries share one code path.

The synchronous `response` is **best-effort**: its return leg over the recipient's *accepted* stream is unreliable (and on a single server it would require the sender's own destination to dial itself). The **authoritative** confirmation is a separate, reverse **`delivered-ack`** the recipient sends once it has stored the message:

```
delivered-ack:  {"op":"delivered-ack","messageId":<id>,"signerPub":<base64>,"sig":<base64>}\n
```

The recipient opens a **fresh outbound connection from the serving destination that received the delivery** back to the sender's serving destination, and signs `messageId` with that **serving destination's own per-destination signing key** (`signerPub` carries the public half; see *Authentication*) - never the server's root identity. The sender verifies the signature and that the `messageId` matches a pending send attempt - and since `messageId` is sealed inside the envelope, only a server that actually unsealed it can know it, so the ack doubles as a capability. On a verified ack the attempt is marked **delivered** (the yellow tick, `../Messages.md`) and the delivery job stops retrying. A `delivered-ack` gets no reply; the side that sent it keeps the connection open until the receiver closes, so the single line flushes before teardown.

Operational note: a freshly created destination must publish its leaseset before peers can reach it; the first connect after startup may return `CANT_REACH_PEER` ("LeaseSet not found") and should be retried with backoff. Verified end to end against a live i2pd (`i2p-federation-smoke`, and the full cross-server `demo-federation-i2p.sh`).

## Destination routing

Routing is **by destination**, not by server fingerprint. The recipient's
serving destination (its b33) travels end to end in the recipient's contact card
(`../Identity.md`), so the sender's server **dials that destination directly**
over I2P. There is no fingerprint->endpoint resolver and no `peers.json` - a
destination is self-routing.

**A server has no single, shared federation destination** (fundamental rule). Each
user is served by their own serving destination (never shared with another
k-anonymity set, or a personal per-user dest - see `InviteAnonymity.md`), and *all*
that user's federation flows over it: inbound delivery is served on it, and an
outbound delivery dials the recipient **from the sender's own serving destination**
- there is no server-wide address to dial from. Discovery (the `card` request
below) is answered on the users' own destinations and dialled from a one-time
transient, so nothing in federation is attributable to the server itself.

A delivery whose `toDest` equals the **sender's own** serving destination (the two
parties already share it) is stored locally; any other destination - including a
co-located recipient the same server hosts on a *different* serving dest - is
federated (the sender's serving dest dials the recipient's serving dest, two
distinct destinations) so delivery latency cannot reveal co-location
(`InviteAnonymity.md`).

For that shared-destination case the confirmation is also **synchronous and
local**: the server marks the attempt **delivered** the instant it stores the
message, with **no** federated `delivered-ack` (the yellow tick is immediate, no
I2P round-trip). An ack here would be both pointless - the two parties share the
destination by construction, so there is no co-location timing channel to mask -
and impossible, since it would require the destination to dial itself. The
federated `delivered-ack` below is therefore only for delivery to a *different*
destination.

## Delivery envelope

Built entirely by the sending **client**; servers never see more than their layer:

```
DeliveryEnvelope:
  toDest: <recipient I2P destination>       # routing only - all the sender's server sees
  sealed:                                   # CMS envelope, encrypted to the serving sealing
                                            # key of that destination (from the contact card)
    class: content | contact                # delivery class (admission selector), not the
                                            # content type, which is E2E - see ../Messages.md
    mailbox: <recipient UID>
    token: <one-time delivery token>        # absent for class "contact"
    messageId: <delivery id: the sender's name for this envelope>  # idempotency / dedup
  payload: <E2E blob>                       # opaque to every server-side component
```

- The sender's own server learns only "user U sent something to destination D"
  (plus size and time) - never the recipient identity. D is always the user's
  **own** destination: an offline-key master held by their client, delegated to
  the server as a time-boxed transient. There is no shared address to fall back
  on, because a shared one would make every user behind it provably co-located.
  A user whose transient has lapsed - or whose account a moderated server has not
  approved - simply has no destination and is not served; the server never
  silently re-points a live, already-shared address.
- The recipient's server learns only "something arrived from destination X into mailbox V with a valid token" - never the sender identity (see `../Contacts.md`).

## Message delivery flow

1. Sender's client -> own server: `send` with the envelope (the client also passes the `messageId` in the clear - the delivery id, see `../Messages.md` - so the server can correlate the recipient's later `delivered-ack` to this attempt). The server accepts it, **spools it on disk** and returns an attempt id at once; the client polls for the outcome (see `ClientApi.md`). The submit never blocks on federation, so the sender's "grey" tick is instant.
2. Sender's server -> recipient's server over I2P: `deliver` with the envelope, **dialled from the sender's own serving destination**.
3. Recipient's server unseals and processes in strict order:
   1. **messageId dedup** - if the `messageId` is still pending in the mailbox, or is in the retained **seen-messageId set** (every admitted id, kept independently of the blob until the TTL sweep ages it out), respond success *without* consuming a token. This makes client retries idempotent and blocks replays of an already-delivered envelope - including tokenless `contact` requests that have since been fetched, acked and swept;
   2. **token check** - hash the presented token, look it up in the mailbox's registered set (`../Contacts.md`); no match -> `DELIVERY_REJECTED`;
   3. **size/quota checks** - explicit typed errors (`QUOTA_EXCEEDED`, `STORAGE_FULL`), revealed only past a valid token;
   4. **store** as pending content; the token is consumed atomically with the store, and the `messageId` is added to the seen set.
4. Recipient's server -> sender's server: a signed **`delivered-ack`** for the stored `messageId` (see *Delivery framing*), dialled from the recipient's serving dest back to the sender's serving dest. The sender verifies it and marks the attempt **delivered**; the client sees this on its next poll (the yellow tick).
5. The outcome propagates to the sender's client via polling. A definite rejection (token/size) is a **failed** state; an unconfirmed delivery (no ack, recipient unreachable) **stays grey** rather than reporting a false failure - the envelope may have been stored while only its ack was lost - and the spooled copy is retried on the schedule in `ClientApi.md`. Once the last attempt is spent the envelope is dropped and the sender is told: their client shows that message as failed and sending it again is the user's decision. A resend produces the same delivery id, so a recipient's server that did store the first copy recognises it and spends no second token.

`contact-request` deliveries skip the token check, are capped in payload size (default 8192 bytes, server-configurable - large enough for the bootstrap prekey + a full reply-token batch), and are subject to a separate quota and per-origin-server rate limiting.

## Requests (draft)

| Request | Purpose |
|---|---|
| `deliver` | Deliver an envelope (message or contact request) for a local mailbox |
| `delivered-ack` | Signed confirmation, sent back to the sender's serving dest, that a delivered `messageId` was stored (drives the yellow tick) |
| `card` | Return a local user's contact card for a sealed fingerprint query - the first-contact card fetch (`FederatedResolve.md`) |

Username resolution (`alias` -> descriptor) is **not** an operator-to-operator
verb; it is served by the separate central resolver over its own I2P endpoint
(`AliasResolver.md`, `FederatedResolve.md`).

## Facade locality

A facade is the client<->own-server last mile and **never** crosses a server
boundary (see `Facade.md`). Servers reach each other only over the federation
transport (I2P), and a client only ever talks to its own server's facade. So no
cross-server operation - delivery, contact request, or alias resolution - may
require a client to connect to, learn, or carry a foreign facade URL. The unit
of cross-server reachability is the recipient's **serving destination** (its
b33), carried in the user-signed **contact card** (`../Identity.md`,
`InviteAnonymity.md`) - distributed inside an invite, never a facade and no
longer a root-signed server card.

This holds in both directions on first contact: a contact request carries the
sender's own **routing** (sealing prekey + destination + serving sealing key) in
its bootstrap, so the recipient replies by federating to the sender's destination
over I2P, having learned only that destination - not a server fingerprint.

## First-contact discovery

First contact needs the peer's UID and contact card (prekey + destination +
serving sealing key) without leaking to intermediaries and without the client
touching a foreign facade. The model is specified in `FederatedResolve.md`; in
summary:

- A **descriptor** (`fp:b33` plus the serving sealing key, shared as a QR or link)
  carries the fingerprint and route directly. The client fetches the user-signed
  contact card for that fingerprint over I2P - the `card` request above, with the
  fingerprint sealed to the serving sealing key - a direct fetch over a transient
  I2P destination preferred, own-server proxy fallback, and verifies the card
  against the fingerprint. `fp`
  anchors integrity; the relay learns neither the fingerprint (sealed) nor the
  result.
- A **username** resolves to a descriptor at the central resolver
  (`AliasResolver.md`), which returns a signed self-verifying record over its I2P
  endpoint; the client then runs the card fetch above.

**Implementation status.** The facade-locality and `.b32.i2p` address invariants
are enforced in code. The descriptor / card-fetch and the central-resolver
`resolve` flows (`FederatedResolve.md`) are **implemented and live-validated over
real I2P**; only the production resolver deployment (baking the resolver
coordinate into the client) remains.

## Authentication

- Federation accountability is **per-connecting-destination**: an inbound peer
  is rate-limited by the I2P destination it connects from (`FederationRateLimiter`,
  implemented). The server root key **never** signs live federation traffic -
  destinations are self-standing and the root only certifies offline; this keeps
  the connection origin from being a server-identifying signal.
- The only key that signs live federation traffic is the **per-destination
  signing key**: every serving destination (each personal
  dest) carries its own signing keypair, persisted **bound to the destination**
  together with its serving sealing key - a `(destination, signing key, serving
  sealing key)` group, never reused on another destination. It signs that
  destination's `delivered-ack`s and nothing else. The **server root identity** is
  a separate crypto-identity used only **outside** federation - the server card /
  `bazarish://server` configuration link (`Facade.md`) and the service-node
  binding - and never signs a federated message. Consequently two users the same
  server hosts share no signing key and no fingerprint, so there is no
  server-wide signer to cluster them as co-located. Since no two users ever share
  a destination, no two users ever share a signer either.

## Open questions

- **Per-server rate limiting and spam policy** between federation peers.
- **Delegated federation keys** vs signing everything with the root key.
