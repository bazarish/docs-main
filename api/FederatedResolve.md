# Federated resolution & contact discovery

Status: **design** (supersedes the earlier draft of this document). Specifies how
a client discovers a contact it knows only by a small scannable identifier or by
name, over I2P, with the home server learning nothing about it - it is not on the
path - and nothing between the two ends able to forge the result: on the
descriptor path the fingerprint anchors it outright, and on the name path the
signature chain leaves the central registry as the one party trusted for anything
(*Residual trust*, below). The identity/certificate formats are in `../Identity.md` and the I2P transport
in `Federation.md`. How the registry daemon is built and run is not public and
is not here: it is documented in its own repository
(`alias-resolver/docs/Daemon.md`).

## Two artifacts

- **Descriptor** - a small, single-QR pointer:
  ```
  bazarish://invite?v=1&fp=<52-char base32 fingerprint>&dest=<b32>.b32.i2p[&name=<percent-encoded>]
  ```
  identity (`fp`) + the user's **own** destination (`dest`, a `.b32.i2p` host),
  plus an optional `name` the
  inviter advertises (percent-encoded). The recipient adopts `name` as the
  contact's local display name at add time - a one-time static label, never
  re-fetched or transmitted afterwards (`Messages.md`). Older invites omit it.
  Rendered for humans as `fp:b32` (the `:` distinguishes it from an alias). It
  does **not** contain the contact card - it points at one.
- **Contact card** - user-signed (`t`, `issuedAt`, `dest`, `sealingKey`, `servingKey`, the owner taken from the signature;
  ~= 8.7 KB, the hybrid post-quantum signature dominates), fetched live and
  verified against `fp` (`../Identity.md`). It names **no server**: a card that
  did would tell every contact who hosts its owner, and let two cards be compared
  for co-location.

The old full-card-in-invite (offline, multi-QR) is **removed**: the descriptor +
a live fetch replaces it. Cost: the user's
server must be reachable at add-time. Benefit: one QR, always-fresh card, and the
home server never sees who is being added (below).

## Two lookups, different trust

1. **Card fetch - `fp` is known** (from a descriptor/QR). Fetch the contact card
   for `fp` at `dest`. `fp` is the integrity anchor: the card is user-signed and
   verified against `fp`, so the serving server can only withhold, never forge.

   The fetch is **direct over I2P and nothing else**: the client dials `dest`
   from a fresh transient destination. No relayed fallback exists: a lookup relayed
   through the requester's own server would disclose to it the identity being
   added. Without a router there is no fetch, and the client reports that condition
   rather than downgrading.

   Query and answer travel **in the clear**: the I2P stream is already encrypted
   and authenticated to the destination, and there is no third party on the path
   to hide them from.

   The **destination is the capability**. A blinded `.b32.i2p` address cannot be
   guessed and cannot be recovered from the network - a floodfill holds only the
   day's blinded key - so the only way to reach the card is to have received
   the descriptor. The destination answers for its own user alone, so the fetch
   cannot be used to ask a server whether it hosts some other fingerprint. A
   caller whose descriptor is out of date reaches nothing and knows to ask for a
   fresh one. Raising a new destination retires every descriptor already shared,
   and generates a new serving key with it.
2. **Alias resolve - `fp` is unknown.** `alias` -> descriptor. This is the only
   step without an `fp` anchor, so it is the step where MITM and the trusted
   alias->identity binding apply. Served by the single central resolver.

## Address / contact forms

| Form | Meaning | Trust |
|---|---|---|
| `<fp>:<b33>` (descriptor) | identity + direct route; primary QR/link | strongest - `fp` anchor, no substitution, and nothing in between to learn `fp` |
| `<alias>` | name on the central resolver | trusted alias->`fp` binding; UI must show the resolved `fp` for out-of-band verification |
| ~~`<x>@<fingerprint>`~~ | **removed** | a fingerprint is an identity, not a route - nothing maps it to an endpoint (no resolver, no `peers.json`) |

