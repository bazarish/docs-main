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
response: {"delivered":<bool>,"errorCode":...,"errorMessage":...,
           "deliveryId":<id>,"signerPub":<base64>,"sig":<base64>}\n
```

The recipient runs the same delivery pipeline whoever dialled (dedup -> token -> size/quota -> store), so a client's delivery and a discovery fetch share one served code path.

The confirmation **rides that reply**: on a store the recipient signs the `deliveryId` with the **serving destination's own per-destination signing key** (`signerPub` carries the public half; see *Authentication*) - never the server's root identity. It comes home this way because it can come home no other way: the dialler is a client on a one-time-ish address that publishes no leaseset, so nothing can dial it back. Since `deliveryId` is sealed inside the envelope, only a server that actually unsealed it can name it, and the signature ties that name to the destination that answered. The sending client believes a `delivered` **only** when the signature verifies against the id it sent; an unsigned or wrongly-signed claim is treated as no answer at all and the attempt is repeated.

Operational note: a freshly created destination must publish its leaseset before peers can reach it; the first connect after startup may return `CANT_REACH_PEER` ("LeaseSet not found") and should be retried with backoff. Verified end to end against a live i2pd (`i2p-federation-smoke`, and the full cross-server `demo-federation-i2p.sh`).

## Destination routing

Routing is **by destination**, not by server fingerprint. The recipient's
serving destination (its b33) travels end to end in the recipient's contact card
(`../Identity.md`), so the sender **dials that destination directly** over I2P.
There is no fingerprint->endpoint resolver and no `peers.json` - a destination is
self-routing.

**A server has no single, shared federation destination** (fundamental rule). Each
user is served by their own serving destination (never shared with another
k-anonymity set, or a personal per-user dest - see `InviteAnonymity.md`), and all
*inbound* federation for that user is served on it. Nothing dials out from it: an
outgoing message is dialled by the sender's own **client**, from a destination it
holds for that one correspondent, and discovery (the `card` request below) is
dialled from a one-time transient. So there is no address anywhere in federation
that is attributable to a server, and no server that knows whom its users write to.

**Who a recipient's destination accepts from is not part of the check.** Admission
is the envelope alone - sealed to that destination's serving key, naming a class, a
mailbox and a delivery id, and carrying a token unless it is a contact request.
Nothing asks whether the dialler is a server, which is exactly what lets a client
deliver its own mail without a protocol of its own.

Every message federates, including one between two users the same server hosts:
there is no local short-circuit and no shortcut for co-located parties, so delivery
latency reveals no co-location (`InviteAnonymity.md`). The one write that does not
federate is a message to the sender's **own other devices**, which their server
stores directly (`ClientApi.md`) - there is no second party for it to reach.

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
    deliveryId: <the sender's name for this envelope>   # idempotency / dedup
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

1. Sender's **client** seals the envelope and dials the recipient's serving destination itself, over I2P, **from a destination it holds for that one correspondent** and never from its own serving destination. How long it holds one and where it takes it from is the client's own affair; what the protocol asks is that one address never carries mail to two correspondents. No server of the sender's is involved in a send at all, so none of them learns whom the sender writes to.
2. Client -> recipient's server: `deliver` with the envelope and its payload, on that one stream.
3. Recipient's server unseals and processes in strict order:
   1. **deliveryId dedup** - if the `deliveryId` is still pending in the mailbox, or is in the retained **seen-deliveryId set** (every admitted id, kept independently of the blob until the TTL sweep ages it out), respond success *without* consuming a token. This makes client retries idempotent and blocks replays of an already-delivered envelope - including tokenless `contact` requests that have since been fetched, acked and swept;
   2. **token check** - hash the presented token, look it up in the mailbox's registered set (`../Contacts.md`); no match -> `DELIVERY_REJECTED`;
   3. **size/quota checks** - explicit typed errors (`QUOTA_EXCEEDED`, `STORAGE_FULL`), revealed only past a valid token;
   4. **store** as pending content; the token is consumed atomically with the store, and the `deliveryId` is added to the seen set.
4. Recipient's server -> sender's client: the confirmation rides **the reply on the same stream**, signed over the stored `deliveryId` with the signing key of the destination that took it. It can come home no other way - the client dialled from an address nobody can dial back - and the client believes a `delivered` only when that signature verifies (the yellow tick). An unsigned claim is treated as no answer at all.
5. A reply is an answer: a typed refusal (token, size, quota, contact rate limit) fails the message at once and repeating it changes nothing. Only a failure to *get* a reply is worth another attempt, and how many to make is the sending client's own policy - nothing is spooled anywhere and no server retries on anyone's behalf. Sending again produces the same delivery id, so a recipient's server that did store the first copy recognises it, stores nothing twice and spends no second token.

`contact-request` deliveries skip the token check, are capped in payload size (default 8192 bytes, server-configurable - large enough for the bootstrap prekey + a full reply-token batch), and are subject to a separate quota and per-origin-server rate limiting.

## Requests (draft)

| Request | Purpose |
|---|---|
| `deliver` | Deliver an envelope (message or contact request) for a local mailbox |
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
  I2P destination, and verifies the card against the fingerprint. `fp` anchors
  integrity; nothing in between learns the fingerprint (sealed) or the result.
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
  destination's delivery confirmations and nothing else. The **server root identity** is
  a separate crypto-identity used only **outside** federation - the server card /
  `bazarish://server` configuration link (`Facade.md`) and the service-node
  binding - and never signs a federated message. Consequently two users the same
  server hosts share no signing key and no fingerprint, so there is no
  server-wide signer to cluster them as co-located. Since no two users ever share
  a destination, no two users ever share a signer either.

## Open questions

- **Per-server rate limiting and spam policy** between federation peers.
- **Delegated federation keys** vs signing everything with the root key.
