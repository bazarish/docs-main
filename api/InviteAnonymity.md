# Invite Anonymity: destination-routed contact cards

Status: **implemented and live-validated** over real I2P (single-server +
cross-server `demo-federation-i2p.sh`, 2026-06-17). Supersedes the old
invite/server-card shape; `../Identity.md`, `../Contacts.md`, `Facade.md` and
`Federation.md` have been updated to match.

> **Note (later supersession).** The destination-routing, per-destination serving
> sealing keys, and route-by-b33 model below still hold. The **invite artifact**
> itself - the full ~= 8 KB user-signed contact card carried inline - has since
> been replaced by a small **descriptor** (`fp:b33` + serving sealing key) plus a
> live card fetch (`FederatedResolve.md`). Read this document
> for the destination-routing rationale; read `FederatedResolve.md` for the
> current first-contact/discovery shape.

## Problem

An invite today carries two artifacts: the user's **subscription certificate**
(user-signed: `user`, `server`, sealing key) and the serving server's **server card**
(root-signed: `server`, transport endpoints, sealing key). A third party who
collects invites from several users of one server can **cluster** them: the
server card is a byte-identical, root-signed blob shared by every user of that
server, and even setting the blob aside, the `server` fingerprint, the endpoints
and the server sealing key are identical field *values* across those users. So
the invite leaks co-location.

The clustering is structural, not cosmetic: routing is **by server
fingerprint** (`DeliveryEnvelope.toServer`; the sender's own server resolves the
fingerprint to an I2P destination), and the envelope is sealed to **one shared
server sealing key**. Both are server-wide, so both are shared.

## Key lever

The subscription certificate is **user-signed**. Fold the routing into it and
drop the root-signed server card entirely. Route **by destination**, not by
server fingerprint. Then the only fields a card carries are the user's own, plus
a destination - and the destination is the single tunable anonymity knob.

```
ContactCard  (hybrid-signed by user U - the only artifact a contact fetches):
  v
  t:           "contact-card"      # what this document is, checked first
  issuedAt:    <unix seconds>      # what orders two cards of one person
  sealingKey:  <SPKI DER>          # what U's mail is sealed to, first and after
  dest:        <i2p destination>   # dest_U - where to deliver to U
  servingKey:  <SPKI DER>          # key the delivery envelope is sealed to
```

The owner is not written in the body: it is the signer, and `fingerprint()` is
taken from the keys that signed. One fact, one place.

No `server` fingerprint, no endpoints list, no separate root-signed card, no
root signature. The card carries **exactly one signature - U's**; there is no
server signature anywhere on the contact path. `dest` and `servingKey`
are plain fields the user vouches for under that one signature.

Why this is safe with no server signature: the first message is E2E-sealed to
`sealingKey`, whose private half only U holds - confidentiality never depended on the server. If
`dest`/`servingKey` were tampered, delivery merely fails (the recipient
server cannot produce a forged acceptance the sender's client would believe, and
no plaintext is exposed). The server was already untrusted for content; routing
is now an unsigned hint the user vouches for, not a trust root.

### Two keys, one signature - why `servingSealingKey` exists

`servingSealingKey` (the `sealingKey_U` of the decided
`sign_U{sealingKey, dest_U}` shape) is a **public encryption key
embedded in the user-signed card** - not a signature and not a trust anchor.
There are two sealed layers with **different readers**, so they need different
keys:

| Layer | Contents | Decrypted by | Key |
|---|---|---|---|
| Content (E2E) | the message itself | **U** (while offline, on next sync) | `sealingKey` |
| Admission header | `mailbox` + `pass` | **U's server** (while U is offline) | `servingSealingKey` |

The server stores-and-forwards while U is offline, so it **must** read
`mailbox` + `pass` to enforce contact-admission and quota (the pass scheme,
`../Contacts.md`) - while staying blind to the content layer, which it cannot
decrypt. One key cannot serve both layers because the readers differ (U vs U's
server). Sealing the admission header is also what keeps the **sender's own
relay server** (and the I2P transport) from learning the recipient. The envelope
no longer passes through that server at all - the client delivers it - and the
sealed header is what kept the "sender's own server never learns the recipient"
invariant (`Federation.md`) even when it did;
it is what keeps the serving key bound to the destination rather than the server.

**Ownership** (so this is not a "server signs" step): for a personal dest
the client may **generate `sealingKey_U` itself and delegate the private half to
its server** (like the offline destination key), making it fully
client-originated. Either way the card still
carries only U's signature.

## Routing by destination

