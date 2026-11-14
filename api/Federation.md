# Federation API (Client -> Serving Destination)

## Transport

- Delivery between servers is **I2P-only**.
  Each user's serving destination runs its
  own STREAM accept loop and is persisted so addresses are stable across restarts;
  a first-contact `op:card` is answered on those same destinations. The server's
  only published I2P addresses of its own are the facade's, for its clients
  (`Facade.md`). Outbound discovery fetches (a
  `card`/`resolve` lookup) are dialled by the client from a
  **warm pool of single-use transient destinations** (`WarmDestPool`): each lookup
  leaves from a fresh one-time dest that is **never reused** (so successive lookups
  are unlinkable, and none is tied to the server), and the pool
  keeps several destinations pre-built so that a lookup does not incur cold
  tunnel-build latency. The pool size is fixed in the client. With no warm
  destination available a fetch builds its own one-time dest and waits for it -
  slower, never a shared address. Delivery dials from a destination the sending
  client holds for that one correspondent, never from its **serving** dest; the
  confirmation returns in the reply on the same stream.
- Every routing/connection target is a full **`.b32.i2p`** address - a
  cryptographic destination address, never an addressbook name (poisonable) and
  never a raw base64 destination. This is enforced in code (`common`
  `isB32I2pHost`/`validateB32I2pHost`, applied wherever a destination is ingested
  or dialled). Every destination is published as an **encrypted LeaseSet2**, so
  the address is always the blinded **b33** (`<b33>.b32.i2p`) and the raw
  destination is not connectable (no plain leaseset is published). A user's own
  destination is published the same way: the delegation carries one blinded-key-authorized
  transient per day, which is what lets the server publish the address encrypted
  without ever holding the master signing key, and a delegation without those keys
  is refused (`Identity.md`, `InviteAnonymity.md`).
- The delivery framing operates on an abstract connected stream; nothing above
  the transport assumes I2P specifics.
- Wire format matches the rest of the system: JSON control messages + raw binary bodies.

### Delivery framing

Over the connected stream, one delivery is:

```
request:  {"op":"deliver","sealed":<base64>,"len":<N>}\n   then N raw payload bytes
stored:   {"delivered":true,"deliveryId":<id>,"signerPub":<base64>,"sig":<base64>}\n
refused:  {"delivered":false,"errorCode":<code>,"errorMessage":<text>}\n
```

Both ops arrive on the user's serving destination, served by three accept workers and one frame dispatcher; `deliver` runs the delivery pipeline (dedup -> admission -> size/quota -> store), `card` answers from the stored card. A stream carries one frame: its header line is at most 64 KiB, and the stream is closed 45 seconds after it opens. An unreadable frame, an unknown op or a payload claiming more than 2 MiB is answered `DELIVERY_REJECTED`.

The confirmation is carried **in that reply**: on storing the envelope the recipient signs the ASCII string `bazarish delivered v1` and a line feed followed by the `deliveryId` with the **serving destination's own per-destination signing key** (`signerPub` carries the public half; see *Authentication*), never with the server's root identity. The reply is the only available path, the dialling party being a client on an address that publishes no leaseset and accepts no inbound connections. As `deliveryId` is sealed inside the envelope, only a server that unsealed it can state it, and the signature binds that statement to the destination that answered. The sending client accepts a `delivered` result **only** when the signature verifies against the id it sent; an unsigned or incorrectly signed claim is treated as no answer (point 5 of *Message delivery flow*).

A destination is reachable once its leaseset is published. After a restart the netDb can still hold the leaseset of the previous process, whose tunnels are gone, until it expires: its last lease ends ten minutes after its newest tunnel was built. A delivery into such a leaseset gets no answer.

## Destination routing

Routing is **by destination**. The recipient's
serving destination (its b33) travels end to end in the recipient's contact card
(`../Identity.md`), so the sender **dials that destination directly** over I2P.
A destination is self-routing.

