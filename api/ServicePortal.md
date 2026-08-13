# Service Portal (account onboarding)

Status: implemented.

The **service node** owns user accounts. It exposes a **web portal** (the
"Service site") through which a user registers and proves ownership of their key.
The **messaging server (server-core)** stays a pure delivery layer: it never
registers anyone. A client can sign and fetch, but **cannot self-register** -
registration is a human/web flow guarded by a captcha.

**Nothing on this node is for sale.** There is no balance, no ledger, no payment
gateway and no paid service: the only paid component of the project is the alias
service, which is a separate (private) deployment with its own portal and its own
payment layer (`AliasResolver.md`). Everything a messaging account gets - its
storage quota, its own I2P destination - it gets for free.

```
            +----------- browser -----------+         +---- Client (holds key) ----+
            | Service portal (HTML, JS/JS-free)|<------| sign-login <challenge>     |
            +---------------------------------+  paste | (key never enters browser) |
                           | HTTP                 sig   +----------------------------+
                  +--------v---------+
                  |   service node   |  accounts, captcha, approval
                  +------------------+
       issues subscription | certs
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
  "registration": {
    "captcha":          true,
    "requiresApproval": false,
    "message":          "<how to reach the operator, when approval is required>"
  },
  "quotaBytes": 52428800
}
```

The client shows the message and the link(s); the user opens one in a browser.
Several links allow mirrors/onion/clearnet alternatives. There is no service
catalogue because nothing is sold here: the only variables are whether a captcha
is required, whether an operator has to approve the account, and the one storage
quota everybody gets. (The global `username` namespace is a separate product sold
by the central alias-resolver, not the service node - see `AliasResolver.md`.)

server-core, when it rejects an unsubscribed sender, returns an error that points
at the same portal.

## 3. Account model

Per user (`subscriptions/<fp>.json`, `grants/<fp>.json`):

- **subscription** - issued on registration; the hybrid-signed subscription
  certificate the client later fetches via `GET /v1/account/subscription`.
- **approval** - `requestedAt` and `approvedAt`. On an open node an account is
  approved the moment it is created; on a moderated one it stays pending (see
  below). A renewal never restarts `requestedAt`, so a pending account cannot
  outlive its deadline by renewing.
- **transient** - the validity of the offline I2P transient the client last
  delegated, recorded so any of the user's devices knows when to issue the next
  one without holding the expiry itself.

Unsubscribing tears the account down rather than merely stopping renewal: the
destination is revoked and the grants are dropped.

### The user's own I2P destination (key custody + lifecycle)

Every account has its own serving destination - it is part of the base protocol,
not a purchase, and there is no shared pool to fall back on (a shared address
would make every user behind it provably co-located).

The master I2P destination key stays on the **client** (the user's long-term
routing identity, sealed at rest in the profile beside the messaging keys). The
client either **generates** a fresh master or **loads** one it already owns from
an unencrypted Ed25519 (signing type 7) `.dat`. The server only ever holds a
short-lived **offline transient** delegated from that master, so the b32 is the
user's own and stable as they move between servers, and an operator is locked out
once its transient expires.

Because the server cannot blind a leaseset it has no master key for, a user's
destination publishes a **standard** LeaseSet (a plain b32, enumerable by
floodfills) rather than an encrypted one. That is the accepted cost of never
sharing an address; every other destination in the project stays encrypted.

A destination can only be delegated once the account exists, so the client
subscribes first, delegates its transient, and immediately re-publishes its
contact card (`POST /v1/account/renew` inside the term it already holds, which
grants nothing and consumes no grant) with the routing folded in. On a moderated
node the delegation is refused until an operator approves the account, so the
client publishes its routing then instead.

The transient is short (~= 1 week). The client re-issues a fresh one ~= 2 days
before it lapses, with a few hours of per-device **jitter**, and always **polls
`GET /v1/account/i2p-status` before issuing** - so when one of the user's devices
has already renewed, the others stand down (the multi-device race). Keeping the
transient fresh is therefore an unconditional background duty of the client: let
it lapse and the destination goes down and senders cannot deliver.

On install/enable the client also sends the master to its **other devices** as a
`device.i2p-master` service message - sealed to the account's own sealing key and
delivered tokenlessly to the account's own destination - so every device adopts
the same master (preserving the b32) and can re-issue transients.

The server provisions/serves the destination on its embedded I2P router
(`POST /v1/account/i2p-dest` -> the messaging server's operator-internal
per-user-destination endpoint) and reports its state on
`GET /v1/messaging/destination` (`none` -> `building` -> `active` -> `expired`).

