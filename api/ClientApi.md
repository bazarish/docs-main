# Client API

Status: draft skeleton. The single client-facing API, reached through a facade (see `Facade.md`).

## Conventions

- Base: one secret URI per facade; everything below is relative to it.
- Control messages: JSON. Binary content (attachments, encrypted blobs): raw HTTP bodies, no base64.
- **Everything below travels inside one sealed tunnel.** The paths named in this
  document are what the *server* sees after it opens a frame; on the wire there
  is a single `POST /t` carrying an opaque body, and a facade in the middle can
  read none of it (`Facade.md`). The paths still mean what they say - they are
  simply not visible outside:
  - `/v1/messaging/*` - answered by the messaging server itself
  - `/v1/account/*` - forwarded by it to the service node, under its own
    signature, with the user named in the path it signs

### The tunnel

Two kinds of frame, and a client's own identity is presented in exactly one of
them.

- **hello** - opens a tunnel. CMS-sealed to the server's sealing key, which the
  client takes from `GET /v1/server/card`: that card is public, identical for
  everyone, and signed by the server root key whose fingerprint the user already
  had before they saw a facade, so the card can be withheld but not substituted.
  The frame carries a fresh secret, a one-time key for the answer, and the hybrid
  signature that says who is opening it. The answer - the tunnel's lifetime - is
  sealed to that one-time key.
  The server's sealing key **must be hybrid**: sealing needs the ML-KEM half, and
  a server whose key predates hybrid sealing keys cannot be talked to at all. It
  refuses to start rather than let every client sit at "connecting".
- **carry** - everything afterwards. The request rides under AES-256-GCM with a
  key derived from the same secret (a different derivation from the session MAC
  key: one secret, two purposes, and holding one is not holding the other),
  addressed by the per-request handle described below. No asymmetric work per
  request, which matters when a client polls every few seconds.
- **A tunnel cannot open the next tunnel.** That is structural rather than
  checked: a hello is only ever CMS-sealed and signature-verified, and a carried
  frame is only ever opened with a tunnel key. Otherwise a leaked secret could
  renew itself indefinitely and the identity would never be presented again,
  which is exactly what the lifetime below exists to bound.
- **The answer's status rides inside.** Outside it is always `200`, because a
  code visible to the facade would say what the seal was meant to hide.
- **Payloads are padded** to a step of a fixed ladder before they are encrypted.
  Without it the length of a waiting-mail list would still count the mail.
