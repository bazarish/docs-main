# Federated resolution & contact discovery

Status: **design** (supersedes the earlier draft of this document). Specifies how
a client discovers a contact it knows only by a small scannable identifier or by
name, over I2P, with the home server (relay) learning as little as possible and
unable to forge the result. The central username registry/daemon is specified in
`AliasResolver.md`; the identity/certificate formats in `../Identity.md`; the I2P
transport in `Federation.md`.

## Two artifacts

- **Descriptor** - a small, single-QR pointer:
  ```
  bazarish://invite?v=1&fp=<52-char base32 fingerprint>&srv=<b33>.b32.i2p&srv_key=<base64 SPKI>
  ```
  identity (`fp`) + route to the user's server (`srv`, a `.b32.i2p` host) + that
  server's serving/resolve sealing key (`srv_key`). Rendered for humans as
  `fp:b33` (the `:` distinguishes it from an alias); the full link/QR also carries
  `srv_key`. It does **not** contain the contact card - it points at one.
- **Contact card** - the user-signed subscription certificate (`prekey`, `dest`,
  `servingSealingKey`; ~= 8 KB), fetched live and verified against `fp`
  (`../Identity.md`).

The old full-card-in-invite (offline, multi-QR) is **removed**: the descriptor +
a live fetch replaces it (rationale in `AliasResolver.md`). Cost: the user's
server must be reachable at add-time. Benefit: one QR, always-fresh card, and the
relay never sees who is being added (below).

## Two lookups, different trust

1. **Card fetch - `fp` is known** (from a descriptor/QR). Fetch the contact card
   for `fp` at `srv`. `fp` is the integrity anchor: the card is user-signed and
   verified against `fp`, so neither `srv` nor the relay can forge it - only
   withhold. The query carries `fp` **sealed to `srv_key`**, so the relay cannot
   read which identity is looked up. **Relay-safe by construction**, and
   substitution-proof.
2. **Alias resolve - `fp` is unknown.** `username` -> descriptor. This is the only
   step without an `fp` anchor, so it is where MITM and the trusted alias->identity
   binding live. Served by the single central resolver (`AliasResolver.md`).

## Address / contact forms

| Form | Meaning | Trust |
|---|---|---|
| `<fp>:<b33>` (descriptor) | identity + direct route; primary QR/link | strongest - `fp` anchor, no substitution (relay learns `fp` only on the proxy path) |
| `<alias>` | name on the central resolver | trusted alias->`fp` binding; UI must show the resolved `fp` for out-of-band verification |
| ~~`<x>@<fingerprint>`~~ | **removed** | a fingerprint is an identity, not a route - nothing maps it to an endpoint (no resolver, no `peers.json`) |

Aliases are global on the single resolver, so the `@server` form is unnecessary;
the protocol stays federation-capable (descriptor + signed records are general)
even though v1 runs one resolver (`AliasResolver.md`).

## Transport: direct-dial preferred, own-server proxy fallback

Mirrors the blob-storage download model.

- **Direct (preferred; the client's embedded I2P router is up).** The client dials `srv` (card
  fetch) or the resolver's b32 (alias resolve) itself over a **fresh transient
  I2P destination**. I2P encrypts the stream end-to-end to the destination and the
  `.b32.i2p` address *is* a commitment to the destination key, so the relay is
  bypassed entirely - **no relay-MITM, no relay metadata, no extra key needed**,
  and the target sees only an unlinkable transient. This is why a `.b32.i2p` host
  (not a facade) is the unit of cross-server reachability (`Facade.md`).
- **Proxy (fallback; thin client whose embedded I2P router is not up).** The client submits the sealed
  query to its **own** server, which relays opaque bytes over I2P and returns the
  opaque response. The client did all sealing/verification; the relay moves bytes.
  Here the relay could in principle MITM the *transport*, so integrity must come
  from an end-to-end anchor (below), not from the relay honestly dialling.

This revises the earlier "the client never connects to the target server itself":
thin clients proxy; clients with their own I2P dial directly (a b33 is not a
facade - direct b33 dials are already used for blob fetch).

## Card-fetch protocol (`fp` -> contact card)

