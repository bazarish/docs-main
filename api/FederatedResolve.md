# Federated resolution & contact discovery

Status: **design** (supersedes the earlier draft of this document). Specifies how
a client discovers a contact it knows only by a small scannable identifier or by
name, over I2P, with the home server learning nothing about it - it is not on the
path - and no party able to forge the result. The central alias registry/daemon is specified in
`AliasResolver.md`; the identity/certificate formats in `../Identity.md`; the I2P
transport in `Federation.md`.

## Two artifacts

- **Descriptor** - a small, single-QR pointer:
  ```
  bazarish://invite?v=1&fp=<52-char base32 fingerprint>&dest=<b32>.b32.i2p&view=<32 hex>[&name=<percent-encoded>]
  ```
  identity (`fp`) + the user's **own** destination (`dest`, a `.b32.i2p` host) +
  the card-read capability its server issued (`view`, 16 random bytes as hex),
  plus an optional `name` the
  inviter advertises (percent-encoded). The recipient adopts `name` as the
  contact's local display name at add time - a one-time static label, never
  re-fetched or transmitted afterwards (`Messages.md`). Older invites omit it.
  Rendered for humans as `fp:b32` (the `:` distinguishes it from an alias); the
  full link/QR also carries `view`. It does **not** contain the contact card -
  it points at one.
- **Contact card** - user-signed (`t`, `issuedAt`, `dest`, `sealingKey`, `servingKey`, the owner taken from the signature;
  ~= 8.7 KB, the hybrid post-quantum signature dominates), fetched live and
  verified against `fp` (`../Identity.md`). It names **no server**: a card that
  did would tell every contact who hosts its owner, and let two cards be compared
  for co-location.

The old full-card-in-invite (offline, multi-QR) is **removed**: the descriptor +
a live fetch replaces it (rationale in `AliasResolver.md`). Cost: the user's
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

   The `view` in the query is what makes the fetch unaskable without a
   descriptor: the server answers only when it matches the capability it issued
   for that account, and a wrong one is refused **identically** to an unknown
   user. Without that, anyone could ask a server whether it hosts a fingerprint.
   A caller whose descriptor is out of date gets that one answer and knows to ask
   for a fresh one. Rotating `view` retires every descriptor already shared -
   which a key doing double duty in the delivery path could never do.
2. **Alias resolve - `fp` is unknown.** `alias` -> descriptor. This is the only
   step without an `fp` anchor, so it is where MITM and the trusted alias->identity
   binding live. Served by the single central resolver (`AliasResolver.md`).

## Address / contact forms

| Form | Meaning | Trust |
|---|---|---|
| `<fp>:<b33>` (descriptor) | identity + direct route; primary QR/link | strongest - `fp` anchor, no substitution, and nothing in between to learn `fp` |
| `<alias>` | name on the central resolver | trusted alias->`fp` binding; UI must show the resolved `fp` for out-of-band verification |
| ~~`<x>@<fingerprint>`~~ | **removed** | a fingerprint is an identity, not a route - nothing maps it to an endpoint (no resolver, no `peers.json`) |

Aliases are global on the single resolver, so the `@server` form is unnecessary;
the protocol stays federation-capable (descriptor + signed records are general)
even though v1 runs one resolver (`AliasResolver.md`).

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
client has descriptor {fp, dest, view}
  query   = { v, fp, view }                                  # plain; the I2P stream is the envelope
  -> dial srv (direct dial over a transient I2P destination)
  srv unseals, looks up the card for fp, returns:
  response = <card DER>                                      # plain, same reason
  client: verify card signature; assert card.user == fp and card.dest is .b32.i2p
