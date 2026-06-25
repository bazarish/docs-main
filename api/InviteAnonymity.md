# Invite Anonymity: destination-routed contact cards

Status: **implemented and live-validated** over real I2P (single-server +
cross-server `demo-federation-i2p.sh`, 2026-06-17). Supersedes the old
invite/server-card shape; `../Identity.md`, `../Contacts.md`, `Facade.md` and
`Federation.md` have been updated to match.

> **Note (later supersession).** The destination-routing, per-destination serving
> sealing keys, and route-by-b33 model below still hold. The **invite artifact**
> itself - the full ~= 8 KB user-signed contact card carried inline - has since
> been replaced by a small **descriptor** (`fp:b33` + serving sealing key) plus a
> live card fetch (`FederatedResolve.md`, `AliasResolver.md`). Read this document
> for the destination-routing rationale; read `FederatedResolve.md` for the
> current first-contact/discovery shape.

## Problem

An invite today carries two artifacts: the user's **subscription certificate**
(user-signed: `user`, `server`, prekey) and the serving server's **server card**
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
ContactCard  (CMS-signed by user U - the only artifact in an invite):
  v
  user:              <U fingerprint>     # equals the signer; the identity anchor
  issuedAt, notAfter
  prekey:            <SPKI DER>          # U's E2E sealing prekey (first message)
  dest:              <i2p destination>   # dest_U - where to deliver to U
  servingSealingKey: <SPKI DER>          # key the delivery envelope is sealed to
```

No `server` fingerprint, no endpoints list, no separate root-signed card, no
root signature. The card carries **exactly one signature - U's**; there is no
server signature anywhere on the contact path. `dest` and `servingSealingKey`
are plain fields the user vouches for under that one signature.

Why this is safe with no server signature: the first message is E2E-sealed to
`prekey`, which only U holds - confidentiality never depended on the server. If
`dest`/`servingSealingKey` were tampered, delivery merely fails (the recipient
server cannot produce a forged acceptance the sender's client would believe, and
no plaintext is exposed). The server was already untrusted for content; routing
is now an unsigned hint the user vouches for, not a trust root.

### Two keys, one signature - why `servingSealingKey` exists

`servingSealingKey` (the `sealingKey_U` of the decided
`sign_U{prekey, sealingKey_U, dest_U}` shape) is a **public encryption key
embedded in the user-signed card** - not a signature and not a trust anchor.
There are two sealed layers with **different readers**, so they need different
keys:

| Layer | Contents | Decrypted by | Key |
|---|---|---|---|
| Content (E2E) | the message itself | **U** (while offline, on next sync) | `prekey` |
| Admission header | `mailbox` + one-time `token` | **U's server** (while U is offline) | `servingSealingKey` |

The server stores-and-forwards while U is offline, so it **must** read
`mailbox` + `token` to enforce contact-admission and quota (the token scheme,
`../Contacts.md`) - while staying blind to the content layer, which it cannot
decrypt. One key cannot serve both layers because the readers differ (U vs U's
server). Sealing the admission header is also what keeps the **sender's own
relay server** (and the I2P transport) from learning the recipient: otherwise
the sender's server would log a social-graph edge "U -> V", violating the
"sender's own server never learns the recipient" invariant (`Federation.md`);
for free users it is what preserves the pool k-anonymity.

**Ownership** (so this is not a "server signs" step): for a paid personal dest
the client may **generate `sealingKey_U` itself and delegate the private half to
its server** (like the offline destination key), making it fully
client-originated; for a free pool dest it is the server-owned key shared across
that pool dest (same k-anonymity set as the dest). Either way the card still
carries only U's signature.

## Routing by destination

`DeliveryEnvelope.toServer` (a server fingerprint) becomes `toDest` (an I2P
destination string). The sender's own server connects **directly** to `toDest`
over I2P - no fingerprint->`peers.json` resolution, no server card needed to
route. The rest of the envelope is unchanged:

```
DeliveryEnvelope:
  toDest: <i2p destination>                 # routing only - all the sender's server sees
  sealed:                                   # CMS envelope to the destination's serving sealing key
    class: content | contact
    mailbox: <recipient UID>
    token: <one-time delivery token>        # absent for class "contact"
    messageId: <random id>
  payload: <E2E blob>
