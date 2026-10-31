# Service Portal (account onboarding)

The **service node** owns user accounts. It exposes a **web portal** (the
"Service site") through which a user registers and proves ownership of their key.
The **messaging server (server-core)** stays a pure delivery layer: it never
registers anyone. Proving the key is the whole registration, whether it is proven
in the portal or by a client calling `POST /v1/account/registration`; there is no
puzzle in front of it, because anything a person reads off a picture a solver
reads too.

**This node sells nothing.** It maintains no balance, no ledger and no payment
gateway, and offers no paid service. The only paid component of the project is the
alias service, a separate deployment with its own portal and payment layer
(`FederatedResolve.md`). What a messaging account receives - a storage quota and its
own I2P destination - is included.

```
            +----------- browser -----------+         +---- Client (holds key) ----+
            | Service portal (HTML, JS/JS-free)|<------| signs the challenge        |
            +---------------------------------+  paste | (key never enters browser) |
                           | HTTP                 sig   +----------------------------+
                  +--------v---------+
                  |   service node   |  accounts and approval      
                  +------------------+
       grants registration | quota
                  +--------v---------+
                  |   server-core    |  delivery only; relays the node's
                  +------------------+  answers unchanged
```

## 1. Identity & login (sign-in-with-key)

A portal session is bound to a **userId = the user's key fingerprint**, proven by
a signature - the same identity used for messaging. The private key never enters
the browser.

The wire contract is one for the whole project and is specified in
`SignInWithKey.md`: the portal issues a challenge naming its **consumer**, the
user signs it in their client, the portal verifies the blob and consumes the
challenge, and the signer's fingerprint is the userId. On success the portal sets
a session cookie carrying `userId` and an expiry (HttpOnly, SameSite=Strict).

What this node puts in the consumer, and how an operator changes it without a
restart, is in `SignInWithKey.md`.

## 2. Portal discovery (server -> links)

The client learns where to onboard from an **unauthenticated** endpoint:

```
GET /v1/account/portal  ->
{
  "message": "<human text shown by the client>",
  "links":   ["https://portal-a.example/...", "https://portal-b.example/..."],  // >=1
  "registration": {
    "requiresApproval": false,
    "message":          "<how to reach the operator, when approval is required>"
  },
  "quotaBytes": 52428800
}
```

The client shows the message and the link(s); the user opens one in a browser.
Several links allow mirrors/onion/clearnet alternatives. There is no service
catalogue because nothing is sold here: the only variables are whether an
operator has to approve the account and the one storage quota every user
receives.
(The global `alias` namespace is a separate product sold by the central
alias-resolver, not the service node, and is not specified here.)

server-core points nowhere itself: it relays the service node's refusal unchanged,
and the refusal carries no links. A refused client registers itself
(`POST /v1/account/registration`); when connecting still fails, it reads
`GET /v1/account/portal` and shows the message and links found there.

## 3. Account model

Per user (`subscriptions/<fp>.json` - the account record - and `grants/<fp>.json`):

- **account** - created when the client first publishes its card; holds the
  granted quota, the published card and when a destination was last delegated.
  The account itself has no term - only the delegated transient expires (below):
  `GET /v1/account/i2p-status` reports what the client needs to see, and the node
  deletes an account idle for `idleDeleteDays` (180). Idle counts from the last
  accepted delegation, or from registration for an account that never delegated;
  a revocation is not a delegation and does not restart the count.
- **approval** - `requestedAt` and `approvedAt`. On an open node an account is
  approved the moment it is created; on a moderated one it stays pending (see
  below). Publishing the card again restarts neither `requestedAt` nor the
  registration time the sweep counts from, so a pending account cannot postpone
  its sweep by republishing.
- **transient** - the validity of the offline I2P transient the client last
  delegated, recorded so any of the user's devices knows when to issue the next
  one without holding the expiry itself.

Closing an account tears it down rather than merely letting it lapse: the
destination is revoked and the grants are dropped.

### The user's own I2P destination (key custody + lifecycle)

Every account has its own serving destination - it is part of the base protocol,
not a purchase, and there is no shared pool to fall back on (a shared address
would make every user behind it provably co-located).

The master I2P destination key stays on the **client** (the user's long-term
routing identity, sealed at rest in the account beside the messaging keys). The
client either **generates** a fresh master or **loads** one it already owns from
an unencrypted Ed25519 (signing type 7) `.dat`. The server only ever holds a
short-lived **offline transient** delegated from that master, so the address is
the user's own and stable as they move between servers, and an operator is locked
out once its transient expires.

