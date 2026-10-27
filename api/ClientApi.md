# Client API

The single client-facing API, reached through a facade (see `Facade.md`).

## Conventions

- Base: a facade's address, answered at `/`; everything below is relative to it.
- Control messages: JSON. Binary content (attachments, encrypted blobs): raw HTTP bodies, no base64.
- **Everything below travels inside one sealed tunnel.** The paths named in this
  document are what the *server* sees after it opens a frame; on the wire there
  is a single `POST /t` carrying an opaque body, and a facade in the middle can
  read none of it (`Facade.md`). The paths keep their meaning - they are not
  visible outside:
  - `/v1/messaging/*` - answered by the messaging server itself
  - `/v1/account/*` - forwarded by it to the service node, under its own
    signature, with the user named in the path it signs

### The tunnel

Two kinds of frame, and a client's own identity is presented in exactly one of
them.

- **hello** - opens a tunnel. Hybrid-sealed to the server's sealing key, which
  the client takes from `GET /v1/server/card`: that card is public, identical
  for everyone, and signed by the server root key whose fingerprint the user
  already had before they saw a facade, so the card can be withheld but not
  substituted. The frame carries a fresh secret, a one-time key for the answer,
  and the hybrid signature that says who is opening it. The answer - the
  tunnel's lifetime - is sealed to that one-time key.
  The server's sealing key **must be hybrid**: sealing needs the ML-KEM half, and
  no tunnel can be opened to a server whose key predates hybrid sealing keys. Such
  a server refuses to start rather than leave every client in "connecting".
- **carry** - everything afterwards. The request is encrypted under AES-256-GCM
  with a key derived from the same secret (a different derivation from the session
  MAC key: one secret, two purposes, and holding one is not holding the other),
  addressed by the per-request handle described below. No asymmetric work per
  request, which matters over a session's life: the held-open `events` request
  alone is sent again each time it is answered, at most 30 seconds apart, and
  every fetch and acknowledgement is a request of its own.
- **A tunnel cannot open the next tunnel.** That is structural rather than
  checked: a hello is only hybrid-sealed and signature-verified, and a carried
  frame is only opened with a tunnel key. Otherwise a leaked secret
  could renew itself indefinitely and the identity would never be presented
  again, which is exactly what the lifetime below exists to bound.
- **The answer's status is carried inside.** Outside it is always `200`, because
  a code visible to the facade would say what the seal was meant to hide.
- **Payloads are padded** to a step of a fixed ladder before they are encrypted.
  Without it the length of a waiting-mail list would still reveal the number of
  messages.
