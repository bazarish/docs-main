# Service Portal (account onboarding, payments, paid services)

Status: design (2026-06-15).

The **service node** owns user accounts. It exposes a **web portal** (the
"Service site") through which a user registers, proves ownership of their key,
tops up a prepaid balance, and buys paid services (a username, a personal
I2P destination). The **messaging server (server-core)** stays a pure delivery
layer: it never sees payments and never registers anyone. A client can sign and
fetch, but **cannot self-register** — registration is a human/web flow guarded by
a captcha (free tier) and payment (paid tier).

```
            ┌─────────── browser ───────────┐         ┌──── Client (holds key) ────┐
            │ Service portal (HTML, JS/JS-free)│◄──────┤ sign-login <challenge>     │
            └──────────────┬──────────────────┘  paste │ (key never enters browser) │
                           │ HTTP                 sig   └────────────────────────────┘
                  ┌────────▼─────────┐
                  │   service node   │  accounts, balance ledger, captcha,
                  │  + payments mod  │  payment gateways (config-driven)
                  └────────┬─────────┘
       issues subscription │ / alias certs, sets entitlements
                  ┌────────▼─────────┐
                  │   server-core    │  delivery only; points unsubscribed
                  └──────────────────┘  senders at the portal
```

## 1. Identity & login (sign-in-with-key)

A portal session is bound to a **userId = the user's key fingerprint**, proven by
a signature — the same identity used for messaging. The private key never enters
the browser.

1. The portal renders a **challenge**:
   `challenge = { v, purpose:"bazarish-login", nonce, ts, audience }`
   where `audience` is the service node's fingerprint and `nonce` is bound by an
   HMAC under a portal secret (`tag = HMAC(secret, nonce‖ts‖audience)`), so the
   portal can later confirm it issued the challenge and that `ts` is fresh
   **without server-side state**.
2. The user signs the challenge with their Client (`bazarish-client sign-login
   <challenge>`), producing a **hybrid signature** (classical + PQ, the common
   `Auth`/CMS primitive) over the canonical challenge. The Client emits an
   opaque base64 blob; the user pastes it back into the portal.
3. The portal verifies: signature valid; `ts` within the freshness window
   (e.g. ±300 s); `audience` == this node; HMAC tag matches; `nonce` not already
   consumed (a short-lived used-nonce cache stops in-window replay). The
   **signer fingerprint = userId**.
4. On success the portal sets a session (a signed cookie carrying `userId` and an
   expiry; HttpOnly, SameSite=Strict). All subsequent account actions — top-ups,
   alias registration, buying services — are logged under that `userId`.

Rationale: the key stays in the Client; the browser only relays a one-shot
signature. `purpose` and `audience` stop the signed blob being replayed against
another protocol surface or another server.

## 2. Portal discovery (server → links)

The client learns where to onboard from an **unauthenticated** endpoint:

```
GET /v1/account/portal  →
{
  "message": "<human text shown by the client>",
  "links":   ["https://portal-a.example/…", "https://portal-b.example/…"],  // ≥1
  "services": {
    "freeRegistration": { "enabled": true, "captcha": true },
    "username":  { "enabled": true, "priceAtomic": "...", "periodDays": 365, "currency": "XMR" },
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
sell a username, a personal I2P destination, and/or a larger storage quota.

**Every paid service is recurring (term-based) and renewed from the cabinet —
none are one-shot.** This is deliberate: an ongoing server-side cost must map to
ongoing payment, otherwise a one-time fee lets a user load the server with a huge
number of always-online destinations for almost nothing. Recurring pricing turns
that into a competitive market of servers (whoever offers a given resource
cheaper wins), instead of a tragedy of the commons. Terms:

- **username** — registered for **one year**, renewed yearly from the cabinet. A
  username is a pure naming layer: it maps `alias → fingerprint` and resolves to
  **whatever I2P destination that user currently uses, even one operated by
  another server** (destinations are user-owned and portable, see `Facade.md` /
  `Federation.md`). So a server can sell names independently of where the named
  user's traffic is actually carried.
- **i2pDest** — a personal always-online destination, billed **monthly** (its
  tunnels/leaseset are a continuous resource).
- **storage** — every account gets `freeQuotaBytes` free (the config minimum,
  10 MiB by default); the paid plan raises the quota and is billed **monthly**.

A lapsed term reverts the service to its free baseline (name released, dest
dropped to the shared pool, quota back to the free minimum). server-core, when it
rejects an unsubscribed sender, returns an error that points at the same portal.

## 3. Account model

Per user (`subscriptions/<fp>.json`, `ledger/<fp>.json`):

- **subscription** — issued on free registration (captcha) or on a paid plan;
  the hybrid-signed subscription certificate the client later fetches via the
  existing `GET /v1/account/subscription`.
- **balance** — a prepaid balance in **atomic units of the settlement currency**
  (integer string, never a float; piconero for monerod), plus the currency code.
- **ledger** — append-only entries `{ ts, type: topup|spend, amount, currency,
  ref, balanceAfter }`. Top-ups credit; purchases debit. This is the account
  history the portal shows.
- **entitlements** — each is term-based with its own paid-through date, renewed
  from the cabinet, reverting to a free baseline on lapse:
  - `username` (alias → fingerprint, yearly) — a naming registration that
    resolves to the user's portable destination, possibly carried by another
    server (see `Facade.md` / `Federation.md`);
  - `i2pDest` (the per-user-destination flag honoured by the messaging server /
    pool provisioning, monthly);
  - **storage quota** — the subscription's `quotaBytes`, the free default
    (10 MiB) until a paid plan raises it, monthly.

## 4. Free registration

Behind login, gated by the configured **captcha**. On success the node issues a
free-tier subscription into the `SubscriptionStore`; the client then fetches its
certificate via `GET /v1/account/subscription`. No payment, no balance touched.

## 5. Payments (config-driven gateways)

The gateway and the paid-service prices are selected **by config**, not
hardcoded — a single JSON file passed as `--payments-config <path>`, the one
source feeding both the discovery advertisement and the buy endpoints (see
`service-node/payments.example.json`):

```json
{
  "gateway": { "type": "monerod", "params": { "url": "...", "user": "...", "password": "...", "minconf": "10", "callback_secret": "..." } },
  "settlementCurrency": "XMR",
  "freeStorageBytes": 10485760,
  "services": {
    "username": { "enabled": true, "priceAtomic": "...", "periodDays": 365 },
    "i2pDest":  { "enabled": true, "priceAtomic": "...", "periodDays": 30 },
    "storage":  { "enabled": true, "priceAtomic": "...", "periodDays": 30, "quotaBytes": ... }
  }
}
```

With no config file the node runs a local-dev default (an auto-confirm `mock`
gateway and cheap services). The code ships **one** real backend — **monerod** —
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

`PaymentState ∈ { pending, paid, confirmed, expired, failed }`. `Money.amount`
is an exact decimal/atomic string (no floats).

### monerod backend

Talks to a `monero-wallet-rpc`. An invoice is a **fresh subaddress** for the
account; the invoice id encodes its address index.

- **createInvoice** → `create_address` → `{ address, address_index }`; `payTo` is
  the subaddress (rendered with the amount as a `monero:` URI / QR).
- **checkStatus** → `get_transfers { in:true, subaddr_indices:[index] }`; sum the
  received amount and report confirmations against the configured `minconf`.
- **callback (push)** → the node exposes `POST /v1/pay/callback/monerod`, called
  by an operator-side watcher (e.g. a `get_transfers` poller or a wallet
  tx-notify hook). The body carries `{ invoiceId, txid }` with an HMAC over a
  configured secret; the gateway **re-verifies against the wallet** (never trusts
  the callback's amount) and settles the invoice.
- **verifyByTxid (pull)** → the frontend submits a `txid` the user paid with; the
  gateway calls `get_transfer_by_txid` and confirms it credits our subaddress for
  ≥ the amount due. This is the fallback when the callback was missed.

Both paths converge on the same wallet-verified settlement, then the **balance**
is credited and a ledger entry written.

## 6. Captcha (pluggable, self-hosted default)

A `Captcha` interface (`issue() -> {id, challenge-render}`, `verify(id, answer)
-> bool`). The built-in default is a **self-hosted server-rendered image/text
captcha** — no third party, works **without JavaScript**. An operator may
register a different provider. Captcha gates free registration (and optionally
other unauthenticated actions); it never depends on an external service by
default, preserving the privacy posture.

## 7. Web frontend (JS + JS-free)

The service node serves the portal HTML. Every flow has a **JS-free** path
(server-rendered forms, full-page reloads) and a **JS-enhanced** path (live
status polling, inline QR):

- **login** — render challenge; field to paste the Client signature.
- **dashboard** — balance, ledger, available services.
- **register** — captcha → free subscription.
- **top-up / pay** — create invoice; show `payTo` + amount + QR; JS polls
  `GET /pay/{id}`; JS-free offers a "refresh" link and a "I paid → enter txid"
  form (`POST /pay/{id}/txid`).
- **buy username / buy i2p-dest** — debit balance, grant entitlement.

## 8. API relationships

| Caller | Endpoint | Auth | Purpose |
|---|---|---|---|
| Client | `GET /v1/account/portal` | none | onboarding text + portal links + offered services |
| Client | `sign-login <challenge>` (local) | key | produce the login signature |
| Browser | `GET /login`, `POST /login` | sig → session | sign-in-with-key |
| Browser | `POST /register` | session + captcha | free subscription |
| Browser | `POST /topup`, `GET /pay/{id}`, `POST /pay/{id}/txid` | session | balance + payment |
| Browser | `POST /buy/username`, `POST /buy/i2p-dest` | session | spend balance, grant entitlement |
| watcher | `POST /v1/pay/callback/<gateway>` | HMAC | payment push notification |
| Client | `GET /v1/account/subscription` | key | fetch the issued certificate |
| Client | `GET /v1/account/resolve`, `/contact` | none | messaging lookups (unchanged) |

## 9. Removed (replaced by the portal)

- `POST /v1/account/subscribe` (client-initiated grant, no captcha/payment) and
  the client `Session::subscribe` direct flow.
- `POST /v1/account/alias` direct registration; aliases become a portal purchase
  (the alias certificate is still user-signed; the node stores it once the user
  is entitled).
- The `bitcoind` payment backend (the abstraction stays; only `monerod` ships).

Kept: `GET /v1/account/subscription`, `resolve`, `contact`; the `mock` gateway
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

- Multi-currency balances / price feeds (monorod-only for now → single currency).
- Abuse/revocation: clawback of a paid service on chargeback/fraud (crypto =
  final, so low risk; fiat gateways later need this).
- Whether server-core embeds portal links in its rejection error or only
  references the service node's `/v1/account/portal`.