The destination publishes an **encrypted** LeaseSet2 at its blinded b33, like
every other destination in the project, although the server holds no master key:
the delegation carries, beside the offline transient, one offline signing key per
UTC day, each authorised by that day's blinded key of the master. The messaging
server refuses a delegation without a valid key for the current day.

A destination can only be delegated once the account exists, so the client
publishes its card first, delegates its transient, and immediately re-publishes its
contact card (`POST /v1/account/card` again, which grants nothing and consumes no
grant) with the routing folded in. On a moderated
node the delegation is refused until an operator approves the account, so the
client publishes its routing then instead.

The transient is short: 14 days by default, 1 to 30 as the user sets it (the node
refuses a longer one), kept the same on every device of the account by a
`device.delegation-term` message. The client re-issues a fresh one
about 5 days before it lapses, less up to 6 hours of per-device **jitter**, and
always **polls `GET /v1/account/i2p-status` before issuing** - so when one of the
user's devices has already renewed, the others do not issue one (the multi-device
race).
Keeping the transient fresh is therefore an unconditional background duty of
the client: if it lapses, the destination goes down and senders cannot deliver.

On install/enable the client also sends the master to its **other devices** as a
`device.i2p-master` service message - sealed to the account's own sealing key and
written to the account's own mailbox with `POST /v1/messaging/self` - so every
device adopts the same master (preserving the address) and can re-issue
transients.

The server provisions/serves the destination on its embedded I2P router
(`POST /v1/account/i2p-dest` -> the messaging server's operator-internal
per-user-destination endpoint) and reports its state on
`GET /v1/messaging/destination` (empty while nothing is delegated, then
`building` -> `active` -> `expired`).

> Outside the server: generating the offline transient and the master `.dat`
> is the client's job (vendored libi2pd); the server consumes and serves the
> blob but never generates it.

## 4. Registration: open or moderated

Proving the key is the registration, and the node records a pending grant for
that fingerprint. The grant becomes an account in the store when the client
publishes its card; the client reads its state back via
`GET /v1/account/i2p-status`.

An operator chooses one of two modes (`registration.requireApproval`):

- **open** - the account is servable the moment it is created.
- **approval** - the account is created but **frozen**: its transient is refused
  with `ACCOUNT_PENDING_APPROVAL`, so its destination is never brought up and the
  account can neither send nor receive. The error carries the operator's
  `registration.message`, which states how to contact the operator to complete
  registration. Approval is performed by an operator from the admin panel. An
  account not approved is swept by the same idle sweep as any other, `idleDeleteDays` (default 180) after it was registered - there is no shorter horizon of its own, so an
  abandoned registration does not hold a fingerprint indefinitely.

This mechanism is moderation, not a charge.

| Endpoint | Purpose |
|---|---|
| `POST /v1/account/approve` | Operator-signed: approve one account (the ones awaiting approval are listed by `GET /v1/account/users?state=pending`) |
| `GET /v1/account/users?q&state&limit&offset` | Operator-signed: the accounts on this node, searched and paged here |
| `GET /v1/account/user?user=` | Operator-signed: one account in full |
| `PUT /v1/account/quota` | Operator-signed: set an account's storage quota (and push it to server-core) |
| `POST /v1/account/undelegate` | Operator-signed: revoke the account's destination and put it back on hold, with a message for its owner |
| `DELETE /v1/account/user?user=` | Operator-signed: delete the account (certificate, grants, destination) |
| `POST /v1/account/ban` | Operator-signed: delete the account and refuse this key from now on (reason defaults to "Banned") |
| `POST /v1/account/unban` | Operator-signed: lift a ban |
| `GET /v1/account/settings`, `PUT /v1/account/settings` | Operator-signed, and the whole of what an operator can change: registration mode and its two messages, `loginConsumer`, `portalLinks`, `facades`, `reseeds` (each an `https` URL and not a `.b32.i2p` address, or the write is refused; a config file naming a reseed that is not `https` is refused at start), `defaultQuotaBytes`, `idleDeleteDays`, `logLevel`, and the three addresses `listen`, `portalListen`, `messaging` - plus the account count and the sum of the quotas. Everything is written back into the node's config file, so a restart keeps it; `logLevel` and the portal's own words apply at once, the three addresses at the next start |
| `POST /v1/account/restart` | Operator-signed, on the operator socket: the node exits with the restart code (90) so that its supervisor starts it again |