**Federation destinations are per user** (fundamental rule). Each
user is served by their own serving destination (never shared with another
user - see `InviteAnonymity.md`), and all *inbound* federation for that user is
served on it. Nothing dials out from it: an outgoing message is dialled by the
sender's own **client**, from a destination it holds for that one correspondent,
and discovery (the `card` request below) is dialled from a one-time transient. So
there is no address anywhere in federation that is attributable to a server, and
no server that knows whom its users write to.

**The identity of the dialling party is not part of admission.** Admission is
determined by the envelope alone: sealed to that destination's serving key, naming
a class, a mailbox and a delivery id, and carrying a pass unless it is a contact
request. No check distinguishes a server from a client, which is what allows a
client to deliver its own messages over the same frame.

Every message federates, including one between two users the same server hosts,
so delivery latency reveals no co-location (`InviteAnonymity.md`). The one write that does not
federate is a message to the sender's **own other devices**, which their server
stores directly (`ClientApi.md`) - there is no second party for it to reach.

**Through a gateway.** A gateway is the server's most trusted node, run for a
narrow circle of family or friends. A client on a gateway reaches it over TLS. The
gateway carries its users' dials to the server it is linked to over TLS
(`bazarish-server/gateway/PROTOCOL.md`, The server link), and connects two of its
own users directly, without I2P (Meeting on the gateway, same document). A dial to
a recipient that server hosts goes over that TLS link even when the recipient does
not use the gateway. The frame and the admission are the same as over I2P.

## Delivery envelope

Built entirely by the sending **client**; servers never see more than their layer:

```
DeliveryEnvelope:
  toDest: <recipient I2P destination>       # routing only - seen by the sending client alone
  sealed:                                   # hybrid seal to the serving sealing
                                            # key of that destination (from the contact card)
    class: content | contact                # delivery class (admission selector), not the
                                            # content type, which is E2E - see ../Messages.md
    mailbox: <recipient UID>
    pass: <delivery pass>                   # absent for class "contact"
    deliveryId: <the sender's name for this envelope>   # idempotency / dedup
  payload: <E2E blob>                       # opaque to every server-side component
```

- The sender's own server learns nothing of a delivery: the envelope does not
  pass through it. D is always the recipient's **own** destination: an
  offline-key master held by their client, delegated to the server as a
  time-boxed transient.
  A user whose transient has lapsed - or whose account a moderated server has not
  approved - has no destination and is not served; the server never
  silently re-points a live, already-shared address.
- The recipient's server learns only "something arrived from destination X into mailbox V with a valid pass" - never the sender identity (see `../Contacts.md`).

## Message delivery flow

1. The sending **client** seals the envelope and dials the recipient's serving destination over I2P, **from a destination held for that one correspondent** and never from its own serving destination. The term of that destination and its provenance are implementation matters; the requirement is that a single outbound address never carries traffic to two correspondents, and that a contact request leaves from an address that has carried nothing before it. The one exception is a **group host**, a bot account that relays one conversation to many members: it delivers `content` to all its members from a single outbound address, because fan-out speed is what limits a group. A server holding several members then sees one sender in their mailboxes and learns that they share a group, though not which group; that is the accepted price. A bot that is not a group host keeps one address per correspondent, and the contact-request rule holds for a group host too. No server of the sender's participates in a delivery, so none of them learns the sender's correspondents.
2. Client -> recipient's server: `deliver` with the envelope and its payload, on that one stream.
3. Recipient's server unseals and processes in strict order:
   1. **deliveryId dedup** - if the `deliveryId` is still pending in the mailbox, or is in the retained **seen-deliveryId set** (every admitted id, kept independently of the blob until the TTL sweep ages it out), respond success. This makes client retries idempotent and blocks replays of an already-delivered envelope, on both admission paths - and with a reusable pass it is the only thing that does;
   2. **admission** - for `content`, hash the presented pass and look it up in the mailbox's registered set (`../Contacts.md`); no match -> `DELIVERY_REJECTED`, and nothing is consumed. For `contact`, which presents nothing, the size cap below IS the admission;
   3. **size/quota checks** - explicit typed errors (`STORAGE_FULL`, `MESSAGE_TOO_LARGE`), revealed only past a valid pass; a contact request that does not fit the quota is answered `CONTACT_RATE_LIMITED`, the answer of the contact-request bounds below. The order is significant: size is checked only past admission, so a payload with an invalid pass gets the same answer on every destination, and nothing on this path tells whether a fingerprint is served here;
   4. **rate** - `content` is limited per pass, 120 a minute; `contact` per sending I2P destination, one a day (below). Both refuse with `CONTACT_RATE_LIMITED`. They come after dedup, so a retry of an envelope already stored is answered as stored, never refused;
   5. **store** as pending content, and add the `deliveryId` to the seen set.
