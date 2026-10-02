# Alias Resolver (central alias service)

Status: **implemented** (one hardening pending - see *Implementation status*). The
`bazarish/alias-resolver` repository (9th repo) - a single global daemon run only
by the developer. It owns the global `alias` namespace (the project's
monetization) and serves the cryptographically-signed `alias -> descriptor`
resolution. The resolve **protocol** (wire format, descriptor, direct-dial vs
proxy, address forms) is specified in `FederatedResolve.md`; this document
specifies the **daemon**: its shape, the web portal, the alias lifecycle, key
rotation and payments.

## Implementation status

Built and unit-tested (`AliasRegistry`, `ResolveResponder`/`ResolveServer`,
`Lifecycle`, `PortalApi`), and **live-validated over real I2P end to end**: the
daemon builds its standard-LeaseSet2 destination on its embedded I2P router and serves the
signed-resolve loop, a client claims a name through the portal, and another
client's `add-user <alias>` resolves it over a transient I2P destination, fetches the
owner's contact card (fp:b33) and opens a conversation - the resolved fingerprint
matching the owner's real identity. What is done:

- **Registry + lifecycle** - `alias -> descriptor` with `active -> frozen (30-day
  anti-hijack grace) -> free`, state a pure function of term + clock; atomic claim,
  yearly renew, owner-only redeem, server-migration descriptor update, post-grace
  release.
- **Signed resolve over I2P** - query opened with the serving key, a self-verifying
  record signed by the delegated key + delegation cert, sealed to the response key;
  verified end to end against the client's `verifyResolveRecord` and over the wire.
- **Web portal** - sign-in-with-key, display-only search, atomic balance-only
  purchase (alias cert + descriptor + length-tier debit), balance/top-up, owned-name
  management; reuses the operator service node's payment/auth primitives **as a
  library** (own ledger/wallet/prices, no runtime coupling).
- **Lifecycle scheduler** - hourly auto-renew-from-balance + release-after-grace.
- **Admin face** (config `admin`) - an admin-gated portal (sign-in-with-key;
  the signed-in fingerprint must equal the configured admin, else no admin routes
  exist at all): a dashboard (active/frozen name counts, outstanding top-up invoices,
  total user balance, payments received all-time / last year / last 30 days, and live
  key status), **manual balance credit** by user fingerprint (grant balance for
  buying/renewing), and **key rotation** (below).
- **Signing-key rotation (operator-driven, hot)** - the admin uploads new signing
  material (delegated key PEM + delegation cert), validated against the root anchor,
  persisted and hot-swapped **immediately**. The root fingerprint stays constant, so
  nothing a client holds changes.

- **The daemon holds no destination key.** Its address is an operator i2pd **HTTP
  server tunnel** pointed at the loopback API, so the destination is the tunnel's and
  the daemon runs no router at all - which is also what makes the hardcoded resolver
  address an operator concern rather than a key this process could leak. The tunnel
  is what names each caller (`X-I2P-DestB32`, stripped from the caller's own request
  first so it cannot be forged), and the face refuses anything arriving without one.
- **Signing custody is offline.** The operator generates the permanent root
  **offline** with `bazarish-alias-resolver-signing generate-root` (kept air-gapped,
  never on the live box) and issues short delegated keys from it
  (`issue-delegation <root.pem> <days> <out-prefix>`). The daemon holds only a
  delegated key and its certificate: a compromise leaks at most an expiring
  delegation, never the root. Live-validated over real I2P: a daemon serving behind
  its tunnel answers a resolve that verifies against the compiled-in root rather than
  against anything it handed itself, and a name nobody holds comes back as a typed
  `ALIAS_UNKNOWN` - which only a daemon that received the frame could have written.

This service is intentionally **centralized**: a globally-unique sellable name
requires a single registry (like a domain registrar). The centralization is
bounded to the name binding - the resolver is **out of the delivery path**
(dest-routing is federated and never consults it after the one-time lookup), it
cannot forge identities (descriptors/cards are user-signed), and it cannot see
who-talks-to-whom. Names are optional sugar over a censorship-resistant base:
users can always exchange `bazarish://invite?...` descriptors directly (QR/link),
bypassing the resolver entirely (`Identity.md`, `FederatedResolve.md`).

## Daemon shape

Two faces, one daemon:

1. **Web portal - plain HTTP on `127.0.0.1`.** Authentication is **sign-in-with-key**
   (the client signs a challenge; the key never enters the browser - the one
   contract in `SignInWithKey.md`). The registry names itself in the challenge -
   "Bazarish alias registry" at the address the operator fronts it under, role
   `Alias owner` - and the client shows those words before signing. Not yet
   implemented here: this daemon still builds against an older core and issues
   the previous challenge envelope. The operator (the developer) fronts this localhost server
   to **i2p (i2pd server tunnel) + clearnet (TLS) + tor (onion)** with their own
   infrastructure; the daemon itself knows nothing about multi-network exposure.
   The portal is the human-facing surface (search, buy, balance, manage).
2. **Signed resolve API - over I2P only.** The daemon owns its own I2P
   destination (offline-key-delegated, see *Key rotation*), whose **b32** is the
   hardcoded resolver address in every client. The full **signed** resolve
   (`FederatedResolve.md`) is served **only here**, for uniformity of client
   automation. The web portal's alias search is a human **display** convenience
   (shows the descriptor or "free"), not the trusted artifact a client adds a
   contact from.

The registry store and the payment module sit behind the localhost web server;
the embedded I2P router is used **only** for the external signed-resolve API.

## Alias syntax

Aliases are **`a-z` and `0-9` only, case-insensitive**: input is normalized to
lowercase as the single canonical form (`Alice` == `alice`). No separators or
non-ASCII, and case-folding removes case-variant duplicates. Length 1-32; the
pricing tiers (1, 2, 3, 4, 5+) count characters. The 1-32 length cap keeps an
alias structurally distinct from the 52-character base32 fingerprint, so the two
can never be confused.

## Portal features

- **Alias search.** Look up a `alias`: shows the bound **descriptor**
  (`{fp, dest, view}`, the normal form) if taken, or "free" if available.
  Display only - the authoritative signed resolve is the I2P path.
- **Atomic purchase of a free name.** Claim a free `alias` -> debit balance ->
  bind to the signed-in account (its `fp`), all under one lock/transaction so two
  buyers cannot race the same name. Purchase is **balance-only**: the buyer must
  already hold sufficient balance (no name is ever held pending an invoice - the
  only asynchronous payment step is top-up). The buyer submits an **alias
  certificate** (user-signed `{alias, user: fp}`, `Identity.md`) plus their
  **descriptor** (`{fp, dest, view}`; `dest`/`view` come from the buyer's home
  server). Both are produced by the buyer's own client - the signing key never
  leaves it - and the portal `POST /portal/buy` is driven with that JSON. The resolver verifies the cert (signer ==
  fp), claims the name, and stores `alias -> descriptor`.
- **Balance.** Prepaid ledger (`LedgerStore`, reused): top-up (monero invoice via
  the payment gateway -> idempotent credit), history display, current balance.
- **Yearly auto-renewal from balance.** Aliases are **term-based (1 year)**. A
  background scheduler renews each due alias by debiting the balance: funds
  present -> extend `notAfter` +1 year; insufficient -> **freeze** (below).
- **Manage.** Update the bound descriptor (e.g. after the owner migrates servers -
  the alias then points at the new `dest`/`view`); view owned aliases and their
  state/expiry.