- Authentication, first contact: signed with **both** identity keys (hybrid, like all identity statements). Canonical string `"v1\n" + timestamp + "\n" + METHOD + "\n" + path + "\n" + hex(sha256(body)) + "\n"`, plus `clientId + "\n"` when the request names a device (see the client ID below); headers `X-Bazarish-Keys` (base64 of `{"c": b64(EC SPKI), "pq": b64(ML-DSA SPKI)}`), `X-Bazarish-Timestamp`, `X-Bazarish-Sig-Classical`, `X-Bazarish-Sig-Pq`. The server verifies both signatures and both key types and derives the caller UID from the presented keys - no prior key exchange. Freshness window +/-300 s, narrowed to exactly-once by a server-side replay cache that remembers each accepted request's signature for the window (the randomized classical signature is the per-request nonce, so a legitimate resend is re-signed and accepted while a captured request replayed verbatim is rejected). Measured size of those headers: **~8.3 KB**.
- Authentication, afterwards: a **session**, which is the tunnel. The signature above is presented once, in the hello, and every later request carries `X-Bazarish-Session` (this request's handle), `X-Bazarish-Seq` and `X-Bazarish-Mac` (HMAC-SHA256 over the same canonical string plus `seq + "\n"`) - **~174 bytes**, and an HMAC instead of an ML-DSA verification. Two properties beyond the size:
  - a signature is evidence a server can show a third party that this user made this request at this time; a MAC is not, because the server could have produced it itself;
  - the handle is `HMAC(secret, seq)`, different on every request, so nothing in the middle can tie a session's requests together by it. Nothing is transmitted to arrange this: both sides derive handles, and the server keeps only the next 64 of them indexed (a client runs two connections, so requests overtake each other).
- Sessions expire at a point drawn at random between **1 and 2 hours**: a fixed lifetime would have every client renew on the same schedule, and the short life is what makes a leaked session key worth little. The issue response carries `expiresUnix`, so a client renews ahead of time (5 minutes early) rather than discovering the lapse from a refused request. A lapsed or unknown session answers `SESSION_INVALID`, which the client answers by opening another tunnel and sending the request through it - never by retrying the same way. Three refusals in a row and it stops asking for 15 minutes; a server with no tunnel face is the same case. A session that could not be opened because **nobody answered** is not that case and waits only a minute: the server said nothing about sessions, the network was down, and it is over as soon as it is over.
- The client ID travels in the `X-Bazarish-Client` header, and is **signed with the request** (it is the last line of the canonical string, under both schemes). It has to be: the server picks a device's mail queue by it, so while it was only a header, anything between the two ends could re-label a request as another device of the same account and acknowledge that device's mail away - the message would be dropped before that device ever fetched it. A request that names no device (an operator push, a login blob) signs the string without that line, so those paths are unchanged.
- All errors follow `../Errors.md`.

## Account (service node)

| Endpoint | Purpose |
|---|---|
| `POST /v1/account/card` | Publish the user-signed **contact card** (`{card}`): the sealing key plus the serving **destination** + **serving sealing key** the client fetched from `GET /v1/messaging/destination` (see `../Identity.md`, `InviteAnonymity.md`). The card names no server and carries no validity window. This call is also what turns a portal registration into an account - the first one redeems the grant (`ServicePortal.md`) - and it is repeated whenever the routing changes; a repeat grants nothing. The destination can only be delegated once the account exists, so the first card carries no routing: the client delegates its transient and publishes again straight after. Responds `{quotaBytes, serverCard, approval, registrationMessage}` |
| `DELETE /v1/account/registration` | Terminate the account. The destination is revoked, the node discards the account record and the grants, and the messaging server is instructed to discard the mailbox and all other state held for that fingerprint (`DELETE /v1/messaging/account`, operator-signed). The request is signed with the account's own identity key, the only credential that can authorise it; a client therefore performs this call before removing its local profile and removes it only on success. **Idempotent**: a key for which the node holds nothing receives the same response as a deletion, with nothing revoked or discarded, so that a client whose account an operator has already removed can still complete the operation locally. A client treats any 4xx response likewise: no state for the account remains on the server, and it removes its local copy. Only the absence of a response (or a 5xx) leaves both sides unchanged, with the reason reported to the user |
| `POST /v1/account/i2p-dest` | Upload a signed offline **transient** (`{transient, expiresUnix}`; empty revokes). Every account owns a destination - it is the base protocol, not a purchase - so the only gates are that the account exists here and, on a moderated node, that it is approved (`ACCOUNT_PENDING_APPROVAL` until then). The node hands the transient to the messaging server, which operates the destination, and records its validity |
| `GET /v1/account/i2p-status` | `{approval, registrationMessage, transientExpires, transientUpdatedAt, ...}` - the client polls this to drive transient re-issue and to see whether a moderated account is still awaiting approval |
| `GET /v1/account/card?user=X&view=V` | The contact card behind a descriptor: returns `{user, card}`. `V` is the descriptor's `view` capability (32 hex characters, issued by the node when the account is created and reported with every card publish); a wrong one is refused **identically** to an unknown user, so this cannot be used to ask whether a server hosts a fingerprint. Reached from the messaging server, which serves a peer's query on the user's own destination - there is no lookup by fingerprint alone. The `username` namespace lives on the central resolver (`api/AliasResolver.md`, `api/FederatedResolve.md`) |

Implemented in the `service-node` repository (`SubscriptionStore`, `AccountApi`; covered by tests). The service node is bound to its messaging server by that server's card (passed at startup). Nothing a contact can fetch names that server.

## Messaging (server)

### Sending

**There is no send endpoint.** A client delivers its own outgoing messages: it
dials the recipient's serving destination over I2P and performs the federation
exchange with it (`Federation.md`), from a destination held for that one
correspondent. The account's own server is never asked to dial a correspondent and
therefore does not learn who its users write to; this is the reason the endpoint
was removed rather than left unused.

The server performs no part of a delivery. Its remaining functions are receipt -
it operates the destination a correspondent dials, admits the envelope and holds
it in the mailbox until the recipient's devices have collected it - and the single
write a device makes to its own account, `POST /v1/messaging/self`, which reaches
no other party.

A **contact request**, which presents nothing, is capped by the protocol at **17890 sealed
bytes** (`kMaxContactRequestBytes`) - not by any operator setting, and not with
room to spare: this is the one thing a stranger may put in a mailbox, so the cap
is the measured worst case and nothing above it.

The worst case is **17858 bytes**, measured through the real path: the two hybrid
sealing keys the bootstrap carries, the delivery pass the requester hands
over so the peer can answer, the ML-KEM ciphertext of the seal, the authorship
block with the sender's keys in it (a request has no card and no pass, so the
signature is the only thing naming them), the longest greeting a user may write
and the longest name an account may carry. The 32 bytes on top are what the
variable-length parts can add - an ECDSA signature is 70 to 72 bytes, a CBOR
integer is as wide as its value.

Two client-side caps are what make that number a number: the greeting is at most
**100 bytes** (`kMaxContactGreetingBytes`) and an account's own display name at
most **64 bytes** (`kMaxAccountNameBytes`, about 32 characters of a non-Latin
alphabet). Both are refused where they are typed rather than at the recipient's
server.

- A contact acceptance is not a separate endpoint: it is an ordinary content
  message presenting the pass received in the request (see `../Contacts.md`).
  Declining a request transmits nothing.
