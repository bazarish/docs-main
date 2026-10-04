# Alias Resolver (central alias service)

Status: **implemented**. The `bazarish/alias-resolver` repository (9th repo) - a
single global daemon run only by the developer. It owns the global `alias`
namespace (the project's monetization) and serves the cryptographically-signed
`alias -> descriptor` resolution. The resolve **protocol** (wire format,
descriptor, direct-dial vs proxy, address forms) is specified in
`FederatedResolve.md`; this document
specifies the **daemon**: its shape, the web portal, the alias lifecycle, key
rotation and payments.

## Implementation status

Built and unit-tested (`AliasRegistry`, `ResolveResponder`, `ResolveApi`,
`Lifecycle`, `PortalApi`, `AdminApi`), and **live-validated over real I2P end to
end**: the daemon serves the signed-resolve loop behind an operator i2pd HTTP
server tunnel - it runs no router and holds no destination key of its own - a
client claims a name through the portal, and another client's `add-user <alias>`
resolves it over a transient I2P destination, fetches the owner's contact card
(fp:b33) and opens a conversation - the resolved fingerprint matching the owner's
real identity. What is done:

- **Registry + lifecycle** - `alias -> descriptor` with `active -> frozen (30-day
  anti-hijack grace) -> free`, state a pure function of term + clock; atomic claim,
  yearly renew, owner-only redeem, server-migration descriptor update, post-grace
  release. Held as **one file per name** under `<state>/aliases`, so a name with
  no file is a name nobody holds and every change writes that one file: measured
  at ten thousand names, a change costs under a millisecond against the second of
  CPU and 109 MB of rewriting the whole namespace cost when it lived in a single
  document. What a name's record weighs is its owner's certificate (8.1 KB of
  hybrid signature, 11.4 KB in the file), and that sets the cost of start-up -
  every file is read and every certificate checked, 1.9 ms per name.
- **Signed resolve over I2P** - a self-verifying record signed by the delegated
  key, with the delegation cert and the owner's own certificate over the name
  beside it; verified end to end against the client's `verifyResolveRecord` and
  over the wire. Nothing on this face encrypts: the query arrives over a direct
  I2P stream, already confidential to this destination, and what a record is worth
  rests on its signature rather than on who could read the question.
- **Web portal** - sign-in-with-key, display-only search, atomic balance-only
  purchase (a length-tier debit and the name held, pointing nowhere),
  balance/top-up, owned-name management; reuses the operator service node's
  payment/auth primitives **as a library** (own ledger/wallet/prices, no runtime
  coupling).
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

Three faces, one daemon, each on a socket of its own:

1. **Web portal (the registry) - plain HTTP.** Authentication is
   **sign-in-with-key** (the client signs a challenge; the key never enters the
   browser - the one contract in `SignInWithKey.md`). The registry names itself in
   the challenge - "Bazarish alias resolver" at the address the operator fronts it
   under (`portal.address`, which the daemon refuses to start without), role
   `Alias owner` - and the client shows those words before signing. The operator
   (the developer) fronts this server to **i2p (i2pd server tunnel) + clearnet
   (TLS) + tor (onion)** with their own infrastructure; the daemon itself knows
   nothing about multi-network exposure. The portal is the human-facing surface
   (search, buy, balance, manage, hand over).
2. **Signed resolve and owner maintenance - over I2P only.** Plain HTTP on
   loopback, reached through an operator i2pd **HTTP server tunnel**: the
   destination is that tunnel's, so the daemon runs no router and holds no
   destination key. The full **signed** resolve (`FederatedResolve.md`) is served
   **only here**, as are the owner's own signed ops (`alias.status`,
   `alias.update`, `alias.renew`, `alias.autorenew`, `alias.transfer.accept`). The
   tunnel names each caller in `X-I2P-DestB32` - it strips that header from the
   caller's own request first, so it cannot be forged - and the face refuses
   anything arriving without a caller. The **window is on the resolve alone**
   (five per ten seconds per caller by default): a resolve is somebody typing a
   name, while an owner's op is signed, is one per name they hold, and a client
   holding twenty names must be able to repoint all twenty in one run. The web
   portal's alias search is a human **display** convenience
   (free/taken/yours), not the trusted artifact a client adds a contact from, and
   the descriptor a name resolves to never leaves over the web at all.