Aliases are global on the single resolver, so the `@server` form is unnecessary;
the protocol stays federation-capable (descriptor + signed records are general)
even though v1 runs one resolver.

## Transport: direct dialling, without a relay

Mirrors the direct file-transfer fetch (`FileTransfer.md`).

The client dials `srv` (card fetch) or the resolver's b32 (alias resolve) itself,
over a **fresh transient I2P destination**. I2P encrypts the stream end to end to
the destination, and the `.b32.i2p` address is itself a commitment to the
destination key; there is consequently no relay to interpose, no relay metadata
and no additional key, and the target observes only an unlinkable transient. This
is why a `.b32.i2p` host, rather than a facade, is the unit of cross-server
reachability (`Facade.md`).

**No proxy fallback exists.** Relaying a lookup through the requester's own server
would disclose to it the identity being added, which this design withholds; a
client without a working I2P router is therefore informed of that condition rather
than downgraded silently. The route that formerly provided the relay
(`POST /v1/messaging/fetch`) has been removed.

## Card-fetch protocol (`fp` -> contact card)

```
client has descriptor {fp, dest}
  query   = { v, fp }                                        # plain; the I2P stream is the envelope
  -> dial srv (direct dial over a transient I2P destination)
  srv unseals, looks up the card for fp, returns:
  response = <card DER>                                      # plain, same reason
  client: verify card signature; assert card.user == fp and card.dest is .b32.i2p
```

The serving key the sender needs for delivery
admission headers with (`../Identity.md`, `InviteAnonymity.md`); no new key. On
the only path there is, the I2P stream is already confidential to `srv`, so
app-layer sealing is optional. Integrity is always the `fp` anchor.

## Alias-resolve protocol (`alias` -> descriptor)

The full **signed** resolve is served over the resolver's I2P endpoint only,
for uniform client automation. The resolver returns a
**self-verifying signed record**:

```
sign_delegated({ v, t: "resolve-record", alias, descriptor: {fp, dest},
                 issuedAt, notAfter })
  + delegationCert     # root -> delegated
  + aliasCert          # the owner's own claim over the name, user-signed
```

The client verifies the chain `record -> delegationCert (delegated key <- root) ->
hardcoded resolver root fingerprint`, both `issuedAt` stamps against its own
clock, both `notAfter`, that the record is for the name it asked about, and the
owner's certificate - over the same name, and by the identity the descriptor
names. An answer carrying no owner certificate is refused, which is what a
registry declining to carry one would send. It then runs the card-fetch above with
the descriptor. A free/unknown name returns `ALIAS_UNKNOWN`.

- **Integrity / anti-MITM** is the signature chain to the **hardcoded root
  fingerprint** - transport-independent, so anything in the path can only DoS,
  never forge. The resolver's b32 also authenticates the destination.
- **Confidentiality** of *which* name is looked up: the client dials the resolver
  itself from a transient destination, so nothing between the two learns the name.
  Nothing is sealed on top of that, and there is no key for it: the resolver signs
  what it serves and has to read the question to answer it, so app-layer sealing
  would hide the name from nobody. The coordinate a client is built with is
  therefore two values, not three.
- **Residual trust**: the resolver asserts the alias->`fp` binding (the nature of
  names), and the owner's certificate bounds that without removing it - a name
  cannot be pointed at somebody who never signed for it, but which identity holds
  a name is the registry's to say. What the hardcoded anchor removes is everything
  else: the path can withhold an answer and nothing more. The defence left is
  **surfacing the resolved `fp`** (safety-number style) - the fingerprint the
  client adds is returned to whatever asked for the add, so it can be compared out
  of band afterwards.
- **Cacheable**: the signed record is self-verifying, so clients may cache it up
  to `notAfter`, relieving load on the single resolver.

### The alias grammar