- Authentication, first contact: a **signed request** with both identity keys (`SignInWithKey.md`, section 1), the device line included when the request names a device. The server derives the caller UID from the presented keys - no prior key exchange - and remembers every accepted nonce for the freshness window, so a captured request replayed verbatim is rejected.
- Authentication, afterwards: a **session**, which is the tunnel. The signature above is presented once, in the hello, and every later request carries `X-Bazarish-Session` (this request's handle), `X-Bazarish-Seq` and `X-Bazarish-Mac` (HMAC-SHA256 over the base string of `SignInWithKey.md` section 1 - no nonce line - plus `seq + "\n"`) - **~174 bytes**, and an HMAC instead of an ML-DSA verification. Two properties beyond the size:
  - a signature is evidence a server can show a third party that this user made this request at this time; a MAC is not, because the server could have produced it itself;
  - the handle is `HMAC(secret, seq)`, different on every request, so nothing in the middle can tie a session's requests together by it. Nothing is transmitted to arrange this: both sides derive handles, and the server keeps only the next 64 of them indexed (a client runs two connections, so requests overtake each other).
- Sessions expire at a point drawn at random between **1 and 2 hours**: a fixed lifetime would have every client renew on the same schedule, and the short life is what makes a leaked session key worth little. The issue response carries `expiresUnix`, so a client renews ahead of time (5 minutes early) rather than discovering the lapse from a refused request. A lapsed or unknown session answers `SESSION_INVALID`, which the client answers by opening another tunnel and sending the request through it - never by retrying the same way. After three consecutive refusals it stops requesting for 15 minutes; a server with no tunnel face is the same case. A session that could not be opened because **no answer arrived** is not that case and waits only a minute: the server said nothing about sessions, the network was down, and the outage may end at any moment.
- The client ID travels in the `X-Bazarish-Client` header, and is **signed with the request** (it is a line of the base string, so both schemes cover it). It has to be: the server picks a device's mail queue by it, so while it was only a header, anything between the two ends could re-label a request as another device of the same account and acknowledge that device's mail - the message would be dropped before that device ever fetched it. A request that names no device (an operator push, a login blob) signs the string without that line, so those paths are unchanged.
- All errors follow `../Errors.md`.

## Account (service node)

| Endpoint | Purpose |
|---|---|
| `POST /v1/account/card` | Publish the user-signed **contact card** (`{card}`): the sealing key plus the serving **destination** + **serving sealing key** the client received from `POST /v1/account/i2p-dest` or reads back from `GET /v1/messaging/destination` (see `../Identity.md`, `InviteAnonymity.md`). The card names no server and carries no validity window. This call is also what turns a portal registration into an account - the first one redeems the grant (`ServicePortal.md`) - and it is repeated whenever the routing changes; a repeat grants nothing. The destination can only be delegated once the account exists, so the first card carries no routing: the client delegates its transient and publishes again immediately afterwards. Responds `{quotaBytes, serverCard, approval, registrationMessage}` |
| `DELETE /v1/account/registration` | Terminate the account. The destination is revoked, the node discards the account record and the grants, and the messaging server is instructed to discard the mailbox and all other state held for that fingerprint (`DELETE /v1/messaging/account`, operator-signed). The request is signed with the account's own identity key, the only credential that can authorise it; a client therefore performs this call before removing its local profile and removes it only on success. **Idempotent**: a key for which the node holds nothing receives the same response as a deletion, with nothing revoked or discarded, so that a client whose account an operator has already removed can still complete the operation locally. A client treats any 4xx response likewise: no state for the account remains on the server, and it removes its local copy. Only the absence of a response (or a 5xx) leaves both sides unchanged, with the reason reported to the user |
| `POST /v1/account/i2p-dest` | Upload a signed offline **transient** (`{transient, expiresUnix}`; empty takes the address offline and keeps its keys). Every account owns a destination - it is the base protocol, not a purchase - so the only gates are that the account exists here and, on a moderated node, that it is approved (`ACCOUNT_PENDING_APPROVAL` until then). The node passes the transient to the messaging server, which operates the destination, and records its validity. Responds `{dest, servingKey}`: the address this delegation gives and the base64 SPKI of the key that destination seals deliveries with. The client needs both to issue its card, and the serving key is the half it cannot derive itself - so the card is published from this answer rather than from a second request. A transient over a new address also means the old destination is dropped at once (`ServerServiceNode.md`), which is what makes the fan-out that follows obligatory (`../Messages.md`) |
| `GET /v1/account/i2p-status` | `{approval, registrationMessage, transientExpires, transientUpdatedAt}`, where `approval` is `none`, `pending` or `approved` - the client polls this to drive transient re-issue and to see whether a moderated account is still awaiting approval |
| `POST /v1/account/registration` | Register with no portal page to visit (`ServicePortal.md`): answers `{ok, quotaBytes, requiresApproval, message}` |
| `GET /v1/account/portal` | The node's onboarding text and registration terms: `{message, links, registration: {requiresApproval, message}, quotaBytes}` |
| `GET /v1/account/card?user=X` | The contact card behind a descriptor: returns `{user, card}`. Reached from the messaging server, which serves a peer's query **on the user's own destination** - that destination is the capability, and it answers for its own user alone, so this cannot be used to ask whether a server hosts some other fingerprint. There is no lookup by fingerprint through a facade. The `alias` namespace lives on the central resolver (`api/FederatedResolve.md`) |

The service node is bound to its messaging server by that server's card (passed at startup). Nothing a contact can fetch names that server.

## Messaging (server)

### Sending

**There is no send endpoint.** A client delivers its own outgoing messages: it
dials the recipient's serving destination over I2P and performs the federation
exchange with it (`Federation.md`), from a destination held for that one
correspondent. The account's own server is never asked to dial a correspondent and
therefore does not learn who its users write to. There are no send attempts to
keep: the account's server holds no outgoing state, and a client holds a delivery
in memory only until its outcome is reported.

The server performs no part of a delivery. Its remaining functions are receipt -
it operates the destination a correspondent dials, admits the envelope and holds
it in the mailbox until the recipient's devices have collected it - and the single
write a device makes to its own account, `POST /v1/messaging/self`, which reaches
no other party.

A **contact request**, which presents nothing, is capped by the protocol at **17590 sealed
bytes** (`kMaxContactRequestBytes`) - not by any operator setting, and with
almost no margin: this is the one thing a stranger may put in a mailbox, so the
cap sits just above the measured worst case.

The worst case is **17558 bytes**, measured through the real path: the two hybrid
sealing keys the bootstrap carries, the delivery pass the requester provides
so the peer can answer, the ML-KEM ciphertext of the seal, the authorship
block with the sender's keys in it (a request has no card and no pass, so the
signature is the only thing naming them), the longest greeting a user may write
and the longest name an account may carry. The body is padded to the 16384-byte
step of the padding ladder before it is sealed, and the seal frame adds 1174
bytes, so every body up to 16380 bytes seals to exactly 17558: the
variable-length parts move nothing until the body outgrows that step. The cap
leaves 32 bytes over that size.