```

The sender's own server still learns only "user U sent something to destination
D" - and D is shared by a whole pool of free users, so it is weaker than a
server fingerprint, not stronger.

### Local delivery only within a destination (no co-location timing channel)

A server may short-circuit delivery (store straight into the local mailbox
instead of dialling out over I2P) **only when the sender and the recipient are
on the same destination** - i.e. `toDest` equals the **sender's own** assigned
destination. In every other case, including any two local users the server hosts
on *distinct* serving destinations (e.g. two free users on different pool dests,
or a pool user and a personal-dest user), the server **must federate** - dial
`toDest` over I2P **from the sender's own serving destination**. The two endpoints
are distinct destinations on the one router; this is not a single dest dialling
itself. (The server has no shared "home" federation address to fall back on - see
the fundamental rule below.)

Rationale: if the server delivered cross-destination recipients locally (fast)
but remote recipients over I2P (slow), the **delivery latency** would reveal
whether the recipient is co-located on the sender's server - a timing
side-channel that re-introduces exactly the co-location signal the
destination-routing model removes. Routing within one destination is safe (the
two parties already share that destination by construction, so "local" leaks
nothing new); routing across destinations must be indistinguishable from a
remote delivery.

The server determines the sender's destination authoritatively from its own
assignment of the authenticated sender (the same lookup that backs the
destination-surfacing step below) - never from a value the sender supplies, so a
sender cannot force a revealing short-circuit.

## The destination knob (`dest_U`)

**Fundamental rule: a server has no single, shared federation destination.** Every
user is served by `dest_U` (a pool dest within a k-anonymity set, or a personal
per-user dest), and *all* of that user's federation rides over it - inbound
delivery, outbound delivery (dialled **from** `dest_U`), and the signed
`delivered-ack` (`Federation.md`). A server-wide "home" address is never used for
user traffic (it serves only first-contact card discovery); a shared federation
source would both leak a common address and, when sender and recipient share a
server, force the home endpoint to dial itself.

- **Free tier** - `dest_U` is one of the server's **address-pool** destinations
  (encrypted-LeaseSet b33; `server-core` `FederationPool`). Free users are
  spread deterministically across the pool (`poolIndexFor`), giving k-anonymity
  ~= N_free / poolSize. Co-location is bounded to "one of the users on this pool
  dest," never "this exact user."
- **Paid tier** - `dest_U` is the user's **personal** destination, brought up
  from the offline-key transient the user delegates (`PerUserDestinations`). It
  is the user's full bidirectional federation identity; no other server can tie
  it to the home server, so there is no co-location signal at all.

The pool/personal split already exists in `server-core`; this revision is what
finally **surfaces** the assigned destination into the user-facing card.

## Per-destination serving sealing key

The delivery envelope's routing metadata is sealed to a **serving sealing key
bound to `dest_U`**, not to one server-wide key:

- A **personal** dest has its own sealing key (effectively per-user).
- A **pool** dest has one sealing key shared by the free users on that dest -
  the same k-anonymity set as the dest itself, so it leaks nothing the dest did
  not already.

The destination server selects the key by **which destination the connection
arrived on** (each dest has its own accept loop already), so there is no
trial-decryption. After unsealing, the server reads `mailbox` and delivers
locally exactly as today. The server holds the private halves of its
destinations' serving sealing keys.

This replaces the single server sealing key carried in the server card; the
value is no longer shared server-wide.

## Subscription certificate vs contact card

The subscription certificate serves two roles today: (a) the lifecycle anchor
the serving server stores ("S serves U until T", anti-replay), and (b) the
peer-facing routing/contact statement. This revision keeps both but re-anchors
them:

- The server still stores a user-signed subscription cert as its lifecycle
  anchor, but the binding "this cert is for me" is checked against the
  **destination + serving sealing key the server operates** (the pool dest it
  assigned, or the personal dest the user delegated), not against the server's
  root fingerprint. A server accepts a subscription iff the cert's `dest` /
  `servingSealingKey` are ones it serves.
- The **contact card** above is the peer-facing artifact. It is the same signed
  body shape, minus anything server-identifying beyond the destination.

Net: the `server` fingerprint disappears from every peer-visible artifact.

## How the user learns `dest_U` and the serving sealing key

At subscribe time (grant-gated; see `ServicePortal.md`) the server returns, in
place of the server card:

- the assigned `dest` (pool dest for free; the personal dest brought up from the
  delegated offline transient for paid), and
- the **public** serving sealing key for that dest.

The client folds both into its self-signed contact card and into the cert it
stores. The server keeps the private serving sealing key and the dest<->mailbox
mapping.

## Flow migration

- **createInvite** - emit the single user-signed contact card; no server card.
- **addByInvite** - verify the card; deliver to `card.dest`, seal the envelope
  to `card.servingSealingKey`. No server card, no fingerprint resolve.
- **contact-request bootstrap** (`../Contacts.md`) - the bootstrap ships the
  sender's **contact card** (dest + serving sealing key + prekey) instead of the
  sender's server card, so the peer replies by federating to the sender's dest.
- **addByUsername / alias resolve** - the resolved trust chain becomes the
  contact card (not sub-cert + server card). The cross-server sealed `resolve`
  (`Federation.md`, still pending - the Stage-2 track) keeps a **server-level**
  resolve key for its entry point: the client cannot seal to a per-dest key for
  a user whose dest it does not yet know. That server-level resolve key is the
  one residual server-scoped artifact, used only to bootstrap alias lookups, and
  is out of scope for the invite revision.
- **Groups** (`../Groups.md`) - the signed roster carries each member's contact
  card (dest + serving sealing key + prekey) instead of sub-cert + server card.
- **Local contact state** - store `contact.dest` + `contact.servingSealingKey`
  in place of `contact.server` + server-card sealing key.

## Federation impact

- `toServer` -> `toDest`; the sender's server dials `toDest` directly.
- **Local short-circuit only within a destination**: deliver locally iff
  `toDest` equals the **sender's own** destination; otherwise federate (dial
  `toDest`), even when the server operates `toDest` (a same-server recipient on a
  different destination) - see "Local delivery ..." above. This is the rule, not an
  optimisation; a fast local path for a different-destination recipient is a
  co-location timing leak.
- Accept side is unchanged in shape: home / pool / personal destinations each
  already run an accept loop into the shared `DeliveryEngine`; add per-dest
  serving-sealing-key selection.
- The `peers.json` fingerprint->dest resolver is no longer on the client-driven
  delivery path (the dest travels end to end). It may stay for operator-pinned
  routes, but the target model routes purely by destination.
- **Accountability** stays per-connecting-destination (`FederationRateLimiter`,
  implemented). The server root key **never** signs live federation traffic;
  destinations are self-standing and the root only certifies offline.

## Honest accounting (what still leaks)

- Free users on one pool dest share `dest` + serving sealing key -> k-anonymity
  ~= N_free / poolSize (tunable by pool size, default 10).
- The home server knows dest<->U (operator trust) - unchanged.
- Global passive I2P traffic-correlation is out of scope.
- Paid personal dests have no co-location signal.

The strongest cross-user clustering field (the shared server fingerprint and the
byte-identical root-signed card) is gone.

## Implementation (all phases done)

1. **common** - the contact-card / extended subscription-cert format: `dest`
   and `servingSealingKey` in the user-signed body (`TestCertificates`). Plus
   `encryptedLeaseSetHost()` (b33), `standardLeaseSetHost()` (standard b32) and
   `routingHost()` so a destination is addressed by its b33
   (encrypted LeaseSet2) or its standard b32 (standard LeaseSet2, offline-key
   per-user) - always a `.b32.i2p` host, never the raw destination
   (`isB32I2pHost`/`validateB32I2pHost`; `TestI2pAddress`).
2. **server-core** - per-destination serving sealing keys (generate, persist,
   select by the destination a connection arrived on); `GET /v1/messaging/destination`
   returns the assigned dest + public serving sealing key; the delivery envelope
   is keyed by `toDest`; federation routes/accepts by destination and **dials the
   b33** (not the raw destination - an encrypted leaseset is only reachable via
   the b33); local short-circuit only on same-destination. TCP federation removed;
   `--i2p` mandatory.
3. **client** - `createInvite` / `addByInvite` / contact-request bootstrap /
   `addByUsername` / groups on the contact card and route-by-dest; `deliver`
   targets a destination; `Client::myDestination()` fetches the assigned routing.
4. **docs** - folded into `../Identity.md`, `../Contacts.md`, `Facade.md`,
   `Federation.md`, `Architecture.md`, `Messages.md`, `Groups.md`, `DeviceSync.md`,
   `ClientApi.md`; the server card is retired from the invite/contact path.

Live-validated end-to-end over real I2P (2026-06-17, vs i2pd 2.60.0): full
single-server dialog and full cross-server Alice<->Bob conversation via invite,
with no `peers.json`, dialling each other's b33.

## Compatibility

Fresh alpha - **no** backward-compatibility shims. Old invites, server cards and
stored contacts are dropped; profiles re-subscribe to obtain a destination-routed
card. The legacy fingerprint routing path is removed, not gated.