`a-z` and `0-9` only, case-insensitive: a name is folded to lowercase as the
single canonical form (`Alice` == `alice`), so case-variant duplicates cannot
exist. No separators and no non-ASCII - a separator would let two names differ by
a character nobody reads. The length bounds are `kAliasMinLength` and
`kAliasMaxLength` in the core (1-16), and **both sides are built with the same
constants**: a client normalizes what was typed against them before it asks, the
registry canonicalizes against them, and a name outside them is answered as a
name nobody holds rather than as a fault. The cap also keeps an alias
structurally distinct from the 52-character base32 fingerprint, so the two can
never be confused.

### What a name can answer

`ALIAS_UNKNOWN` is the only negative, and it covers every reason a name does not
resolve: never registered, released, bound to nothing, or lapsed and inside the
registry's own grace. A client therefore cannot tell those apart and should
not attempt to.

A name that stops resolving is not a contact lost: resolve is an **onboarding**
step, and whoever already holds a descriptor keeps messaging over dest-routing
without ever consulting the resolver again. Only new lookups stop.

### The owner's own ops

On the same I2P face, an owner keeps their own names working. Each request is
signed by the owner's identity - the signature is what names them, so no request
carries a fingerprint field anybody could fill in and there is no way to ask
about somebody else - and each answer is signed by the resolver's delegated key
and travels with its delegation certificate, so a device that receives an
answer from a sibling device checks it against the same built-in root as a
record it fetched itself.

```
POST /v1/op/alias.status            every name this account holds
POST /v1/op/alias.update            point one of them at {fp, dest}, with
                                    the owner's alias certificate over the name
POST /v1/op/alias.renew             pay for another term from the deposit
POST /v1/op/alias.autorenew         let the service pay, or stop it
POST /v1/op/alias.binding           make one of them resolve, or stop it
POST /v1/op/alias.transfer.accept   take a name offered to this account
```

`alias.binding` carries a boolean, like `alias.autorenew`. Switching it off
clears the stored descriptor, so the name answers nobody while the account keeps
holding it and keeps paying for it; switching it on makes the name resolvable
again and leaves the client to repoint it with `alias.update`. The same switch
exists on the registry's own website, and either side may operate it.

**Two flags per name, and only one of them is the client's.** `alias.status`
reports both. `bindingWanted` is whether the name resolves, and either the client
or the website may set it. The second flag says whether the name belongs in the
messenger at all, and **only the website sets it**: an account may hold a hundred
names for other purposes while five are the ones it wants to be reachable by, and
a client lists those five alone. Switching a name's resolving off in the client
does not take it out of that list - it stays, switched off, and can be switched
back on without the website. A client therefore filters its own display on the
website flag and services every name that carries `bindingWanted`, including ones
it does not show: after a move, each of them has to be repointed by the key that
signed it, and nobody else can do it.

A request carries its own `issuedAt` and is refused outside a freshness window;
the op named in the envelope must be the op the signature covers, so relabelling
an envelope cannot turn a status into an update. A status answer carries its own
window too, which is how long it may stand in for a freshly fetched one when a
device forwards it to the account's other devices.

### What the face refuses, and what it counts

The resolve face answers nobody it cannot name: the i2pd tunnel in front puts the
caller's own `.b32.i2p` on each request, having stripped that header from what
the caller sent, and a request arriving without one is refused. **Resolves are
counted against a per-caller window** and a client must expect to be told it has
asked too often; an owner's ops are not counted, because they are signed, are one
per name held, and a client with twenty names has to be able to repoint all twenty
in one run.

### The coordinate a client is built with

Two values, and no more: the resolver's **root fingerprint** and the **b32 it
answers at**. There is no third key, because the resolver signs and never
encrypts - so nothing a client holds goes stale when the resolver rotates its
signing material behind the same root. The root fingerprint is the single
constant `bazarish::kResolverRootFingerprint`, the same one the daemon is built
with. With either half empty the alias path reports that resolution is
unavailable rather than guessing. A test or a client pointed at another
resolver is given its coordinate explicitly (`Session::setResolverCoordinate`);
there is no second, ambient way to set it.

### Why one registry at all

