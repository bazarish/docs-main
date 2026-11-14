# Invite Anonymity: destination-routed contact cards

This document covers how a contact is routed: by the user's own destination,
with per-destination serving keys, and with nothing server-wide in what a contact
receives. An invite is a small **descriptor** (`fp:b33`), and the contact card it
points to is fetched live (`FederatedResolve.md`).

## Problem

Anything server-wide in what a contact receives lets a third party who collects
invites from several users of one server **cluster** them: a root-signed server
card, a `server` fingerprint, transport endpoints or a server sealing key is
identical across every user of that server, and leaks co-location.

## Key idea

The contact card is **user-signed** and carries the routing itself. Routing is
**by destination**. The only fields a card carries are the user's own, plus a
destination.

```
ContactCard  (hybrid-signed by a device of user U - the only artifact a contact fetches):
  v
  t:           "contact-card"      # what this document is, checked first
  issuedAt:    <unix seconds>      # what orders two cards of one person
  kind:        user | bot | chat
  epochs:      [{n, from, g, by, key}]  # what U's mail is sealed to, this epoch and the next
  dest:        <i2p destination>   # dest_U - where to deliver to U
  servingKey:  <SPKI DER>          # key the delivery envelope is sealed to
  devices:     [device-cert]       # U's active devices, each certified by U's root
  revocation:  revocation-list     # U's newest list of revoked devices, signed by U's root
```

The owner is not written in the body: it is the root that certified the
devices, and the card's signer must be one of them. One fact, one place.

The card carries **only U's signatures** - one device's over the card and the
root's over each certificate and the list - and no server signature appears
anywhere on the contact path. `dest` and `servingKey` are plain fields
the user vouches for under the card's signature.

Why this is safe without a server signature: the first message is E2E-sealed to
U's current epoch key, whose private half only U's devices hold, so confidentiality does not depend
on the server. If `dest`/`servingKey` are tampered with, delivery fails (the
recipient server cannot produce a forged acceptance the sender's client would
believe, and no plaintext is exposed). Routing is a hint the user vouches for,
not a trust root.

### Two keys, one signature - why `servingSealingKey` exists

`servingSealingKey` is a **public encryption key embedded in the user-signed
card**. There are two sealed layers with **different readers**, so they need different
keys:

| Layer | Contents | Decrypted by | Key |
|---|---|---|---|
| Content (E2E) | the message itself | **U** (while offline, on next sync) | the current key in `epochs` |
| Admission header | `mailbox` + `pass` | **U's server** (while U is offline) | `servingSealingKey` |

The server stores-and-forwards while U is offline, so it **must** read
`mailbox` + `pass` to enforce contact-admission and quota (the pass scheme,
`../Contacts.md`) - while staying blind to the content layer, which it cannot
decrypt. One key cannot serve both layers because the readers differ (U vs U's
server). Sealing the admission header also keeps the I2P transport from learning
the recipient. The envelope does not pass through the **sender's own server**:
the client delivers it, so that server never learns the recipient
(`Federation.md`).

**Ownership**: the account issues the
serving key with its destination and hands its private half to the server that
operates it; the client publishes its public half in the card, which carries
only U's signatures.

## Routing by destination

`DeliveryEnvelope.toDest` is an I2P destination string. The sending client
connects **directly** to `toDest` over I2P. The envelope:

```
DeliveryEnvelope:
  toDest: <i2p destination>                 # routing only - and only the sending client sees it
  sealed:                                   # hybrid seal to the destination's serving sealing key
    class: content | contact
    mailbox: <recipient UID>
    pass: <delivery pass>                   # absent for class "contact"
    deliveryId: <delivery id>
  payload: <E2E blob>
```

The sender's own server learns nothing of the delivery: the envelope does not
pass through it, the sending client dialling D itself (`Federation.md`).

### Every message federates (co-location timing)

Every message is dialled over I2P by the sending **client** to the recipient's
serving destination, including where the same server hosts both parties. The only
local write is a message addressed to the sender's **own
other devices**, for which there is no second party and no destination to dial.

Rationale: every recipient is reached the same way, so **delivery latency** does
not indicate whether the two parties share a server. The client holds a
destination and dials it, and cannot determine which server operates it.

Through a gateway its user chose, dials take a shorter path. A gateway linked to
the server of the same operator carries its users' dials to that server's
destinations over TLS (`bazarish-server/gateway/PROTOCOL.md`, The server link),
and connects two of its own users directly, without I2P (Meeting on the gateway,
same document). Those users can tell from the latency which recipients that
server and that gateway host. The operator who runs both already knows it, and a
gateway serves a narrow circle of family or friends who chose it. A client on its
own router, or on a gateway somebody else runs, sees the same latency for every
recipient.

## The destination parameter (`dest_U`)