A ban outlives the account it removed - it is what stops the same key from
registering again - so it is stored apart from the record it deletes, and a
banned fingerprint stays in the listing (`banned: true`) because that is the only
way back to lifting it. A banned key is refused at the portal and at
`POST /v1/account/card`, and the reason the operator wrote is what its owner is
shown.

Settings changed here are applied live and written back to the node's config
file, so what an operator sets in the panel is what the node reads at its next
start. A write that cannot be persisted is refused rather than applied.

`q` is a case-insensitive substring of the fingerprint **or of the account's
destination address**, and `state` is
`all` / `pending` / `approved`; the answer carries the page (`users`) and the
number of matches (`total`). The search belongs here rather than in the panel:
the node is the only side that has the accounts, and sending them all to filter
one would be both slow and a leak of the whole roster into a browser.

## 5. Web frontend (JS-free)

The portal is **a single page** at `/`, server-rendered without JavaScript (forms,
full-page reloads, post-redirect-get). It presents one action at a time; there is
no account area, as the node offers nothing to purchase or manage:

1. **the user's key** - the page renders a challenge; the user signs it in their client
   and pastes the blob back. `POST /login` verifies it and sets the session
   cookie.
There is no second step: the grant is issued as the key is proven, so the page
that follows the sign-in already reports the account - its quota, what the
messaging server holds for it, and the server descriptor to paste into a client -
or, on a moderated node, the "waiting for the operator" notice with the
operator's instructions. An account's owner can also end it from here.
`POST /logout` drops the session.

The portal's **web files are stored on disk**, not in the binary: `portal.assets
<dir>` (mandatory with `portal.listen`) holds one template per state of the
flow, the text catalog `messages.json`, the stylesheet, the logo, the fonts and
an optional `portal.js`. Every word on the page is in those files; the node only
fills in the values a template asks for by name:

| Template | Placeholders |
|---|---|
| `page.html` (shell) | `title`, `strapline` (the line naming this node's messaging-server fingerprint), `session`, `content`, and `lang`, `dir`, `switch` for the page's language |
| `session.html` | `fingerprint` |
| `signin.html` | `challenge`, `error` |
| `registered.html` | `quota`, `descriptor`, `mailbox`, `devices` |
| `registered-approval.html` (moderated node) | the same, plus `message` - the operator's note |
| `waiting.html` | the same as the approval page |
| `banned.html` | `message` - the operator's reason |
| `confirm-delete.html` | `mailbox`, `devices` |

`error` and `message` carry a whole notice or nothing at all, because a
placeholder in a file cannot remove its own box when there is nothing to say.
A placeholder with no value is left in place, so a typo in a template appears
verbatim rather than vanishing.

The script is progressive enhancement only: it adds a copy button to the blocks
meant to be copied, and a browser without scripts loses nothing else. Each file is read once
and served from memory afterwards, so an edit takes effect on restart; only files
in that directory, of a type the portal publishes, are served.

## 6. API relationships

| Caller | Endpoint | Auth | Purpose |
|---|---|---|---|
| Client | `GET /v1/account/portal` | none | onboarding text + portal links + registration terms |
| Client | signs the challenge (local) | key | produce the login blob |
| Browser | `GET /`, `POST /login` | sig -> session | sign-in-with-key, which is the registration |
| Client | `POST /v1/account/registration` | key | the same registration, with no page to visit |
| Panel | `POST /v1/account/approve` | operator sig | moderation |
| Panel | `GET /v1/account/users`, `GET /v1/account/user`, `PUT /v1/account/quota` | operator sig | the operator's account list, search and quota |
| Panel | `POST /v1/account/restart` | operator sig | restart the node, so that a start-only setting takes effect |
| Client | `POST /v1/account/card` | key | publish the card, redeem the grant |
| Client | `POST /v1/account/i2p-dest` | key | upload a signed offline transient (refused while the account is unapproved) |
| Client | `GET /v1/account/i2p-status` | key | approval state and the recorded transient validity |

## 7. What the service node does not do

- **Names.** The global `alias` namespace belongs to the central
  **alias-resolver**, whose own repository documents it. The service node neither
  sells, stores nor resolves names, and it sells nothing at all: a destination
  and a mailbox quota come with the account. There is no payment layer here, not
  even an abstraction.
- **Captcha.** A puzzle a person can read is a puzzle a solver can read, so there
  is none; registration has the two modes above.

## 8. Security notes

- The key never enters the browser; login is a relayed one-shot signature, bound
  to this portal by the consumer and the tag (`SignInWithKey.md`).
- Sessions are short-lived signed cookies.
- Approval and deletion are operator-signed actions on the private channel; the
  portal itself can neither approve an account nor see who is waiting.