A globally unique sellable name needs a single registry, as a domain does. The
centralization is bounded to the name binding: the resolver is **out of the
delivery path** (routing is federated and never consults it after the one-time
lookup), it cannot forge an identity (descriptors and cards are user-signed), and
it cannot see who talks to whom. Names are a convenience layer over a base
that works without them - two people can always exchange `bazarish://invite?...`
descriptors by QR or link and never contact the resolver (`../Identity.md`). How
that registry is built, what it charges and how it is operated are its own
concern and are not specified here.

## What is hidden from whom

| Party | Card fetch | Alias resolve |
|---|---|---|
| The requester's own server | Nothing: it is not on the path, and no path through it exists | Nothing: it is not on the path |
| I2P network | nothing: the stream is encrypted and authenticated to the destination, and the bodies inside it travel in the clear for that reason | the same |
| Target (the serving server / resolver) | the `fp` (it serves that card) and that someone holding the descriptor asked | the alias (it owns the namespace); requester unlinkable on direct path |
| Client | the verified card | the verified descriptor -> card |

Namespace anti-enumeration is out of scope for the resolver (resolve is unauthenticated by design). The **card fetch** is not: it answers only on the account's own destination, which has to have been shared, and is not routed through the clearnet facade at all - otherwise a contact could take the descriptor it legitimately holds and try it against every facade it knows until one answered "yes, here".

## Phasing

1. **common** - *done.* Descriptor parse/encode (`bazarish://invite?...`), the
   resolve query/response and signed-record bodies + JSON, and the delegation-cert
   (`record -> delegated -> root`) on the existing hybrid-cert primitives. Unit
   tests round-trip each.
2. **server-core** - *done.* The card-fetch responder on every serving
   destination (the home one and every user's own: read the `{fp}` query,
   answer with the card when the capability matches and refuse identically when it
   does not; both directions in the clear, for the reason above). The proxy relay
   that once backed a thin client is gone, along with the route it was served on.
3. **client** - *done.* Descriptor handling, a fetch transport that dials the
   target directly over a transient I2P destination (a served `CARD_UNKNOWN` /
   `ALIAS_UNKNOWN` is authoritative), address parsing (`fp:b33` vs `<alias>`), `addByAlias`
   routed to the resolver (record chain verified against the hardcoded root, the
   owner's certificate checked beside it, the typed name normalized against the
   core's own `a-z0-9` bounds before anything is asked), and OOB-fingerprint
   display (the resolved fingerprint is returned to the CLI/GUI). The resolver
   coordinate is the two values the shipped binary is built with (root fingerprint
   + dest), overridable in a session for a test resolver. Unit tests drive
   resolve/card over an injected transport (happy path, unknown name, wrong root,
   alias mismatch, a certificate over another name or by another identity).
4. **alias-resolver** - *implemented* (its own repository, not specified here):
   registry + lifecycle,
   signed resolve over I2P (live-validated on i2pd), portal/payments (sign-in-with-key,
   atomic balance-only purchase, balance/top-up, manage), delegation-cert signing,
   hourly lifecycle scheduler, and **offline signing custody with a hot swap** -
   the root is offline, the daemon holds only a short delegated key, and a rotation
   takes effect on the next record with no restart. The destination is not this
   daemon's at all: an operator i2pd HTTP server tunnel is pointed at its loopback
   API, so custody of that key is the operator's i2pd configuration and moving the
   daemon is a tunnel change. Its root fingerprint and that address are built into
   the client (`defaultResolverCoordinate`), enabling the `<alias>` path in
   production.
5. **docs + live** - *done.* Both the card-fetch (`fp:b33`) path and the full
   `<alias>` path (add-user -> I2P resolve -> fp:b33 card fetch -> conversation)
   are live-validated over real I2P, and the deployed resolver's coordinate is the
   one the client is built with.

## Open / follow-up

- `bazarish://resolver/<...>` sharing for a federated (non-central) resolver - the
  protocol supports it; v1 ships one resolver only.
- Optional clearnet/Tor DNS-SRV -> b33 convenience for `<alias>@<domain>` (weakest
  rung; unauthenticated DNS, opt-in, always show the `fp`).