3. **The operator's face - plain HTTP.** The overview, one account in full, manual
   credit, key rotation and the signed `/healthz`. A second web server rather than
   a path under the first, so the registry can be published without publishing
   this. Its session is its own: a registry cookie is not an operator cookie.

The registry store and the payment module sit behind the portal; neither web face
limits its callers, because on this side of the operator's front nothing can name
one - see *Settled*.

## Alias syntax

Aliases are **`a-z` and `0-9` only, case-insensitive**: input is normalized to
lowercase as the single canonical form (`Alice` == `alice`). No separators or
non-ASCII, and case-folding removes case-variant duplicates. The length bounds are
`kAliasMinLength`/`kAliasMaxLength` in the core (1-16), and every side is built
with them: the registry canonicalizes against them, the client normalizes a typed
name against them before anything is asked of the registry, and a name outside
them is answered as a name nobody holds. The cap also keeps an alias structurally
distinct from the 52-character base32 fingerprint, so the two can never be
confused. The pricing tiers count characters.

## Portal features

- **Alias search.** Look up a `alias`: shows the bound **descriptor**
  (`{fp, dest, view}`, the normal form) if taken, or "free" if available.
  Display only - the authoritative signed resolve is the I2P path.
- **Atomic purchase of a free name.** Claim a free `alias` -> debit balance ->
  hold it for the signed-in account (its `fp`), all under one lock/transaction so
  two buyers cannot race the same name. Purchase is **balance-only**: the buyer
  must already hold sufficient balance (no name is ever held pending an invoice -
  the only asynchronous payment step is top-up). `POST /portal/buy` is driven with
  the name alone, because a browser holds no key and no address: **buying a name
  and pointing it somewhere are two separate acts.** A bought name points nowhere
  and answers no resolve until its owner asks for a binding (off by default) and
  their own client supplies one - a **descriptor** (`{fp, dest, view}`) together
  with an **alias certificate** (user-signed `{alias, user: fp}`, `Identity.md`),
  both produced by that client, the signing key never leaving it. The registry
  checks the certificate against the signer and the name, keeps it, and serves it
  with every record. The binding arrives either as the owner's signed
  `alias.update` over I2P or through `POST /portal/update`; both run the same
  check, in one place.
- **Balance.** Prepaid ledger (`LedgerStore`, reused): top-up (a monero or
  bitcoin invoice via the payment gateway -> idempotent credit), history display,
  current balance.
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

Denominated in **US dollars**, counted in micro-dollars, and **constants in the
code** rather than a deployment setting: the price of a name is a decision, and a
name's own price is frozen into it when it is bought, so changing the list can
never disturb a name that already exists. The price is **per alias length**, in
eight tiers - **1, 2, 3, 4, 5, 6, 7-15 and 16 characters** - where 1 is the
premium against squatting and the longest allowed length is the floor, so that
nobody is left unable to hold a name at all. The figures are not repeated here:
the portal's price page asks the code for them, so a published list cannot drift
from what is charged.

Both **purchase** and **yearly renewal** debit the balance by the price frozen
into the name. It is re-read from the list only when the name changes hands, which
is what stops an early cheap price being passed on indefinitely. There is no
per-purchase invoice: the only asynchronous payment step is **top-up** (a monero
or bitcoin invoice -> confirmed -> balance credited, at a rate taken as the median
of several independent venues); purchase and renewal are instant balance debits.

## Signed resolve (I2P-only)

