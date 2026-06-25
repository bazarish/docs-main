# Service Portal (account onboarding, payments, paid services)

Status: design (2026-06-15).

The **service node** owns user accounts. It exposes a **web portal** (the
"Service site") through which a user registers, proves ownership of their key,
tops up a prepaid balance, and buys paid services (a personal I2P
destination, a larger storage quota). The **messaging server (server-core)** stays a pure delivery
layer: it never sees payments and never registers anyone. A client can sign and
fetch, but **cannot self-register** - registration is a human/web flow guarded by
a captcha (free tier) and payment (paid tier).

```
            +----------- browser -----------+         +---- Client (holds key) ----+
            | Service portal (HTML, JS/JS-free)|<------| sign-login <challenge>     |
            +---------------------------------+  paste | (key never enters browser) |
                           | HTTP                 sig   +----------------------------+
                  +--------v---------+
                  |   service node   |  accounts, balance ledger, captcha,
                  |  + payments mod  |  payment gateways (config-driven)
                  +------------------+
       issues subscription | certs, sets entitlements
                  +--------v---------+
                  |   server-core    |  delivery only; points unsubscribed
                  +------------------+  senders at the portal
```

## 1. Identity & login (sign-in-with-key)

A portal session is bound to a **userId = the user's key fingerprint**, proven by
a signature - the same identity used for messaging. The private key never enters
the browser.

1. The portal renders a **challenge**:
   `challenge = { v, purpose:"bazarish-login", nonce, ts, audience }`
   where `audience` is the service node's fingerprint and `nonce` is bound by an
   HMAC under a portal secret (`tag = HMAC(secret, nonce||ts||audience)`), so the
   portal can later confirm it issued the challenge and that `ts` is fresh
   **without server-side state**.
2. The user signs the challenge with their Client (`bazarish-client sign-login
   <challenge>`), producing a **hybrid signature** (classical + PQ, the common
   `Auth`/CMS primitive) over the canonical challenge. The Client emits an
   opaque base64 blob; the user pastes it back into the portal.
3. The portal verifies: signature valid; `ts` within the freshness window
   (e.g. +/-300 s); `audience` == this node; HMAC tag matches; `nonce` not already
   consumed (a short-lived used-nonce cache stops in-window replay). The
   **signer fingerprint = userId**.
4. On success the portal sets a session (a signed cookie carrying `userId` and an
   expiry; HttpOnly, SameSite=Strict). All subsequent account actions - top-ups
   and buying services - are logged under that `userId`.

Rationale: the key stays in the Client; the browser only relays a one-shot
signature. `purpose` and `audience` stop the signed blob being replayed against
another protocol surface or another server.

## 2. Portal discovery (server -> links)

The client learns where to onboard from an **unauthenticated** endpoint:

```
GET /v1/account/portal  ->
{
  "message": "<human text shown by the client>",
  "links":   ["https://portal-a.example/...", "https://portal-b.example/..."],  // >=1
  "services": {
    "freeRegistration": { "enabled": true, "captcha": true },
    "i2pDest":   { "enabled": true, "priceAtomic": "...", "periodDays": 30,  "currency": "XMR" },
    "storage":   {
      "freeQuotaBytes": 10485760,
      "paid": { "enabled": true, "quotaBytes": 5368709120,
                "priceAtomic": "...", "periodDays": 30, "currency": "XMR" }
    }
  }
}
```

The client shows the message and the link(s); the user opens one in a browser.
Several links allow mirrors/onion/clearnet alternatives. The `services` block is
driven by the server's config: a node may offer only free registration, or also
sell a personal I2P destination and/or a larger storage quota. (The global
`username` namespace is a separate product sold by the central alias-resolver,
not the service node - see `AliasResolver.md`.)

**Every paid service is recurring (term-based) and renewed from the cabinet -
none are one-shot.** This is deliberate: an ongoing server-side cost must map to
ongoing payment, otherwise a one-time fee lets a user load the server with a huge
number of always-online destinations for almost nothing. Recurring pricing turns
that into a competitive market of servers (whoever offers a given resource
cheaper wins), instead of a tragedy of the commons. Terms:

