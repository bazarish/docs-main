# Client API

Status: draft skeleton. The single client-facing API, reached through a facade (see `Facade.md`).

## Conventions

- Base: one secret URI per facade; everything below is relative to it.
- Control messages: JSON. Binary content (attachments, encrypted blobs): raw HTTP bodies, no base64.
- Routing by path prefix (performed by the facade):
  - `/v1/messaging/*` → server
  - `/v1/account/*` → service node
- Authentication: every request is signed with **both** identity keys (hybrid, like all identity statements). Canonical string `"v1\n" + timestamp + "\n" + METHOD + "\n" + path + "\n" + hex(sha256(body)) + "\n"`; headers `X-Bazarish-Keys` (base64 of `{"c": b64(EC SPKI), "pq": b64(ML-DSA SPKI)}`), `X-Bazarish-Timestamp`, `X-Bazarish-Sig-Classical`, `X-Bazarish-Sig-Pq`. The server verifies both signatures and both key types and derives the caller UID from the presented keys — no prior key exchange. Freshness window ±300 s; a nonce cache narrowing replay to exactly-once is future hardening.
- The client ID travels in the `X-Bazarish-Client` header.
- All errors follow `../Errors.md`.

## Account (service node)

| Endpoint | Purpose |
|---|---|
| `POST /v1/account/subscribe` | Submit a subscription certificate (`{cert}`). The service node verifies the hybrid signature, that it names this server, is unexpired, and is within the max term; responds `{notAfter, quotaBytes, maxTermSeconds, serverCard}`. The certificate may carry the user's sealing **prekey** (see `../Identity.md`) |
| `POST /v1/account/renew` | Renew the subscription before expiry (submit a fresh certificate; same handler) |
| `GET /v1/account/subscription` | Current subscription state: `{notAfter, quotaBytes}`. (Usage lives on the messaging server.) |
| `DELETE /v1/account/subscription` | Unsubscribe (migration cutover); the server purges all pending content immediately |
| `POST /v1/account/alias` | Register an alias (`{cert}`, signed nickname certificate); `ALIAS_TAKEN` if owned by another user |
| `DELETE /v1/account/alias` | Release an alias (`{alias}`; owner only) |
| `GET /v1/account/resolve?alias=X` | Resolve an alias to `{user, aliasCert, subscriptionCert}` (the messaging server adds its server card). Unauthenticated; `ALIAS_UNKNOWN` is explicit (aliases are findable by design) |
| `GET /v1/account/contact?user=X` | Look up a user by fingerprint: returns `{user, subscriptionCert}`. The certificate carries the serving server and the user's sealing **prekey**, which a first-contact sender uses to E2E-encrypt the contact request. Unauthenticated, like resolve |

Implemented in the `service-node` repository (`SubscriptionStore`, `AccountApi`; covered by tests). The service node is bound to its messaging server by that server's card (passed at startup), from which it takes the server fingerprint subscription certificates must name.

## Messaging (server)

### Sending: submit + poll

Delivery over I2P can take tens of seconds, so sending is asynchronous at the API level while the server keeps **no persistent outbound state** (attempt state is in-memory only):

| Endpoint | Purpose |
|---|---|
| `POST /v1/messaging/send` | Submit a delivery envelope (see `Federation.md`; kinds: `message`, `contact-request`). Early validation (subscription, size), then returns an `attemptId` immediately |
| `GET /v1/messaging/send/{attemptId}` | Poll the attempt: `pending` / `delivered` / `failed` + typed error. Optional `wait` parameter for long-polling. Completed attempts are kept briefly, then dropped |

- Unknown `attemptId` (server restarted, state expired) → the client treats it as failure and resubmits with the **same messageId**; the recipient's server dedups, so retries are idempotent and never consume a second token.
- On failure the message stays only on the client; the server holds no outbound queue.
- A contact accept is not a separate endpoint: it is an ordinary `message` send using a token received in the request (see `../Contacts.md`). Declining a request sends nothing.

| Endpoint | Purpose |
|---|---|
| `POST /v1/messaging/resolve` | Submit a sealed alias-resolution query for relay to a target server (see `Federation.md`); returns an `attemptId` |
| `GET /v1/messaging/resolve/{attemptId}` | Poll the resolution outcome (same attempt semantics as `send`) |

### Receiving

| Endpoint | Purpose |
|---|---|
| `GET /v1/messaging/pending` | List pending entries for this client ID: `{pending: [{id, class}]}` where `class` is the delivery class `content` or `contact` (the admission selector, not the content type — see `../Messages.md`) |
| `GET /v1/messaging/pending/{blobId}` | Fetch the raw E2E payload bytes |
| `POST /v1/messaging/ack` | Confirm receipt for this client ID; when all registered client IDs of the user have acked, the server deletes the content immediately |

### Content store (bulk media, see `../Messages.md`)

| Endpoint | Purpose |
|---|---|
| `POST /v1/content` | Upload opaque ciphertext; returns `{id, size}` where `id = sha256(ciphertext)` hex. Content-addressed and deduplicated; the store never sees the content type or decryption key |
| `GET /v1/content/{id}` | Fetch the ciphertext for a content id |

### Delivery tokens (see `../Contacts.md`)

| Endpoint | Purpose |
|---|---|
| `POST /v1/messaging/tokens` | Register a batch of one-time token hashes for this user's mailbox |
| `DELETE /v1/messaging/tokens` | Delete listed unused hashes (contact revocation or epoch reset) |

### Client registry (see `../DeviceSync.md`)

| Endpoint | Purpose |
|---|---|
| `POST /v1/messaging/clients` | Register this client ID (device enrollment) |
| `DELETE /v1/messaging/clients/{clientId}` | Retire a client ID — own or another (lost) device's. The server re-evaluates ack-completeness of pending content |

Every request carries the client's unique random **client ID**; the "delete when all clients received" rule operates on the registered set. Requests from an unregistered client ID (other than registration itself) fail with `CLIENT_UNREGISTERED`.

## Open questions

- **Incoming delivery mechanism** — long-poll vs websocket vs plain polling on `/pending`. Affects facade transparency.
- **Pagination / batching** for `/pending` with many messages.
- Retention period of completed send attempts before they are dropped.
- Endpoint naming and request/response schemas — to be specified per endpoint once flows are agreed.