A client resolves `alias` by dialling the resolver's hardcoded b32 over I2P
(a direct fetch over a transient I2P destination of the client's own - `FederatedResolve.md`)
and receives a **signed descriptor record**:

```
sign_delegated({ v, t: "resolve-record", alias, descriptor: {fp, dest, view},
                 issuedAt, notAfter })
  + delegationCert          # root -> delegated
  + aliasCert               # the owner's own claim over the name, user-signed
```

The client verifies the chain `record -> delegationCert (delegated <- root) ->
hardcoded root fingerprint`, both `issuedAt` stamps against its own clock, both
`notAfter`, that the record is for the name it asked about, and the owner's
certificate: over the same name, and by the identity the descriptor names. An
answer without that certificate is refused. The two signatures answer different
questions - the registry's says "this name resolves here", the owner's says "I am
this name" - and the second is the one the registry cannot write for somebody who
never signed it. What remains the registry's word, and the reason the resolved
fingerprint is surfaced, is *which* identity holds a name at all.

A `frozen`/unknown name returns `ALIAS_UNKNOWN`, as does a name outside the
length bounds and a name held but never bound. Because the record is
self-verifying, it is **cacheable** up to `notAfter` (bounded by the daemon's
`recordTtl` and clamped to the name's own term, so a record never vouches past the
name's expiry) - this relieves load on the single resolver and softens its
availability as a single point.

**The daemon keeps what it has already answered**, both the record and the
nothing a free name gets, so a repeat of a name costs neither a signature (two
hybrid signatures and some 38 KB of it) nor a walk of the registry. What makes
that safe is not a timer: every change to the namespace bumps a counter the kept
answers are keyed on, so a withdrawn binding cannot be answered from memory, and
an answer is never served past the `notAfter` it carries - the `Cache-Control` on
it is the remaining life of the record being served, not a fixed window.
`answerCacheMegabytes` (500 by default, 0 to keep nothing) bounds what this may
occupy; past that the least recently asked for goes first. A rotation of the
signing key drops it all, since the point of rotating is that the next answer
carries the new key. Confidentiality of *which* name a client looks up
is the transport's: the client dials the resolver itself from a throwaway
destination, so nothing in between learns the name, and nothing is sealed on top -
the resolver has to read the question to answer it.

## Key rotation (seamless, root stays offline)

Two independent anchors; the master/root keys never leave the developer's offline
custody, and production holds only short-lived delegated material, so a prod
compromise is bounded to the delegation window. The signing rotation is **operated
through the admin face** (upload form, validated server-side against the anchor);
the destination is the operator's i2pd configuration and nothing this daemon does.

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

The client hardcodes the resolver coordinate `{root fingerprint, resolver b32}`
(`defaultResolverCoordinate()`), baked into the shipped binary by the developer -
**exactly two values, because the resolver signs and never encrypts**, so there is
no third key to ship and nothing goes stale when the signing material rotates
behind the same root. The root fingerprint is the single constant
`bazarish::kResolverRootFingerprint`, which the daemon is built with too: a
delegation this daemon accepts is one every client accepts. With either half empty
the alias path reports that resolution is unavailable. A test or a differently
pointed client is handed its coordinate through `Session::setResolverCoordinate`;
there is no ambient second way in.

## Payments

No runtime coupling to operator service nodes. The payment layer lives in this
repository and nowhere else in the project: `PaymentGateway` with a Monero, a
Bitcoin and a mock backend, `LedgerStore`, `InvoiceStore` and `RateOracle`, on the
**developer's** wallet, prices and ledger. Buying an alias transfers payment to
the developer over the portal; the buyer's home operator is never involved, and
there is no inter-operator settlement. An operator's service node carries none of
this - not a gateway, not a ledger, not an abstraction of one: an account there is
free, and registering is proving a key.

## Settled

- **The web faces carry no limiter, and that is where it stays.** Only the I2P
  face limits its callers, because only there is the caller named by something it
  cannot forge - the tunnel's `X-I2P-DestB32`. On the web faces the operator's
  front (i2pd tunnel, TLS reverse proxy, onion) is the only place a caller has a
  name at all, so the limit belongs there; a limiter here keyed on a header a
  caller writes itself would be a control that holds for nobody. Namespace
  enumeration is not what it would buy either: a name's availability is public by
  construction - it is a shop - and the price page says the same thing the search
  does. Not with a puzzle in any case: a picture a person reads is a picture a
  solver reads.