**Fundamental rule: a server has no single, shared federation destination.** Every
user is served by `dest_U` (their own per-user destination), at which their mail
arrives and on whose key their server signs for it (`Federation.md`). It initiates
no connections: outgoing mail is dialled by the sending client, from a destination
held for that one correspondent. A server keeps no address of its own - the
first-contact card fetch is answered on `dest_U` as well.

- Every user's `dest_U` is their own: an offline-key master held by the client,
  delegated to the server as a time-limited transient. No address is shared
  between users, so no two contact cards prove their owners are hosted together.

## Per-destination serving sealing key

The delivery envelope's routing metadata is sealed to a **serving sealing key
bound to `dest_U`**:

- Every dest is one user's own and has its own serving sealing key, so the key
  is shared with nobody the dest is not shared with and leaks nothing the dest
  did not already.

The destination server selects the key by **which destination the connection
arrived on** (each dest has its own accept loop). After unsealing, the server reads `mailbox` and stores
the delivery in it. The server holds the private halves of its
destinations' serving sealing keys, as each account handed them over; it issues
none itself (`../Identity.md`).

No serving sealing key is shared server-wide.

## Per-destination signing key

The same per-destination binding applies to the key that **signs** live
federation traffic (a delivery confirmation, `Federation.md`). Each serving
destination carries its own signing keypair, issued by the account with the
destination and its serving sealing key - a `(destination, signing key, serving
sealing key)` group, never reused on another destination:

- Every dest is one user's own and has its own signing key, so the signer is
  shared with nobody the dest and its serving sealing key are not shared with.

Two users share an ack key **iff** they share a destination, so the signer leaks
nothing the destination does not. The server **root identity** signs only
outside federation (the server card / `https://bazari.sh/#server` link, `Facade.md`),
never a federated message.

## Nothing peer-visible names the server

- **The card** above is what a contact fetches. It names no server and carries
  no expiry and no server signature; its `issuedAt` only orders two cards of one
  person.
- **Registration** opens an unbounded channel; the client authenticates every
  request by signature. The only clock is the **delegation** of the destination,
  which the I2P network itself enforces, and an account with none for the node's
  idle window is deleted.
- The card fetch is answered on the account's own destination and for that
  account alone, so a card cannot be requested by fingerprint alone and a server
  never confirms which users it hosts.
  The fetch is performed directly over I2P, so the requester's own server does
  not learn which identity is being added; being direct, it requires no
  application-layer sealing.

Routing that changes is repaired by the `routing` field carried in every message.

## How the user's card gets `dest_U` and the serving sealing key

The client makes both: the destination is its own offline key, and it issues the
serving sealing key and signing key with it. When it delegates the destination
(grant-gated; see `ServicePortal.md`) it hands the server the transient and the
private halves of both keys, and the server answers with the assigned `dest`
(brought up from the transient). The client folds the `dest` and the public
serving sealing key into its self-signed contact card. The server keeps the keys
it was handed and the dest<->mailbox mapping.

## Flows

- **createInvite** - emit the descriptor of the single user-signed contact card.
- **addByInvite** - fetch and verify the card; deliver to `card.dest`, seal the
  envelope to `card.servingSealingKey`.
- **contact-request bootstrap** (`../Contacts.md`) - the bootstrap carries the
  sender's routing (dest + serving sealing key + epoch keys), and its first
  letter the sending device's certificate, so the peer replies by federating to
  the sender's dest.
- **addByAlias / alias resolve** - the central resolver answers with a
  descriptor (`FederatedResolve.md`), and the client then fetches the contact
  card as for an invite. The lookup is not sealed; the client dials the
  resolver itself from a transient destination.
- **Local contact state** - `contact.dest` + `contact.servingSealingKey`.

## Federation

- The delivery envelope is addressed `toDest`, and the sending client dials
  `toDest` directly.
- **Every message is federated**, even between two users the same server
  hosts - see "Every message federates (co-location timing)" above.
- Accept side: every personal destination runs its own accept loop into the
  shared `DeliveryEngine`, with the serving sealing key selected by the
  destination the connection arrived on.
- The dest travels end to end, and routing is purely by destination.
- **Accountability** rests on the pass for mail (120 a minute each) and on the
  connecting destination for a contact request (one a day each). The server root key **never** signs live
  federation traffic; destinations are self-standing and the root only certifies
  offline. Each destination signs its own delivery confirmations with its own
  per-destination signing key (above), so the signer is never server-wide.

## Residual disclosure

- No two users ever share a `dest` or a serving sealing key, so a pair of
  contact cards can never prove their owners are hosted together.
- The home server knows dest<->U (operator trust).
- Global passive I2P traffic-correlation is out of scope.

Nothing a contact receives is shared across users of one server, and live
federation carries no server-wide signer (a delivery confirmation is signed per
destination).