```

`view` is a read capability and nothing else - it encrypts nothing. The serving key the sender needs for delivery
admission headers with (`../Identity.md`, `InviteAnonymity.md`); no new key. On
the only path there is, the I2P stream is already confidential to `srv`, so
app-layer sealing is optional. Integrity is always the `fp` anchor.

## Alias-resolve protocol (`alias` -> descriptor)

The full **signed** resolve is served over the resolver's I2P endpoint only,
for uniform client automation (`AliasResolver.md`). The resolver returns a
**self-verifying signed record**:

```
sign_delegated({ v, alias, descriptor: {fp, dest, key}, issuedAt, notAfter }) + delegationCert
```

The client verifies the chain `record -> delegationCert (delegated key <- root) ->
hardcoded resolver root fingerprint`, plus `notAfter`. It then runs the card-fetch
above with the descriptor. A free/unknown name returns `ALIAS_UNKNOWN`.

- **Integrity / anti-MITM** is the signature chain to the **hardcoded root
  fingerprint** - transport-independent, so anything in the path can only DoS,
  never forge. The resolver's b32 also authenticates the destination.
- **Confidentiality** of *which* name is looked up: the client dials the resolver
  itself, so nothing between the two learns the name. The query is sealed to the
  resolver's key regardless (authenticated via the hardcoded root fingerprint -
  bootstrap the rotatable sealing key from a root-signed server card).
- **Residual trust**: the resolver asserts the alias->`fp` binding (the nature of
  names). The only defense is **surfacing the resolved `fp` in the UI** for
  out-of-band verification (safety-number style); the hardcoded anchor removes
  everything but the resolver itself from trust.
- **Cacheable**: the signed record is self-verifying, so clients may cache it up
  to `notAfter`, relieving load on the single resolver.

## What is hidden from whom

| Party | Card fetch | Alias resolve |
|---|---|---|
| The requester's own server | Nothing: it is not on the path, and no path through it exists | Nothing: it is not on the path |
| I2P network | two opaque sealed blobs | two opaque sealed blobs |
| Target (the serving server / resolver) | the `fp` (it serves that card) and that someone holding the descriptor asked | the alias (it owns the namespace); requester unlinkable on direct path |
| Client | the verified card | the verified descriptor -> card |

Namespace anti-enumeration is out of scope for the resolver (resolve is unauthenticated by design). The **card fetch** is not: it answers only to a caller returning the descriptor's `view` capability, refuses a wrong one exactly as it refuses an unknown user, and is not routed through the clearnet facade at all - otherwise a contact could take the descriptor it legitimately holds and try it against every facade it knows until one answered "yes, here".

## Phasing

1. **common** - *done.* Descriptor parse/encode (`bazarish://invite?...`), the
   resolve query/response and signed-record bodies + JSON, and the delegation-cert
   (`record -> delegated -> root`) on the existing hybrid-cert primitives. Unit
   tests round-trip each.
2. **server-core** - *done.* The card-fetch responder on every serving
   destination (the home one and every user's own: unseal `fp` query -> return the
   card sealed to the response key). The proxy relay that once backed a thin
   client is gone, along with the route it was served on.
3. **client** - *done.* Descriptor handling, a fetch transport that dials the
   target directly over a transient I2P destination (a served `CARD_UNKNOWN` /
   `ALIAS_UNKNOWN` is authoritative),
   ephemeral seal/verify, address parsing (`fp:b33` vs `<alias>`), `addByAlias`
   routed to the resolver (record chain verified against the hardcoded root,
   alias normalized to `a-z0-9`/1-32), and OOB-fingerprint display (the resolved
   fingerprint is returned to the CLI/GUI). The resolver coordinate (root fp +
   dest + serving key) is a configurable placeholder until phase 4 ships. Unit
   tests drive resolve/card over an injected transport (happy path, unknown name,
   wrong root, alias mismatch).
4. **alias-resolver** - *implemented* (`AliasResolver.md`): registry + lifecycle,
   signed resolve over I2P (live-validated on i2pd), portal/payments (sign-in-with-key,
   atomic balance-only purchase, balance/top-up, manage), delegation-cert signing,
   hourly lifecycle scheduler, and **offline-key destination custody + hot-swap
   rotation** - the master is offline, the daemon runs a short transient, and a
   rotation serves at the same b32 while the old destination drains (live-validated
   over real I2P). The seamless transient hot-swap requires **i2pd >= 2.61.0**
   (the upstream offline-transient refresh + I2P destination teardown fixes). Once deployed,
   its root fingerprint + destination + serving key are baked into the client
   (`defaultResolverCoordinate`), enabling the `<alias>` path in production.
5. **docs + live** - cross-I2P validation. Both the card-fetch (`fp:b33`) path
   and the full `<alias>` path (add-user -> I2P resolve -> fp:b33 card fetch ->
   conversation) are live-validated over real I2P; production use waits only on
   the resolver deployment (baking the coordinate into the client).

## Open / follow-up

- `bazarish://resolver/<...>` sharing for a federated (non-central) resolver - the
  protocol supports it; v1 ships one resolver only.
- Optional clearnet/Tor DNS-SRV -> b33 convenience for `<alias>@<domain>` (weakest
  rung; unauthenticated DNS, opt-in, always show the `fp`).