Two client-side caps are what keep the body inside that step: the greeting is
at most **100 bytes** (`kMaxContactGreetingBytes`) and an account's own display
name at most **64 bytes** (`kMaxAccountNameBytes`, about 32 characters of a
non-Latin alphabet). A value over either cap is refused where it is typed rather
than at the recipient's server.

- A contact acceptance is not a separate endpoint: it is an ordinary content
  message presenting the pass received in the request (see `../Contacts.md`).
  Declining a request transmits nothing.
- A repeated delivery is the same delivery: the delivery id is derived from the
  message and the recipient mailbox (`../Messages.md`), so the recipient's server
  recognises the repetition and stores no second copy. It remembers a delivery id for 30 days after storing it, so a repetition within that time is answered as delivered.

### Own destination

| Endpoint | Purpose |
|---|---|
| `GET /v1/messaging/destination` | Returns the caller's assigned `{dest, servingKey, state}` - the destination this server operates for the user (the blinded b33 of its encrypted LeaseSet2, a `.b32.i2p` host equal to the b33 of the master the client holds), the public serving sealing key for it, and `state` (`none` when the user delegated none, else `building` -> `active` -> `expired`). `dest` is empty only while nothing is delegated - the server derives it from the delegation at once, before the destination publishes - and a client refreshing its card then fills in its own master's b33, the same address either way. The client includes dest+key in the contact card it signs |
| `GET /v1/messaging/storage-usage` | The authenticated caller's own mailbox storage: `{usedBytes, quotaBytes}` - bytes of pending content currently held, and the effective quota (the user's override, else the server default). Read-only; the client shows it as used/free/total in per-account settings. |

### Receiving