4. Recipient's server -> sending client: the confirmation is returned **in the reply on the same stream**, signed over the stored `deliveryId` with the signing key of the destination that admitted it. No other path is available, the client having dialled from an address that accepts no inbound connections. The client accepts a `delivered` result only when that signature verifies against the delivery id it sent; an unsigned claim is treated as no answer.
5. A reply constitutes an answer, and a typed refusal ends the delivery. An unknown pass or an oversized payload is **final**: repeating the delivery does not change the outcome. A full mailbox (`STORAGE_FULL`) or a rate limit (`CONTACT_RATE_LIMITED`) is **transient**: the same delivery is accepted once the owner's devices have taken their mail in or the limit has passed, and the sender may repeat it later. An attempt without a reply does not end the delivery: the sending client keeps it, and no server holds it or retries it on another party's behalf. The next attempt to that destination is made only on a leaseset newer than the one the unanswered attempt used:
   - the sender drops what it holds of the recipient's leaseset and looks it up again right after the unanswered attempt, and again whenever something arrives from that contact; a leaseset published later than the one it held opens the way at once;
   - eleven minutes after the unanswered attempt - a lease's ten minutes and the 60 seconds by which two routers' clocks may differ - the leaseset that attempt used has expired everywhere, so any leaseset found then is a newer one;
   - a sender whose own link has come back, or whose device has moved to another network, tries every recipient again once its own server has answered it, since the silence may have been its own.

   A recipient that stays unreachable is therefore tried once every eleven minutes for as long as a delivery to it waits, and not at all when none does. A repeated delivery carries the same delivery id, so a recipient's server that stored the first copy recognises it and stores no second copy.

6. `DELIVERY_REJECTED` is the one refusal that says something about the **capability**: the presented pass is not in the mailbox's registered set. A correspondent holds one pass, so a sender that receives it can no longer deliver, and restoring delivery requires a new contact request. No refusal costs a sender anything: a pass is not spent, so `STORAGE_FULL` and `MESSAGE_TOO_LARGE` leave it exactly as it was, and so does this one.

`contact-request` deliveries skip the pass check, are capped in payload size (`kMaxContactRequestBytes`, a protocol constant measured against the real worst case - see `ClientApi.md`), and are rate-limited per receiving destination with a cap on how many may be waiting at once. They are also admitted **once per sending I2P destination**: the server keeps a hash of the address a request came from for 24 hours and refuses any further request from it. A client sends every request from an address of its own, so a second request from one address is not a client's behaviour, only an attack's. The list is the server's own and is held in memory. They are stored against the mailbox's one quota, as every class is.

## Requests

A serving destination answers two requests, one per stream, and refuses any
other with `DELIVERY_REJECTED`.

| Request | Purpose |
|---|---|
| `deliver` | Deliver an envelope (message or contact request) for a local mailbox |
| `card` | Return **this destination's own user's** contact card for a fingerprint query - the first-contact card fetch (`FederatedResolve.md`). A destination answers for the one user it belongs to and for nobody else, so no published address reveals whether another fingerprint is served there |