- **i2pDest** - a personal always-online destination, an **opt-in toggle**
  (off by default - every user already has a fixed pool address) billed
  **monthly**. The cabinet states the privacy trade-off: a personal destination
  is portable and stable across servers, but a unique, lasting address across all
  the user's contacts (less crowd-blending than the shared pool). See
  *Per-user i2p-dest* below.
- **storage** - every account gets `freeQuotaBytes` free (the config minimum,
  10 MiB by default); the paid plan raises the quota and is billed **monthly**.

Term-based services **auto-renew from the prepaid balance**; a lapse reverts to
the free baseline, but the lapse rule differs per service so the user notices:
- **storage** - strict: when the balance cannot cover a renewal the upgrade
  drops to the free quota (a visible shrink, pushed to the messaging server) and
  stops; the user re-buys to restore it.
- **i2pDest** - a **one-step negative-balance grace**: the first unfunded
  renewal is debited anyway (the balance goes negative) and the destination is
  served a further 30 days; a second is refused while the balance is negative, so
  the personal destination goes **offline** (the user is **not** silently moved
  to the pool - that would either break already-shared descriptors or expose a
  second destination, a deanonymisation surface). The user comes back online by
  topping up (the same b32 is restored) or by turning the option **off**
  explicitly, which reverts them to their fixed pool address. A top-up that
  clears the debt re-arms the grace.

server-core, when it rejects an unsubscribed sender, returns an error that points
at the same portal.

## 3. Account model

Per user (`subscriptions/<fp>.json`, `ledger/<fp>.json`):

- **subscription** - issued on free registration (captcha) or on a paid plan;
  the hybrid-signed subscription certificate the client later fetches via the
  existing `GET /v1/account/subscription`.
- **balance** - a prepaid balance in **atomic units of the settlement currency**
  (integer string, never a float; piconero for monerod), plus the currency code.
- **ledger** - append-only entries `{ ts, type: topup|spend, amount, currency,
  ref, balanceAfter }`. Top-ups credit; purchases debit. This is the account
  history the portal shows.
- **entitlements** - each is term-based with its own paid-through date, renewed
  from the cabinet, reverting to a free baseline on lapse:
  - `i2pDest` (the per-user-destination flag honoured by the messaging server /
    pool provisioning, monthly);
  - **storage quota** - the subscription's `quotaBytes`, the free default
    (10 MiB) until a paid plan raises it, monthly.

### Per-user i2p-dest (key custody + lifecycle)

The master I2P destination key stays on the **client** (the user's long-term
routing identity, sealed at rest in the profile beside the messaging keys). The
client either **generates** a fresh master or **loads** one it already owns from
an unencrypted Ed25519 (signing type 7) `.dat`. The server only ever holds a
short-lived **offline transient** delegated from that master, so the b32 is the
user's own and stable as they move between servers, and an operator is locked out
once its transient expires.

Two clocks: the **entitlement** is monthly (auto-renewed from balance, with the
one-step grace above); the **transient** is short (~= 1 week). The client re-issues
a fresh transient ~= 2 days before it lapses, with a few hours of per-device
**jitter**, and always **polls `GET /v1/account/i2p-status` before issuing** - so
when one of the user's devices has already renewed, the others stand down
(the multi-device race). `i2p-status` reports `enabled`/`active`, the recorded
transient validity (so any device knows when to re-issue without storing it),
the effective **storage quota**, and **honest projected shutoff dates** for both
storage and i2pDest computed from the remaining balance.

On install/enable the client also sends the master to its **other devices** as a
`device.i2p-master` service message - sealed to the account's own sealing key and
delivered tokenlessly to the account's own destination - so every device adopts
the same master (preserving the b32) and can re-issue transients.

The server provisions/serves the personal destination on its embedded I2P router
(`POST /v1/account/i2p-dest` -> the messaging server's operator-internal
per-user-destination endpoint), reports its state on `GET /v1/messaging/destination`
(`building` -> `active` -> `expired`), and falls a free user back to a fixed
round-robin **pool** address (default pool size 10; see `Federation.md`).

> Off the box: minting the offline transient and the master `.dat` is the
> client's job (vendored libi2pd); the server consumes and serves the blob but
> never generates it.