> Off the box: minting the offline transient and the master `.dat` is the
> client's job (vendored libi2pd); the server consumes and serves the blob but
> never generates it.

## 4. Registration: open or moderated

Behind login, gated by the configured **captcha**. On success the node issues a
subscription into the `SubscriptionStore`; the client then fetches its
certificate via `GET /v1/account/subscription`.

An operator chooses one of two modes (`registration.requireApproval`):

- **open** - the account is servable the moment it is created.
- **approval** - the account is created but **frozen**: its transient is refused
  with `ACCOUNT_PENDING_APPROVAL`, so its destination is never brought up and it
  can neither send nor receive. The error carries the operator's
  `registration.message`, which is where they say how to reach them to finish
  registration. The operator approves by hand from the admin panel. An account
  nobody approves within `registration.ttlDays` (default 7) is deleted, so a dead
  registration does not hold a fingerprint forever.

This is moderation, not a paywall.

| Endpoint | Purpose |
|---|---|
| `GET /v1/account/pending` | Operator-signed: accounts awaiting approval, oldest first |
| `POST /v1/account/approve` | Operator-signed: approve one account |

## 5. Captcha (pluggable, self-hosted default)

A `Captcha` interface (`issue() -> {id, challenge-render}`, `verify(id, answer)
-> bool`). The built-in default is a **self-hosted server-rendered image/text
captcha** - no third party, works **without JavaScript**. An operator may
register a different provider. Captcha gates free registration (and optionally
other unauthenticated actions); it never depends on an external service by
default, preserving the privacy posture.

## 6. Web frontend (JS + JS-free)

The service node serves the portal HTML on the same port as the JSON API, at
`/` (the JSON API stays under `/portal/*` and `/v1/*`, untouched). The
**JS-free** path (server-rendered forms, full-page reloads, post-redirect-get)
is the whole frontend: with nothing to pay for there is no status to poll and no
QR to render. Pages, all reusing the JSON endpoints' logic:

- **login** (`GET /login`, `POST /login`) - render a challenge; a field to paste
  the Client signature; on success set a session cookie and redirect to `/`.
- **cabinet** (`GET /`) - subscription state, the server descriptor to paste into
  a client, and - on a moderated node - the "awaiting approval" notice with the
  operator's instructions.
- **register** (`GET /register`, `POST /register`) - captcha, then the grant.

## 7. API relationships

| Caller | Endpoint | Auth | Purpose |
|---|---|---|---|
| Client | `GET /v1/account/portal` | none | onboarding text + portal links + registration terms |
| Client | `sign-login <challenge>` (local) | key | produce the login signature |
| Browser | `GET /login`, `POST /login` | sig -> session | sign-in-with-key |
| Browser | `POST /register` | session + captcha | subscription grant |
| Panel | `GET /v1/account/pending`, `POST /v1/account/approve` | operator sig | moderation |
| Client | `GET /v1/account/subscription` | key | fetch the issued certificate |
| Client | `POST /v1/account/i2p-dest` | key | upload a signed offline transient (refused while the account is unapproved) |
| Client | `GET /v1/account/i2p-status` | key | approval state and the recorded transient validity |
| Client | `GET /v1/account/contact` | none | fingerprint -> contact card lookup |

## 8. Removed (replaced by the portal)

- `POST /v1/account/subscribe` (client-initiated grant, no captcha) and
  the client `Session::subscribe` direct flow.
- `POST`/`DELETE /v1/account/alias`, `GET /v1/account/resolve`, and the on-node
  alias registry: the global `username` namespace moved out to the central
  **alias-resolver** (`AliasResolver.md`). The service node no longer sells,
  stores, or resolves names - and since the payment layer moved to the resolver
  too, it now sells nothing at all: a destination and a mailbox quota come with
  the account.

Kept: `GET /v1/account/subscription`, `contact`; the `mock` gateway
(tests); the `PaymentGateway` registry/abstraction.

## 9. Security notes

- The key never enters the browser; login is a relayed one-shot signature bound
  by `purpose` + `audience` + a fresh, HMAC-bound, single-use nonce.
- Captcha defaults to self-hosted (no third-party calls, JS-free capable).
- Sessions are short-lived signed cookies.
- Approval and deletion are operator-signed actions on the private channel; the
  portal itself can neither approve an account nor see who is waiting.

## 10. Open questions

- Whether server-core embeds portal links in its rejection error or only
  references the service node's `/v1/account/portal`.