The `card` request and its answer:

```
request:  {"op":"card","sealed":<base64 of the query sealed under the card key>}\n
answer:   {"ok":true,"sealed":<base64 of the card frame sealed under the card key>}\n
refused:  {"ok":false,"errorCode":"CARD_UNKNOWN"}\n
```

`CARD_UNKNOWN` is the same answer for a fingerprint that is not this destination's
user and for a user with no card, so the refusal says nothing about who else is
served.

Alias resolution (`alias` -> descriptor) is served by the separate central
resolver over its own I2P endpoint
(`FederatedResolve.md`).

## Facade locality

A facade is the client<->own-server last mile and **never** crosses a server
boundary (see `Facade.md`). A client reaches another server only on its users'
serving destinations over I2P, and a facade only of its own server. So no
cross-server operation - delivery, contact request, or alias resolution - may
require a client to connect to, learn, or carry a foreign facade address. The unit
of cross-server reachability is the recipient's **serving destination** (its
b33), carried in the user-signed **contact card** (`../Identity.md`,
`InviteAnonymity.md`) and named by the invite's descriptor - never a facade and
never a server card.

This holds in both directions on first contact: a contact request carries the
sender's own **routing** (epoch keys + destination + serving sealing key) in
its bootstrap, so the recipient replies by federating to the sender's destination
over I2P, having learned only that destination - not a server fingerprint.

## First-contact discovery

First contact needs the peer's UID and contact card (epoch keys + devices +
destination + serving sealing key) without leaking to intermediaries and without the client
contacting a foreign facade. The model is specified in `FederatedResolve.md`; in
summary:

- A **descriptor** (`fp:b33`, shared as a QR or link) carries the fingerprint
  and route directly. The client fetches the user-signed contact card for that
  fingerprint over I2P - the `card` request above, with the fingerprint in the
  clear inside the stream - a direct fetch over a transient I2P destination, and
  verifies the card against the fingerprint; the serving sealing key arrives in
  the card. `fp` anchors integrity; nothing in between learns the fingerprint or
  the result, the I2P stream being encrypted to the destination.
- An **alias** resolves to a descriptor at the central resolver
  (`FederatedResolve.md`), which returns a signed self-verifying record over its I2P
  endpoint; the client then runs the card fetch above.

The client is built with the production resolver's coordinate
(`defaultResolverCoordinate`).

## Authentication

- Federation accountability rests on what a delivery presents: mail on its pass
  (120 a minute each), a contact request on the I2P destination it came from
  (one a day each). The server root key **never** signs live
  federation traffic - destinations are self-standing and the root only certifies
  offline; this keeps the connection origin from being a server-identifying
  signal.
- The only key that signs live federation traffic is the **per-destination
  signing key**: every serving destination (each personal
  dest) carries its own signing keypair, issued by the account **with the
  destination** and its serving sealing key and handed to the server with each
  delegation - a `(destination, signing key, serving sealing key)` group, never
  reused on another destination. It signs that
  destination's delivery confirmations and nothing else. The **server root identity** is
  a separate crypto-identity used only **outside** federation - the server card /
  `https://bazari.sh/#server` configuration link (`Facade.md`) and the service-node
  binding - and never signs a federated message. Consequently two users the same
  server hosts share no signing key and no fingerprint, so there is no
  server-wide signer to cluster them as co-located. Since no two users ever share
  a destination, no two users ever share a signer either.

## Limits

A serving destination admits mail at 120 a minute per
delivery pass, and contact requests at 3 a minute and at most 30 waiting, and one a
day per sending I2P destination; every rate limit, and a contact request that does
not fit the quota, refuses with `CONTACT_RATE_LIMITED`. A contact request is at most 33966 bytes
(`CONTACT_REQUEST_TOO_LARGE`) and a message at most 512 KiB (`MESSAGE_TOO_LARGE`).
Revoking a pass (`../Contacts.md`) stops an established contact.