`DeliveryEnvelope.toServer` (a server fingerprint) becomes `toDest` (an I2P
destination string). The sending client connects **directly** to `toDest` over I2P
- no fingerprint->`peers.json` resolution, no server card needed to route. The rest
of the envelope is unchanged:

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

The sender's own server still learns only "user U sent something to destination
D" - which, since D is the recipient's own destination, is as strong as a
server fingerprint, not stronger.

### No local delivery (no co-location timing channel)

No short-circuit exists. Every message is dialled over I2P by the sending
**client** to the recipient's serving destination, including where the same server
hosts both parties: the server is not asked to deliver and therefore has nothing to
short-circuit. The only local write is a message addressed to the sender's **own
other devices**, for which there is no second party and no destination to dial.

Rationale: if a co-located recipient were served locally and a remote one over
I2P, the difference in **delivery latency** would indicate whether the two parties
share a server - a timing side channel that reintroduces the co-location signal
destination routing removes. With delivery performed by the client the question
does not arise: the client holds a destination and dials it, and cannot determine
which server operates it.

## The destination knob (`dest_U`)

**Fundamental rule: a server has no single, shared federation destination.** Every
user is served by `dest_U` (their own per-user destination), at which their mail
arrives and on whose key their server signs for it (`Federation.md`). It initiates
no connections: outgoing mail is dialled by the sending client, from a destination
held for that one correspondent. A server-wide "home" address is never used for
user traffic (it serves first-contact card discovery only); a shared federation
source would both expose a common address and, where sender and recipient share a
server, cause one user's destination to dial another's.

- Every user's `dest_U` is their own: an offline-key master held by the client,
  delegated to the server as a time-boxed transient. There is deliberately no
  shared fallback - a shared address would let anyone holding two contact cards
  prove their owners are hosted together, which is the metadata the
  project treats as a critical bug.

Per-user destinations already exist in `server-core`; this revision is what
finally **surfaces** the assigned destination into the user-facing card.

## Per-destination serving sealing key

The delivery envelope's routing metadata is sealed to a **serving sealing key
bound to `dest_U`**, not to one server-wide key:

- A **personal** dest has its own sealing key (effectively per-user).
  the same k-anonymity set as the dest itself, so it leaks nothing the dest did
  not already.

The destination server selects the key by **which destination the connection
arrived on** (each dest has its own accept loop already), so there is no
trial-decryption. After unsealing, the server reads `mailbox` and delivers
locally exactly as today. The server holds the private halves of its
destinations' serving sealing keys.

This replaces the single server sealing key carried in the server card; the
value is no longer shared server-wide.

## Per-destination signing key

The same per-destination binding applies to the key that **signs** live
federation traffic (a delivery confirmation, `Federation.md`). Each serving
destination carries its own signing keypair, stored bound to the destination
alongside its serving sealing key - a `(destination, signing key, serving sealing
key)` group, never reused on another destination:

- A **personal** dest has its own signing key (effectively per-user).
  same k-anonymity set as the dest and its serving sealing key.

This closes the last server-wide signer. A single server-identity ack key would
let two users the same server hosts on **different** destinations observe the same
signature key and cluster as co-located - re-introducing exactly the shared-server
fingerprint this model removes. Binding the signer to the destination means two
users share an ack key **iff** they already share a destination (and thus a
k-anonymity set), so it leaks nothing new. The server **root identity** signs only
outside federation (the server card / `bazarish://server` link, `Facade.md`),
never a federated message.

## Done: there is no subscription certificate

Implemented 2026-08-23. The old subscription certificate served two audiences
with one signed object - the node's lifecycle anchor and the peer-facing routing
statement - and told both which server serves the user. Both halves are gone:

- **The card** above is what a contact fetches. It has no `server`, no
  `issuedAt`/`notAfter`, and no server signature. Nothing peer-visible names the
  server.
- **The subscription is gone entirely.** Registration opens an unbounded channel;
  the client authenticates every request by signature anyway, so a signed "I
  subscribe here" told the node nothing it had not already verified. The only
  clock is the **delegation** of the destination, which the I2P network itself
  enforces, and an account with none for the node's idle window is deleted.
- The card fetch is answered on the account's own destination and for that
  account alone, so a card cannot be requested by fingerprint alone and a server
  never confirms which users it hosts.
  The fetch is performed directly over I2P, since relaying it would disclose to the
  requester's own server which identity is being added; being direct, it requires
  no application-layer sealing.

The card carries no expiry either: routing that has changed is repaired by the
`routing` field carried in every message.

## How the user learns `dest_U` and the serving sealing key

At subscribe time (grant-gated; see `ServicePortal.md`) the server returns, in
place of the server card:

- the assigned `dest` (brought up from the client's delegated offline
  transient), and
- the **public** serving sealing key for that dest.

The client folds both into its self-signed contact card and into the cert it
stores. The server keeps the private serving sealing key and the dest<->mailbox
mapping.

## Flow migration

- **createInvite** - emit the single user-signed contact card; no server card.
- **addByInvite** - verify the card; deliver to `card.dest`, seal the envelope
  to `card.servingSealingKey`. No server card, no fingerprint resolve.
- **contact-request bootstrap** (`../Contacts.md`) - the bootstrap ships the
  sender's **contact card** (dest + serving sealing key + sealing key) instead of the
  sender's server card, so the peer replies by federating to the sender's dest.
- **addByAlias / alias resolve** - the resolved trust chain becomes the
  contact card (not sub-cert + server card). The cross-server sealed `resolve`
  (`Federation.md`, still pending - the Stage-2 track) keeps a **server-level**
  resolve key for its entry point: the client cannot seal to a per-dest key for
  a user whose dest it does not yet know. That server-level resolve key is the
  one residual server-scoped artifact, used only to bootstrap alias lookups, and
  is out of scope for the invite revision.
- **Local contact state** - store `contact.dest` + `contact.servingSealingKey`
  in place of `contact.server` + server-card sealing key.

## Federation impact

- `toServer` -> `toDest`; the sending client dials `toDest` directly.
- **No local short-circuit**: every message is federated, even between two users
  the same server hosts - see "No local delivery at all" above. This is the rule,
  not an optimisation; a fast local path for a co-located recipient is a timing
  leak.
- Accept side is unchanged in shape: home / personal destinations each
  already run an accept loop into the shared `DeliveryEngine`; add per-dest
  serving-sealing-key selection.
- The `peers.json` fingerprint->dest resolver is no longer on the client-driven
  delivery path (the dest travels end to end). It may stay for operator-pinned
  routes, but the target model routes purely by destination.
- **Accountability** stays per-connecting-destination (`FederationRateLimiter`,
  implemented). The server root key **never** signs live federation traffic;
  destinations are self-standing and the root only certifies offline. Each
  destination signs its own delivery confirmations with its own per-destination
  signing key (above), so the signer is never server-wide.

## Residual disclosure

- No two users ever share a `dest` or a serving sealing key, so a pair of
  contact cards can never prove their owners are hosted together.
- The home server knows dest<->U (operator trust) - unchanged.
- Global passive I2P traffic-correlation is out of scope.
- Paid personal dests have no co-location signal.

The strongest cross-user clustering field (the shared server fingerprint and the
byte-identical root-signed card) is gone, and live federation now carries no
server-wide signer either (a delivery confirmation is signed per destination).

## Implementation (all phases done)

1. **common** - the contact-card / extended subscription-cert format: `dest`
   and `servingSealingKey` in the user-signed body (`TestCertificates`). Plus
   `encryptedLeaseSetHost()` and `routingHost()`, so a destination is addressed by
   the b33 of its encrypted LeaseSet2 - the only form published anywhere in the
   project - always a `.b32.i2p` host, never the raw destination
   (`isB32I2pHost`/`validateB32I2pHost`; `TestI2pAddress`).
2. **server-core** - per-destination serving sealing keys (generate, persist,
   select by the destination a connection arrived on); `GET /v1/messaging/destination`
   returns the assigned dest + public serving sealing key; the delivery envelope
   is keyed by `toDest`; federation routes/accepts by destination and **dials the
   b33** (not the raw destination - an encrypted leaseset is only reachable via
   the b33); local short-circuit only on same-destination. TCP federation removed;
   `i2p.enabled` mandatory.
3. **client** - `createInvite` / `addByInvite` / contact-request bootstrap /
   `addByAlias` on the contact card and route-by-dest; `deliver`
   targets a destination; `Client::myDestination()` fetches the assigned routing.
4. **docs** - folded into `../Identity.md`, `../Contacts.md`, `Facade.md`,
   `Federation.md`, `Architecture.md`, `Messages.md`, `DeviceSync.md`,
   `ClientApi.md`; the server card is retired from the invite/contact path.

Live-validated end-to-end over real I2P (2026-06-17, vs i2pd 2.60.0): full
single-server dialog and full cross-server Alice<->Bob conversation via invite,
with no `peers.json`, dialling each other's b33.

## Compatibility

Fresh alpha - **no** backward-compatibility shims. Old invites, server cards and
stored contacts are dropped; accounts re-subscribe to obtain a destination-routed
card. The legacy fingerprint routing path is removed, not gated.