| Endpoint | Purpose |
|---|---|
| `GET /v1/messaging/events` | Hold the request open until something is pending for this client (or `wait` seconds pass, capped at 30), then answer in the same shape as `pending`. How a server holds it open is an implementation matter; what the protocol requires is that a delivery ends the wait, at once. This is the **only** way a client learns that mail has arrived: there is no second, slower mechanism to fall back to, and a wait that ends in an error is repeated rather than delegated to one |
| `GET /v1/messaging/pending` | List pending entries for this client ID: `{pending: [{id, class}]}` where `class` is the delivery class `content`, `contact` or `device` (the admission selector, not the content type - see `../Messages.md`). **Oldest arrival first**, and `events` answers in the same order: a burst is fetched in the order the mailbox took it, not in whatever order the server's storage enumerates. The order is arrival only - the position of each message in the conversation is decided by the `sentAt` inside the sealed envelope, which no server sees. The list is not paged: one answer lists every entry this client has not acknowledged, and the mailbox quota bounds its length. A client takes in as many entries per pass as it chooses and acknowledges them; while any remain unacknowledged, `events` answers at once |
| `GET /v1/messaging/pending/{blobId}` | Fetch the raw E2E payload bytes |
| `POST /v1/messaging/ack` | Confirm receipt of `{blobId}` for this client ID; when all registered client IDs of the user have acked, the server deletes the content immediately |

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
| `GET <portal>/i2pseeds.su3` | This server's netDb as an **su3 reseed archive** - the ordinary I2P format, under the name that format gives it, read by any router. Unauthenticated by necessity (a client fetching its first netDb holds nothing yet) and safe by construction (every RouterInfo inside carries its own router's signature). Packed from a random sample of about sixty routers and rebuilt no more than once an hour: one archive for everyone who asks within that hour, so repeated requests learn no more of the netDb than a single one |

It is published by the **portal**, not by a facade, and that is deliberate:
a facade is an I2P address, and a client needs this file *before* it has I2P. The
portal is already on the clearnet.

A server need not publish it at all. What a client is given is a reseed
**address** - a base URL, exactly as for any I2P reseed - and the router appends
`i2pseeds.su3` to it. So the address may name this portal or any other reseed
anywhere, and no client or router needs a special case to read it.

A server descriptor may name **reseed addresses** of that form. Nothing Bazarish
is spoken there: the client's own I2P engine fetches the archive, unpacks it and
loads the routers, exactly as it would from any reseed. What the addresses provide is
that a client's first start does not have to reach the public reseed hosts its
router carries, and that the operator may publish them anywhere at all, which is
what keeps the bootstrap from pointing at the server (`Facade.md`).

With no reseed address in the descriptor, the router uses the public hosts it
carries. That is the fallback, and the only case in which the bootstrap leaves
the network the user chose.

### Delivery passes (see `../Contacts.md`)

| Endpoint | Purpose |
|---|---|
| `POST /v1/messaging/passes` | `{passes: [<hash>, ...]}` -> `{ok, held}`. Register the hashes of the passes this user's mailbox admits. Hashes, not the passes themselves: a sender presents the pass, so what is on the node's disk recognises a delivery and cannot produce one. Registering one already held is how a block is lifted |
| `POST /v1/messaging/passes/revoke` | `{passes}` -> `{revoked}`. Drop exactly these passes and answer how many were there. Exact and immediate - the account names what it issued, so there is nothing to sweep for (`../Contacts.md`) |

### Client registry (see `../DeviceSync.md`)

| Endpoint | Purpose |
|---|---|
| `POST /v1/messaging/clients` | Register this client ID, `{clientId}` (device enrollment) |
| `GET /v1/messaging/clients` | The registered client IDs: `{clients: [{clientId, current, queue}]}` - `current` marks the caller, `queue` counts the entries waiting for that client |
| `DELETE /v1/messaging/clients/{clientId}` | Retire a client ID - own or another (lost) device's. The server re-evaluates ack-completeness of pending content |
| `POST /v1/messaging/self` | Store `{deliveryId, payload}` (payload base64) in the account's own mailbox for its other devices: answers `{ok, stored}`, and `stored` is false while the account has one registered client (`../DeviceSync.md`) |

Every request carries the client's unique random **client ID**; the "delete when all clients received" rule operates on the registered set. Requests to `events`, `pending`, `ack`, `passes` and `self` from an unregistered client ID fail with `CLIENT_UNREGISTERED`. A successful write that has nothing to report answers `{ok: true}`.