- A repeated delivery is the same delivery: the delivery id is derived from the
  message and the recipient mailbox (`../Messages.md`), so the recipient's server
  recognises the repetition and stores no second copy.

### Own destination

| Endpoint | Purpose |
|---|---|
| `GET /v1/messaging/destination` | Returns the caller's assigned `{dest, servingKey, state}` - the destination this server operates for the user (its standard-LeaseSet2 b32, equal to the master b32 the client holds), the public serving sealing key for it, and `state` (empty when the user delegated none, else `building` -> `active` -> `expired`). `dest` is empty until the destination publishes, so a client that has just delegated fills the card with its own master b32 - the same address either way. The client folds dest+key into the contact card it signs |
| `GET /v1/messaging/storage-usage` | The authenticated caller's own mailbox storage: `{usedBytes, quotaBytes}` - bytes of pending content currently held, and the effective quota (the user's override, else the server default). Read-only; the client shows it as used/free/total in per-account settings. |

### Receiving

| Endpoint | Purpose |
|---|---|
| `GET /v1/messaging/events` | Hold the request open until something is pending for this client (or `wait` seconds pass, capped at 30), then answer in the same shape as `pending`. How a server holds it open is its own business; what the protocol requires is that a delivery ends the wait. A client that gets a 404 here falls back to polling `pending` |
| `GET /v1/messaging/pending` | List pending entries for this client ID: `{pending: [{id, class}]}` where `class` is the delivery class `content` or `contact` (the admission selector, not the content type - see `../Messages.md`) |
| `GET /v1/messaging/pending/{blobId}` | Fetch the raw E2E payload bytes |
| `POST /v1/messaging/ack` | Confirm receipt for this client ID; when all registered client IDs of the user have acked, the server deletes the content immediately |

### Files (see `FileTransfer.md`)

There is no storage service and no upload endpoint. A file is announced in the
message and transferred directly between the two clients over a one-time I2P
destination when the recipient asks for it.

### Reseed

No call in this API is made over clearnet: every one of them goes to a facade
over I2P. A server does publish one thing for a client that has no transport yet,
and it is not an API:

| Where | What |
|---|---|
| `GET <portal>/i2pseeds.su3` | This server's netDb as an **su3 reseed archive** - the ordinary I2P format, under the name that format gives it, read by any router. Unauthenticated by necessity (a client fetching its first netDb holds nothing yet) and safe by construction (every RouterInfo inside carries its own router's signature). Packed from a random sample of about sixty routers and rebuilt no more than once an hour: one archive for everyone who asks within that hour, so repeated asking learns no more of the netDb than a single one |

It is published by the **portal**, not by a facade, and that is not an accident:
a facade is an I2P address, and a client needs this file *before* it has I2P. The
portal is on the clearnet anyway.

A server need not publish it at all. What a client is given is a reseed
**address** - a base URL, exactly as for any I2P reseed - and the router appends
`i2pseeds.su3` to it. So the address may name this portal or any other reseed
anywhere, and no client or router needs a special case to read it.

A server descriptor may name **reseed addresses**, and each is a full `https://`
URL to an **su3 archive** - the ordinary I2P reseed format. Nothing Bazarish is
spoken there: the client's own I2P engine fetches the archive, unpacks it and
loads the routers, exactly as it would from any reseed. What the addresses buy is
that a client's first start does not have to reach the public reseed hosts its
router carries, and that the operator may publish them anywhere at all, which is
what keeps the bootstrap from pointing at the server (`Facade.md`).

With no reseed address in the descriptor, the router uses the public hosts it
carries. That is the fallback, and the only case in which the bootstrap leaves
the network the user chose.

### Delivery passes (see `../Contacts.md`)

| Endpoint | Purpose |
|---|---|
| `POST /v1/messaging/passes` | Register the hashes of the passes this user's mailbox admits. Hashes, not the passes themselves: a sender presents the pass, so what is on the node's disk recognises a delivery and cannot produce one. Registering one already held is how a block is lifted |
| `POST /v1/messaging/passes/revoke` | Drop exactly these passes and answer how many were there. Exact and immediate - the account names what it issued, so there is nothing to sweep for (`../Contacts.md`) |

### Client registry (see `../DeviceSync.md`)

| Endpoint | Purpose |
|---|---|
| `POST /v1/messaging/clients` | Register this client ID (device enrollment) |
| `DELETE /v1/messaging/clients/{clientId}` | Retire a client ID - own or another (lost) device's. The server re-evaluates ack-completeness of pending content |

Every request carries the client's unique random **client ID**; the "delete when all clients received" rule operates on the registered set. Requests from an unregistered client ID (other than registration itself) fail with `CLIENT_UNREGISTERED`.

## Open questions

- **Incoming delivery mechanism** - long-poll vs websocket vs plain polling on `/pending`. Affects facade transparency.
- **Pagination / batching** for `/pending` with many messages.
- Retention period of completed send attempts before they are dropped.
- Endpoint naming and request/response schemas - to be specified per endpoint once flows are agreed.
