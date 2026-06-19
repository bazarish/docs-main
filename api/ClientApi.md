# Client API

Status: draft skeleton. The single client-facing API, reached through a facade (see `Facade.md`).

## Conventions

- Base: one secret URI per facade; everything below is relative to it.
- Control messages: JSON. Binary content (attachments, encrypted blobs): raw HTTP bodies, no base64.
- Routing by path prefix (performed by the facade):
  - `/v1/messaging/*` -> server
  - `/v1/account/*` -> service node
- Authentication: every request is signed with **both** identity keys (hybrid, like all identity statements). Canonical string `"v1\n" + timestamp + "\n" + METHOD + "\n" + path + "\n" + hex(sha256(body)) + "\n"`; headers `X-Bazarish-Keys` (base64 of `{"c": b64(EC SPKI), "pq": b64(ML-DSA SPKI)}`), `X-Bazarish-Timestamp`, `X-Bazarish-Sig-Classical`, `X-Bazarish-Sig-Pq`. The server verifies both signatures and both key types and derives the caller UID from the presented keys - no prior key exchange. Freshness window +/-300 s; a nonce cache narrowing replay to exactly-once is future hardening.
- The client ID travels in the `X-Bazarish-Client` header.
- All errors follow `../Errors.md`.

## Account (service node)

| Endpoint | Purpose |
|---|---|
| `POST /v1/account/subscribe` | Submit a subscription certificate (`{cert}`). The service node verifies the hybrid signature, that it names this server, is unexpired, and is within the max term; responds `{notAfter, quotaBytes, maxTermSeconds}`. The certificate is the user's **contact card** - it carries the sealing **prekey** plus the serving **destination** + **serving sealing key** the client fetched from `GET /v1/messaging/destination` (see `../Identity.md`, `InviteAnonymity.md`). Registration is grant-gated through the portal (`ServicePortal.md`) |
| `POST /v1/account/renew` | Renew the subscription before expiry (submit a fresh certificate; same handler) |
| `GET /v1/account/subscription` | Current subscription state: `{notAfter, quotaBytes}`. (Usage lives on the messaging server.) |
| `DELETE /v1/account/subscription` | Unsubscribe (migration cutover); the server purges all pending content immediately |
| `POST /v1/account/i2p-dest/setting` | Turn the **per-user i2p-dest** option on/off (`{enabled}`). On charges a term (or resumes if still paid; `402` on an insufficient balance); off stops auto-renew and revokes the personal destination (fall back to the pool). See `ServicePortal.md` |
| `POST /v1/account/i2p-dest` | Upload a signed offline **transient** (`{transient, expiresUnix}`; empty revokes). Gated by an active i2pDest entitlement; the server hands it to the messaging server, which operates the personal destination, and records its validity |
| `GET /v1/account/i2p-status` | `{enabled, active, paidThrough, projectedShutoff, transientExpires, transientUpdatedAt, storageQuotaBytes, storageActive, storageProjectedShutoff, balanceAtomic, currency}` - the client polls this to drive transient re-issue and to show the user when their dest/storage will lapse from the balance |
| `GET /v1/account/contact?user=X` | Look up a user by fingerprint: returns `{user, subscriptionCert}`. The certificate is the contact card - the user's sealing **prekey**, serving **destination** and **serving sealing key**, which a first-contact sender uses to route to and E2E-encrypt the contact request. Unauthenticated (a user must be reachable to be added). The `username` namespace lives on the central resolver, not the service node (`api/AliasResolver.md`, `api/FederatedResolve.md`) |

Implemented in the `service-node` repository (`SubscriptionStore`, `AccountApi`; covered by tests). The service node is bound to its messaging server by that server's card (passed at startup), from which it takes the server fingerprint subscription certificates must name.

## Messaging (server)

### Sending: submit + poll

Delivery over I2P can take tens of seconds, so sending is asynchronous at the API level while the server keeps **no persistent outbound state** (attempt state is in-memory only):

| Endpoint | Purpose |
|---|---|
| `POST /v1/messaging/send` | Submit a delivery envelope (see `Federation.md`; kinds: `message`, `contact-request`). Early validation (subscription, size), then returns an `attemptId` immediately |
| `GET /v1/messaging/send/{attemptId}` | Poll the attempt: `pending` / `delivered` / `failed` + typed error. Optional `wait` parameter for long-polling. Completed attempts are kept briefly, then dropped |

- Unknown `attemptId` (server restarted, state expired) -> the client treats it as failure and resubmits with the **same messageId**; the recipient's server dedups, so retries are idempotent and never consume a second token.
- On failure the message stays only on the client; the server holds no outbound queue.
- A contact accept is not a separate endpoint: it is an ordinary `message` send using a token received in the request (see `../Contacts.md`). Declining a request sends nothing.

### Own destination

| Endpoint | Purpose |
|---|---|
| `GET /v1/messaging/destination` | Returns the caller's assigned `{dest, servingKey, state}` - the serving destination this server routes to the user (its b32; a paid offline-key personal destination is a standard-LeaseSet2 b32), the public serving sealing key for it, and the serving `state` (`home` / `pool` / for a personal destination `building` -> `active` -> `expired`) so the client only publishes a personal address once it is `active`. The client folds dest+key into its self-signed subscription certificate (its contact card) |

### Federation fetch relay

| Endpoint | Purpose |
|---|---|
| `POST /v1/messaging/fetch` | Relay a sealed fetch frame (`{op, sealed}`; `op` is `card` or `resolve`) to a target destination over I2P and return the sealed reply synchronously (see `Federation.md`, `FederatedResolve.md`). This is the own-server proxy fallback for clients with no local SAM bridge; the preferred path is a direct transient-SAM fetch |

### Receiving

| Endpoint | Purpose |
|---|---|
| `GET /v1/messaging/pending` | List pending entries for this client ID: `{pending: [{id, class}]}` where `class` is the delivery class `content` or `contact` (the admission selector, not the content type - see `../Messages.md`) |
| `GET /v1/messaging/pending/{blobId}` | Fetch the raw E2E payload bytes |
| `POST /v1/messaging/ack` | Confirm receipt for this client ID; when all registered client IDs of the user have acked, the server deletes the content immediately |

### Large media: blob storage (see `LargeFiles.md`)

Large content (files, and any message blob over 512 KB) is not stored on the
messaging server. The client uploads ciphertext to the **blob store** and the
message carries only a small sealed pointer; the recipient fetches it over I2P.

| Endpoint | Purpose |
|---|---|
| `PUT /v1/storage/blob` | Upload blob ciphertext (authed; forwarded by the facade to the blob store). Returns `{blobUrl, blobId, deleteToken}`. Retention rides in `X-Blob-Ttl`/`X-Blob-Count`/`X-Blob-Sha256` headers |
| `GET /v1/messaging/blob-proxy?host=&path=` | Own-server I2P proxy fetch of a blob's `.b32.i2p` host+path, for clients without a local SAM bridge (the direct path is a transient SAM session straight to the blob's b33) |

### Delivery tokens (see `../Contacts.md`)

| Endpoint | Purpose |
|---|---|
| `POST /v1/messaging/tokens` | Register a batch of one-time token hashes for this user's mailbox |
| `DELETE /v1/messaging/tokens` | Delete listed unused hashes (contact revocation or epoch reset) |

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