```
client has descriptor {fp, srv, srv_key}
  query   = seal({ v, fp, responseKey: <ephemeral SPKI> }, srv_key)   # only srv can open
  -> dial srv (direct dial over a transient I2P destination) or relay via own server
  srv unseals, looks up the card for fp, returns:
  response = seal({ v, subscriptionCert: <b64 DER> }, responseKey)    # only the client can open
  client: verify card signature; assert card.user == fp and card.dest is .b32.i2p
```

`srv_key` is the server's serving sealing key - the same key it unseals delivery
admission headers with (`../Identity.md`, `InviteAnonymity.md`); no new key. On
the direct path the I2P stream is already confidential to `srv`, so app-layer
sealing is optional; on the proxy path it hides `fp` from the relay. Integrity is
always the `fp` anchor.

## Alias-resolve protocol (`username` -> descriptor)

The full **signed** resolve is served over the resolver's I2P endpoint only,
for uniform client automation (`AliasResolver.md`). The resolver returns a
**self-verifying signed record**:

```
sign_delegated({ v, alias, descriptor: {fp, srv, srv_key}, issuedAt, notAfter }) + delegationCert
```

The client verifies the chain `record -> delegationCert (delegated key <- root) ->
hardcoded resolver root fingerprint`, plus `notAfter`. It then runs the card-fetch
above with the descriptor. A free/unknown name returns `ALIAS_UNKNOWN`.

- **Integrity / anti-MITM** is the signature chain to the **hardcoded root
  fingerprint** - transport-independent, so even a malicious relay on the proxy
  path can only DoS, never forge. On the direct path, the resolver's b32 also
  authenticates the destination.
- **Confidentiality** of *which* name is looked up: direct-dial hides it from the
  relay entirely; on the proxy path the query is sealed to the resolver's key
  (authenticated via the hardcoded root fingerprint - bootstrap the rotatable
  sealing key from a root-signed server card).
- **Residual trust**: the resolver asserts the alias->`fp` binding (the nature of
  names). The only defense is **surfacing the resolved `fp` in the UI** for
  out-of-band verification (safety-number style); the hardcoded anchor removes the
  *relay* from trust, not the resolver itself.
- **Cacheable**: the signed record is self-verifying, so relays/clients may cache
  it up to `notAfter`, relieving load on the single resolver.

## What is hidden from whom

| Party | Card fetch | Alias resolve |
|---|---|---|
| Relay (own server), **direct** path | nothing (bypassed) | nothing (bypassed) |
| Relay, **proxy** path | "U fetched from `srv`" - not `fp` (sealed), not the card (sealed) | "U queried the resolver" - not the alias (sealed), not the result (sealed) |
| I2P network | two opaque sealed blobs | two opaque sealed blobs |
| Target (`srv` / resolver) | the `fp` (it serves that card) | the alias (it owns the namespace); requester unlinkable on direct path |
| Client | the verified card | the verified descriptor -> card |

Namespace anti-enumeration is out of scope (resolve is unauthenticated by design).

## Phasing

1. **common** - *done.* Descriptor parse/encode (`bazarish://invite?...`), the
   resolve query/response and signed-record bodies + JSON, and the delegation-cert
   (`record -> delegated -> root`) on the existing hybrid-cert primitives. Unit
   tests round-trip each.
2. **server-core** - *done.* The card-fetch responder on every serving
   destination (home/pool/per-user: unseal `fp` query -> return the card sealed to
   the response key) and the proxy relay (`POST /v1/messaging/fetch`, op `card`
   and `resolve`, `.b32.i2p`-validated `toDest`) that moves opaque sealed bytes.
3. **client** - *done.* Descriptor handling, a fetch transport that prefers a
   direct fetch over a transient I2P destination and falls back to the own-server proxy (a served
   `CARD_UNKNOWN` / `ALIAS_UNKNOWN` is authoritative and does not fall back),
   ephemeral seal/verify, address parsing (`fp:b33` vs `<alias>`), `addByUsername`
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
- Submit+poll for the proxy relay (vs synchronous) if I2P dial latency demands it.
- Optional clearnet/Tor DNS-SRV -> b33 convenience for `<alias>@<domain>` (weakest
  rung; unauthenticated DNS, opt-in, always show the `fp`).