- **Admin face** (opt-in via the config's `admin`). Admin-gated pages
  (sign-in-with-key; the signed-in fingerprint must equal the configured admin): an
  overview dashboard (active/frozen names, outstanding top-up invoices, total user
  balance, payments received all-time / last year / last 30 days, live key status
  with install/expiry times), **manual balance credit** by user fingerprint (grant
  balance to oneself or others for buying/renewing - recorded as `admin-credit:`, so
  it never counts as a received payment), and the **key rotation** upload forms
  (*Key rotation*). With no admin fingerprint configured the admin routes do not
  exist.

## Alias lifecycle

```
   purchase
      |
      v
  +--------+  renewal debit OK (yearly)   +--------+
  | active | ---------------------------> | active |   (resolves; signed descriptor served)
  +--------+                              +--------+
      | renewal debit fails
      v
  +--------+  owner pays (redeem)
  | frozen | ---------------------------> active
  | (30d)  |   - NOT resolvable (name stops working - staleness signal)
  +--------+   - NOT purchasable by anyone (anti-hijack grace)
      | 30d elapse, still unpaid
      v
  +------+
  | free |   released to the pool - purchasable by anyone again
  +------+
```

- **Freeze is the anti-hijack window.** When a name lapses it is **not** instantly
  grabbable: for 30 days only the original owner can redeem it (by paying); during
  that window the name does not resolve and no one else can buy it. This protects
  against snipe-on-expiry.
- **Existing contacts are unaffected by freeze.** Resolve is an onboarding-only
  operation; contacts who already hold the descriptor keep messaging over
  dest-routing without consulting the resolver. Freezing only stops **new**
  lookups.
- **Post-freeze release.** After the 30-day grace an unpaid alias returns to the
  free pool (purchasable by anyone again), so names remain reclaimable and cannot
  be held hostage by freeze indefinitely.

## Pricing

Fixed in config, denominated in **piconero** (XMR pico; 1 XMR = 1012 pico). The
price is **per alias length**, with five tiers - 1, 2, 3, 4, and **5-or-more**
characters - where the 5+ tier is the floor (cheapest) and shorter names are
premium:

| Alias length | Price |
|---|---|
| 1 char | `price[1]` (highest) |
| 2 chars | `price[2]` |
| 3 chars | `price[3]` |
| 4 chars | `price[4]` |
| >= 5 chars | `price[5+]` (lowest, floor) |

Both **purchase** and **yearly renewal** debit the balance by the alias's
length-tier price. There is no per-purchase invoice: the only asynchronous
payment step is **top-up** (monero invoice -> confirmed -> balance credited);
purchase and renewal are instant balance debits.

## Signed resolve (I2P-only)

A client resolves `alias` by dialling the resolver's hardcoded b32 over I2P
(a direct fetch over a transient I2P destination of the client's own - `FederatedResolve.md`)
and receives a **signed descriptor record**:

```
sign_delegated({ v, alias, descriptor: {fp, dest, key}, issuedAt, notAfter })
  + delegationCert
```

The client verifies the chain `record -> delegationCert (delegated <- root) ->
hardcoded root fingerprint` and `notAfter`. A `frozen`/unknown name returns
`ALIAS_UNKNOWN`. Because the record is self-verifying, it is **cacheable** by
relays/clients up to `notAfter` - this relieves load on the single resolver and
softens its availability as a single point. Confidentiality of *which* name a
client looks up is handled by the transport choice (direct-dial, or sealing on the
proxy path), not by this record.

## Key rotation (seamless, root stays offline)

Two independent anchors; the master/root keys never leave the developer's offline
custody, and production holds only short-lived delegated material, so a prod
compromise is bounded to the delegation window. Both rotations are **operated through
the admin face** (upload forms, validated server-side) - see *Implementation status*
for what is wired today.

- **I2P destination - the operator's tunnel.** The daemon holds no destination key
  and runs no router: an i2pd HTTP server tunnel is pointed at its loopback API, and
  that tunnel's destination is the resolver's address. Custody of the key is
  therefore the operator's i2pd configuration, where it can be an offline-key
  transient of a master whose address never changes, and rotating or moving the
  daemon is a tunnel change rather than anything this process does. The address is a
  `....b32.i2p` host like every other in the project.
- **Signing - delegation certificates.** The root hybrid identity (offline) signs
  short-lived `{delegatedPubKey, notAfter}` delegation certs. Production signs each
  resolve record with the **current delegated key** and ships the delegation cert.
  Rotation = provision a new offline-signed delegation, switch the signing key
  atomically. The **root fingerprint is the constant anchor** hardcoded in clients,
  so rotation is invisible to them => zero downtime.

The client hardcodes only the resolver coordinate `{root fingerprint, resolver
b32, serving sealing key}` (`defaultResolverCoordinate()`), baked into the shipped
binary by the developer once the resolver is deployed - until then it is an empty
placeholder and the alias path reports "no resolver configured". The three parts
can be overridden at runtime with `BAZARISH_RESOLVER_ROOT` / `_DEST` / `_KEY` (all
required), so a freshly built or local resolver is exercised without a rebuild.

## Payments

No runtime coupling to operator service nodes. The payment layer lives in this
repository and nowhere else in the project: `PaymentGateway` with a Monero, a
Bitcoin and a mock backend, `LedgerStore`, `InvoiceStore` and `RateOracle`, on the
**developer's** wallet, prices and ledger. Buying an alias transfers payment to
the developer over the portal; the buyer's home operator is never involved, and
there is no inter-operator settlement. An operator's service node carries none of
this - not a gateway, not a ledger, not an abstraction of one: an account there is
free, and registering is proving a key.

## Open / to confirm

- Whether the human web search is rate-limited against namespace enumeration
  (the I2P signed path is the automation interface; the web search is a
  convenience and may warrant abuse controls). Not with a puzzle: a picture a
  person reads is a picture a solver reads.
