# Client API

Status: draft skeleton. The single client-facing API, reached through a facade (see `Facade.md`).

## Conventions

- Base: one secret URI per facade; everything below is relative to it.
- Control messages: JSON. Binary content (attachments, encrypted blobs): raw HTTP bodies, no base64.
- Routing by path prefix (performed by the facade):
  - `/v1/messaging/*` -> server
  - `/v1/auth/*` -> server (a session is sealed to a key that server holds)
  - `/v1/account/*` -> service node
- Authentication, first contact: signed with **both** identity keys (hybrid, like all identity statements). Canonical string `"v1\n" + timestamp + "\n" + METHOD + "\n" + path + "\n" + hex(sha256(body)) + "\n"`; headers `X-Bazarish-Keys` (base64 of `{"c": b64(EC SPKI), "pq": b64(ML-DSA SPKI)}`), `X-Bazarish-Timestamp`, `X-Bazarish-Sig-Classical`, `X-Bazarish-Sig-Pq`. The server verifies both signatures and both key types and derives the caller UID from the presented keys - no prior key exchange. Freshness window +/-300 s, narrowed to exactly-once by a server-side replay cache that remembers each accepted request's signature for the window (the randomized classical signature is the per-request nonce, so a legitimate resend is re-signed and accepted while a captured request replayed verbatim is rejected). Measured size of those headers: **~8.3 KB**.
- Authentication, afterwards: a **session**. The signature above is presented once to `POST /v1/auth/session`, and every later request carries `X-Bazarish-Session` (this request's handle), `X-Bazarish-Seq` and `X-Bazarish-Mac` (HMAC-SHA256 over the same canonical string plus `seq + "\n"`) - **~174 bytes**, and an HMAC instead of an ML-DSA verification. Two properties beyond the size:
  - a signature is evidence a server can show a third party that this user made this request at this time; a MAC is not, because the server could have produced it itself;
  - the handle is `HMAC(secret, seq)`, different on every request, so the facade in the middle cannot tie a session's requests together. Nothing is transmitted to arrange this: both sides derive handles, and the server keeps only the next 64 of them indexed (a client runs two connections, so requests overtake each other).
- Sessions expire at a point drawn at random between **1 and 2 hours**: a fixed lifetime would have every client renew on the same schedule, and the short life is what makes a leaked session key worth little. The issue response carries `expiresUnix`, so a client renews ahead of time (5 minutes early) rather than discovering the lapse from a refused request. A lapsed or unknown session answers `SESSION_INVALID`, which the client answers by signing that one request and opening a session for the next - never by retrying the same way. Three refusals in a row and it stops asking for 15 minutes; a server with no session face, or an account with no serving key to seal to, is the same case.
- The client ID travels in the `X-Bazarish-Client` header.
- All errors follow `../Errors.md`.

## Account (service node)

| Endpoint | Purpose |
|---|---|
| `POST /v1/account/card` | Publish the user-signed **contact card** (`{card}`): the sealing **prekey** plus the serving **destination** + **serving sealing key** the client fetched from `GET /v1/messaging/destination` (see `../Identity.md`, `InviteAnonymity.md`). The card names no server and carries no validity window. This call is also what turns a portal registration into an account - the first one redeems the grant (`ServicePortal.md`) - and it is repeated whenever the routing changes; a repeat grants nothing. The destination can only be delegated once the account exists, so the first card carries no routing: the client delegates its transient and publishes again straight after. Responds `{quotaBytes, serverCard, approval, registrationMessage}` |
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

A **tokenless contact request** is capped by the protocol at **11700 sealed
bytes** (`kMaxContactRequestBytes`) - not by any operator setting, and not with
room to spare: this is the one thing a stranger may put in a mailbox, so the cap
is the measured worst case and nothing above it.

The worst case is **11668 bytes**, measured through the real path: the two hybrid
sealing keys the bootstrap carries, the 16-token reply batch the requester hands
over so the peer can answer, the ML-KEM ciphertext of the seal, the authorship
block with the sender's keys in it (a request has no card and no token, so the
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
  message that spends a token received in the request (see `../Contacts.md`).
  Declining a request transmits nothing.
- A repeated delivery is the same delivery: the delivery id is derived from the
  message and the recipient mailbox (`../Messages.md`), so the recipient's server
  recognises the repetition, stores no second copy and consumes no second token.

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

### Private reseed

| Endpoint | Purpose |
|---|---|
| `GET /v1/messaging/reseed` | A random slice of the server's netDb as `{routers: [<base64 RouterInfo>]}`, so a client's first I2P start never reaches a public reseed host. Unauthenticated on purpose - the client has no transport yet - and safe, because every RouterInfo carries its own router's signature |

The reseed is the one request that goes to a **reseed** address - the clearnet
half of the server descriptor - and it is the only thing those addresses serve.
It bootstraps the transport every other request needs, and it is reached under
that address's secret prefix (`Facade.md`), which the client already holds
because the prefix is part of the URL in the descriptor. If it cannot be fetched -
no reseed in the descriptor, or none answered - the client falls back to whatever
public reseed hosts its router knows, and only then. Everything else the client asks for goes to
a facade, over I2P. Being unauthenticated, that prefix is
the request's only gate: a facade published at a bare root would answer a reseed -
and so identify itself as Bazarish - to any scanner.

### Delivery tokens (see `../Contacts.md`)

| Endpoint | Purpose |
|---|---|
| `POST /v1/messaging/tokens` | Register a batch of one-time tokens for this user's mailbox. The tokens themselves, not their hashes: revoking a correspondent's tokens (below) is a sweep over their shape |
| `POST /v1/messaging/tokens/revoke` | Drop every token this user holds that was minted under the given mask - one correspondent's, all of them. Answers before the sweep rather than after it (`../Contacts.md`) |

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