## 4. Free registration

Behind login, gated by the configured **captcha**. On success the node issues a
free-tier subscription into the `SubscriptionStore`; the client then fetches its
certificate via `GET /v1/account/subscription`. No payment, no balance touched.

## 5. Payments (config-driven gateways)

The gateway and the paid-service prices are selected **by config**, not
hardcoded - a single JSON file passed as `--payments-config <path>`, the one
source feeding both the discovery advertisement and the buy endpoints (see
`service-node/payments.example.json`):

```json
{
  "gateway": { "type": "monerod", "params": { "url": "...", "user": "...", "password": "...", "minconf": "10", "callback_secret": "..." } },
  "settlementCurrency": "XMR",
  "freeStorageBytes": 10485760,
  "services": {
    "i2pDest":  { "enabled": true, "priceAtomic": "...", "periodDays": 30 },
    "storage":  { "enabled": true, "priceAtomic": "...", "periodDays": 30, "quotaBytes": ... }
  },
  "portal": {
    "message": "<onboarding text shown by the client>",
    "links": ["https://portal-a.example", "http://mirror.b32.i2p"],
    "captcha": { "enabled": true, "length": 5, "ttlSeconds": 300 }
  }
}
```

The `portal` block drives onboarding discovery (section 2): `message` and `links` are
advertised verbatim by `GET /v1/account/portal`, and `captcha.enabled` is both
advertised there as `freeRegistration.captcha` **and** enforced at
`POST /portal/register` - the two cannot disagree. With `captcha.enabled` false,
free registration is open to any signed-in (sign-in-with-key) user without a
captcha; `length`/`ttlSeconds` size the self-hosted captcha when it is on.

With no config file the node runs a local-dev default (an auto-confirm `mock`
gateway and cheap services). The code ships **one** real backend - **monerod** -
plus the deterministic `mock` used in tests. Backends implement the
`PaymentGateway` interface and register by `type`; a self-written gateway plugs
in via `registerPaymentGateway(type, factory)` with no change to the plumbing.

### PaymentGateway interface (server-side)

```
createInvoice(amount, orderRef) -> Invoice { id, payTo, amount, expiresAt, state }
checkStatus(invoiceId)          -> PaymentStatus { state, paidAmount, confirmations }
verifyByTxid(invoiceId, txid)   -> PaymentStatus     // pull path (callback missed)
parseWebhook(body, signature)   -> PaymentStatus?    // push path (callback)
```

`PaymentState  in  { pending, paid, confirmed, expired, failed }`. `Money.amount`
is an exact decimal/atomic string (no floats).

### monerod backend

Talks to a `monero-wallet-rpc`. An invoice is a **fresh subaddress** for the
account; the invoice id encodes its address index.

- **createInvoice** -> `create_address` -> `{ address, address_index }`; `payTo` is
  the subaddress (rendered with the amount as a `monero:` URI / QR).
- **checkStatus** -> `get_transfers { in:true, subaddr_indices:[index] }`; sum the
  received amount and report confirmations against the configured `minconf`.
- **callback (push)** -> the node exposes `POST /v1/pay/callback/monerod`, called
  by an operator-side watcher (e.g. a `get_transfers` poller or a wallet
  tx-notify hook). The body carries `{ invoiceId, txid }` with an HMAC over a
  configured secret; the gateway **re-verifies against the wallet** (never trusts
  the callback's amount) and settles the invoice.
- **verifyByTxid (pull)** -> the frontend submits a `txid` the user paid with; the
  gateway calls `get_transfer_by_txid` and confirms it credits our subaddress for
  >= the amount due. This is the fallback when the callback was missed.

Both paths converge on the same wallet-verified settlement, then the **balance**
is credited and a ledger entry written.

## 6. Captcha (pluggable, self-hosted default)

A `Captcha` interface (`issue() -> {id, challenge-render}`, `verify(id, answer)
-> bool`). The built-in default is a **self-hosted server-rendered image/text
captcha** - no third party, works **without JavaScript**. An operator may
register a different provider. Captcha gates free registration (and optionally
other unauthenticated actions); it never depends on an external service by
default, preserving the privacy posture.

## 7. Web frontend (JS + JS-free)

The service node serves the portal HTML on the same port as the JSON API, at
`/` (the JSON API stays under `/portal/*` and `/v1/*`, untouched). The
**JS-free** path (server-rendered forms, full-page reloads, post-redirect-get)
is implemented; the first **JS-enhanced** layer - inline, same-origin live
payment-status polling on the pay page - is implemented on top (with the
Refresh link as the JS-free fallback). Inline QR is still planned (kept out of
the JS-free path; a JS QR renderer would add a third-party dependency). Pages,
all reusing the JSON endpoints' logic:

- **login** (`GET /login`, `POST /login`) - render a challenge; a field to paste
  the Client signature; on success set a session cookie and redirect to `/`.
- **cabinet** (`GET /`) - balance, subscription state, offered services.
- **register** (`GET /register`, `POST /register`) - inline captcha (when
  enabled) -> free subscription.
- **top-up / pay** (`GET /topup`, `POST /topup`, `GET /pay/{id}`) - create
  invoice; show `payTo` + amount; JS-free offers a "refresh" link and a
  "I paid -> enter txid" form (`POST /pay/{id}/txid`); JS may poll `GET /pay/{id}`.
- **buy** (`POST /buy`, `service` field) - debit balance, grant/renew the
  entitlement.

## 8. API relationships

| Caller | Endpoint | Auth | Purpose |
|---|---|---|---|
| Client | `GET /v1/account/portal` | none | onboarding text + portal links + offered services |
| Client | `sign-login <challenge>` (local) | key | produce the login signature |
| Browser | `GET /login`, `POST /login` | sig -> session | sign-in-with-key |
| Browser | `POST /register` | session + captcha | free subscription |
| Browser | `POST /topup`, `GET /pay/{id}`, `POST /pay/{id}/txid` | session | balance + payment |
| Browser | `POST /buy` (`service` field) | session | spend balance, grant/renew entitlement |
| Browser | `POST /i2p-dest` (+ JSON `POST /portal/i2p-dest`) | session | turn the per-user i2p-dest option on/off |
| watcher | `POST /v1/pay/callback/<gateway>` | HMAC | payment push notification |
| Client | `GET /v1/account/subscription` | key | fetch the issued certificate |
| Client | `POST /v1/account/i2p-dest/setting` | key | turn the per-user i2p-dest option on/off |
| Client | `POST /v1/account/i2p-dest` | key | upload a signed offline transient (gated by an active i2pDest entitlement) |
| Client | `GET /v1/account/i2p-status` | key | enabled/active, transient validity, storage quota, balance-projected shutoff dates |
| Client | `GET /v1/account/contact` | none | fingerprint -> contact card lookup |

## 9. Removed (replaced by the portal)

- `POST /v1/account/subscribe` (client-initiated grant, no captcha/payment) and
  the client `Session::subscribe` direct flow.
- `POST`/`DELETE /v1/account/alias`, `GET /v1/account/resolve`, and the on-node
  alias registry: the global `username` namespace moved out to the central
  **alias-resolver** (`AliasResolver.md`). The service node no longer sells,
  stores, or resolves names; it sells only `i2pDest` and `storage`.
- The `bitcoind` payment backend (the abstraction stays; only `monerod` ships).

Kept: `GET /v1/account/subscription`, `contact`; the `mock` gateway
(tests); the `PaymentGateway` registry/abstraction.

## 10. Security notes

- The key never enters the browser; login is a relayed one-shot signature bound
  by `purpose` + `audience` + a fresh, HMAC-bound, single-use nonce.
- Payment settlement is **always** re-verified against the wallet; callbacks are
  triggers, not sources of truth. Money amounts are integer atomic units.
- Captcha defaults to self-hosted (no third-party calls, JS-free capable).
- Sessions are short-lived signed cookies; paid-service grants and balance moves
  are logged in the per-user ledger.

## 11. Open questions

- Multi-currency balances / price feeds (monorod-only for now -> single currency).
- Abuse/revocation: clawback of a paid service on chargeback/fraud (crypto =
  final, so low risk; fiat gateways later need this).
- Whether server-core embeds portal links in its rejection error or only
  references the service node's `/v1/account/portal`.
