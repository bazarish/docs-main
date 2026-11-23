# Client API

The single client-facing API, reached through a facade (see `Facade.md`).

## Conventions

- Base: a facade's address, answered at `/`; everything below is relative to it.
- Control messages: JSON. Binary content (attachments, encrypted blobs): raw HTTP bodies, no base64.
- **Everything below travels inside a sealed tunnel, and there are two.** The
  paths named in this document are what a backend sees after it opens a frame; on
  the wire there is `POST /t` or `POST /n` carrying an opaque body, and a facade
  in the middle can read none of it (`Facade.md`). The paths keep their meaning -
  they are not visible outside:
  - `/v1/messaging/*` - in the messaging tunnel (`/t`), answered by the messaging
    server
  - `/v1/account/*` - in the account tunnel (`/n`), answered by the server service

### The tunnel

Two kinds of frame, and a client's own identity is presented in exactly one of
them. Both tunnels work the same way and are opened independently, each when the
first call for it is made.

- **hello** - opens a tunnel. Hybrid-sealed to the sealing key of the backend it
  is for, which the client takes from that backend's card: `GET /v1/server/card`
  for the messaging server, `GET /v1/node/card` for the server service. Both cards
  are public, identical for everyone, and signed by the server root key whose
  fingerprint the user already had before they saw a facade, so a card can be
  withheld but not substituted; each names what it is for, so one is not taken
  for the other. A client takes a card only when it names the server
  fingerprint it joined. The frame carries a fresh secret, a one-time key for the answer,
  and the hybrid signature of the device that is opening it, with that device's
  certificate. The signature covers the CBOR `{secret, server}`, `server` being
  the fingerprint of the server the hello is for, and a server refuses a hello
  signed for another fingerprint. The answer - the
  tunnel's lifetime - is sealed to that one-time key.
  Each sealing key **must be hybrid**: sealing needs the ML-KEM half, and a
  server whose sealing key is not X25519 with ML-KEM-768, or whose identity is
  not Ed25519 with ML-DSA-65, refuses to start.
- **carry** - everything afterwards. The request is encrypted under AES-256-GCM
  with a key derived from the same secret (a different derivation from the session
  MAC key: one secret, two purposes, and holding one is not holding the other),
  addressed by the per-request handle described below. Each frame has a key of
  its own: HKDF-SHA256(tunnel key, salt = the handle, `bazarish tunnel request v1`)
  for a request and `bazarish tunnel answer v1` for its answer, and an answer
  must name its request's handle, so an answer opens only for its own request. No asymmetric work per
  request, which matters over a session's life: the held-open `events` request
  alone is sent again each time it is answered, and every fetch and
  acknowledgement is a request of its own.
- **A tunnel cannot open the next tunnel.** That is structural: a hello is only
  hybrid-sealed and signature-verified, and a carried frame is only opened with a
  tunnel key. So a leaked secret cannot renew itself, and the device signs again
  at every renewal, which is what the lifetime below bounds.
- **The answer's status is carried inside.** Outside it is always `200`, because
  a code visible to the facade would say what the seal was meant to hide.
- **Payloads are padded** to a step of a fixed ladder before they are encrypted,
  so the length of a waiting-mail list does not reveal the number of messages.
- Authentication, first contact: a **signed request** (`SignInWithKey.md`, section 1) made with the device's own two signing keys, its certificate in `X-Bazarish-Device` in place of `X-Bazarish-Keys`. The server verifies the certificate against the root it names and the signatures against the certificate's keys, and takes the caller UID from that root and the device and its role from the certificate - no prior key exchange. It remembers every accepted nonce for the freshness window, so a captured request replayed verbatim is rejected. The tunnel the hello opens belongs to that one device.
- Authentication, afterwards: a **session**, which is the tunnel. The signature above is presented once, in the hello, and every later request carries `X-Bazarish-Session` (this request's handle), `X-Bazarish-Seq` and `X-Bazarish-Mac` (HMAC-SHA256 over the base string of `SignInWithKey.md` section 1 - no nonce line - plus `seq + "\n"`) - **~174 bytes**, and an HMAC instead of an ML-DSA verification. Two properties beyond the size:
  - a signature is evidence a server can show a third party that this user made this request at this time; a MAC is not, because the server could have produced it itself;
  - the handle is `HMAC(secret, seq)`, different on every request, so nothing in the middle can tie a session's requests together by it. Nothing is transmitted to arrange this: both sides derive handles, and the server keeps only the next 64 of them indexed (a client runs two connections, so requests overtake each other).
- Sessions expire at a point drawn at random between **1 and 2 hours**: the random point keeps clients from renewing on one schedule, and the short life is what makes a leaked session key worth little. The issue response carries `expiresUnix`, so a client renews ahead of time (5 minutes early). A lapsed or unknown session answers `SESSION_INVALID`, which the client answers by opening another tunnel and sending the request through it - never by retrying the same way. An answered refusal - a status from the facade or the server, a hello the server does not welcome, a server card that does not verify - is not waited out: the next request asks again at once. Only a session that could not be opened because **no answer arrived** holds off the next try for a minute (60 s), and that wait ends early when the I2P link comes back (a network change counts) or the facade publishes a newer lease set.
- The device a request acts for is the one whose certificate signed the hello of its tunnel. The server picks a device's mail queue by it, and the signature verifies only under that certificate's keys, so nothing between the two ends can re-label a request as another device of the same account and acknowledge that device's mail before that device fetched it.
- Every request is checked against the revocation of the account's published card (`../Identity.md`, Revocation): a revoked device is refused with `403 DEVICE_REVOKED`, by the messaging server and by the server service alike, and the messaging server unregisters every revoked device of the account, dropping what only that device still owed.
- All errors follow `../Errors.md`.

## Account (server service)

| Endpoint | Purpose |
|---|---|
| `POST /v1/account/card` | Publish the **contact card**, signed by the account's root and sent by a primary device, with the account's card key (`{card, cardKey}`, the key as unpadded base64url; `FederatedResolve.md`): the epoch keys and the revocation plus the serving **destination** - from `POST /v1/account/i2p-dest` or `GET /v1/messaging/destination` - and the **serving sealing key**, which the account issued itself (see `../Identity.md`, `InviteAnonymity.md`). The card names no server and carries no validity window. A card that is not newer than the one held, or whose revocation `number` is lower than the held card's, is refused with `409`; one sent by a device that card revokes, with `403 DEVICE_REVOKED`; one sent by a secondary device, with `403 DEVICE_NOT_PRIMARY`. This call is also what turns a portal registration into an account - the first one redeems the grant (`ServicePortal.md`) - and it is repeated whenever the routing changes; a repeat grants nothing. Every card names a destination, and the server service refuses one that does not (`400`). The served destination can only be delegated once the account exists, so the first card names the account's own address: the client delegates its transient and publishes again immediately afterwards. Responds `{approval, registrationMessage}` |
| `DELETE /v1/account/registration` | Terminate the account. The destination is revoked, the server service discards the account record and the grants, and the messaging server is instructed to discard the mailbox and all other state held for that fingerprint (`DELETE /v1/messaging/account`, operator-signed). Only a primary device may make it; a secondary one is refused with `403 DEVICE_NOT_PRIMARY`. A client performs this call before removing its local profile and removes it only on success. **Idempotent**: a key for which the server service holds nothing receives the same response as a deletion, with nothing revoked or discarded, so that a client whose account an operator has already removed can still complete the operation locally. A client treats any 4xx response likewise: no state for the account remains on the server, and it removes its local copy. Only the absence of a response (or a 5xx) leaves both sides unchanged, with the reason reported to the user |
| `POST /v1/account/i2p-dest` | Upload a signed offline **transient** with the keys of its address: `{transient, expiresUnix, sealingKey, signingKey}`, the last two the private PEM of the address's hybrid serving sealing key and of its Ed25519 signing key (`../Identity.md`). An empty transient takes the address offline and carries no keys; one without them is refused with `400`. Every account owns a destination, so the only gates are that the account exists here and, on a moderated server service, that it is approved (`ACCOUNT_PENDING_APPROVAL` until then); the call is taken only from a primary device, for every transient, a renewal of the same address included, and for taking the address offline (`403 DEVICE_NOT_PRIMARY`). The server service passes the transient and the keys to the messaging server, which operates the destination with exactly these keys, and records its validity. Responds `{dest}`: the address this delegation gives. A server never issues a key of its own for an address, so the same transient and keys handed to another server serve the same address there (moving servers, `../Identity.md`). A transient over a new address also means the old destination is dropped at once (`CoreService.md`), which is what makes the fan-out that follows obligatory (`../Messages.md`) |
| `GET /v1/account/i2p-status` | `{approval, registrationMessage, transientExpires}`, where `approval` is `none`, `pending` or `approved`; the client asks it before it re-issues a transient. With `hold=1&since=<approval>` the server service holds the request while `approval` is still `<approval>` and answers when it changes, with no time limit: approved (`approved`), declined - the account removed (`none`) - or banned (`403 REQUEST_FAILED` with the ban reason). A pending account waits for its approval this way. Any later `i2p-status` request from the same device answers a held one, as does a broken link; a `since` that differs from the current state is answered at once |
| `POST /v1/account/registration` | Register with no portal page to visit (`ServicePortal.md`): answers `{ok}`; the approval and its message come with the first card (`POST /v1/account/card`) |
| `GET /v1/account/card?user=X` | The contact card behind a descriptor: returns `{user, card, cardKey}`, or `ACCOUNT_UNKNOWN` for an account with no card or one still awaiting approval. Reached from the messaging server, which serves a peer's query **on the user's own destination** - that destination is the capability, and it answers for its own user alone, so this cannot be used to ask whether a server hosts some other fingerprint. The `alias` namespace lives on the central resolver (`api/FederatedResolve.md`) |

The server service is bound to its messaging server by the node card: the messaging server signs it with the server root at every start of the server service, over their pinned link, and it carries the server service's own sealing key (`CoreService.md`). Nothing a contact can fetch names that server.

## Messaging (server)

The messaging server serves only the accounts its server service holds and has approved: before anything is written it asks the server service for the signer's card, and an account the server service does not know, has not approved yet, or has deleted is answered `ACCOUNT_UNKNOWN` (404). So an applicant on a moderated server service has no mailbox until the operator approves it, and a deleted account's mailbox cannot come back.

### Sending

A client delivers its own outgoing messages: it
dials the recipient's serving destination over I2P and performs the federation
exchange with it (`Federation.md`), from a destination held for that one
correspondent. The account's own server is never asked to dial a correspondent and
therefore does not learn who its users write to. The account's server holds no
outgoing state, and a client holds a delivery in memory only until its outcome is
reported.

The server performs no part of a delivery. Its remaining functions are receipt -
it operates the destination a correspondent dials, admits the envelope and holds
it in the mailbox until the recipient's devices have collected it - and the single
write a device makes to its own account, `POST /v1/messaging/self`, which reaches
no other party.

A **contact request**, which presents nothing, is capped by the protocol at **33966 sealed
bytes** (`kMaxContactRequestBytes`), with no margin: this is the one thing a stranger may put in a mailbox, so the
cap is the measured worst case.

The worst case is **33966 bytes**, measured through the real path: the two epoch
keys the bootstrap carries, the delivery pass the requester provides
so the peer can answer, the ML-KEM ciphertext of the seal, the authorship
block with the sending device's certificate in it (a request has no card and no
pass, so the signature is the only thing naming the sender), the longest greeting
a user may write and the longest name an account may carry. The body is padded to
twice the 16384-byte top step of the padding ladder, 32768 bytes, before it is
sealed, and the seal frame adds 1198 bytes, so every body from 16381 to 32764
bytes seals to exactly 33966: the variable-length parts move nothing until the
body outgrows that size. A test holds the cap within 64 bytes of the measured
request.

Two client-side caps are what keep the body inside that size: the greeting is
at most **100 bytes** (`kMaxContactGreetingBytes`) and an account's own display
name at most **64 bytes** (`kMaxAccountNameBytes`, about 32 characters of a
non-Latin alphabet). A value over either cap is refused where it is typed.

- A contact acceptance is not a separate endpoint: it is an ordinary content
  message presenting the pass received in the request (see `../Contacts.md`).
  Declining a request transmits nothing.
- A repeated delivery is the same delivery: the delivery id is derived from the
  message and the recipient mailbox (`../Messages.md`), so the recipient's server
  recognises the repetition and stores no second copy. It remembers a delivery id for 30 days after storing it, so a repetition within that time is answered as delivered.

### Own destination

| Endpoint | Purpose |
|---|---|
| `GET /v1/messaging/destination` | Returns the caller's assigned `{dest}` - the destination this server operates for the user (the blinded b33 of its encrypted LeaseSet2, a `.b32.i2p` host equal to the b33 of the master the client holds); its state is in `overview`. `dest` is empty only while nothing is delegated - the server derives it from the delegation at once, before the destination publishes - and a client refreshing its card then fills in its own master's b33, the same address either way. The client includes the dest and its own serving key in the contact card it signs |
| `GET /v1/messaging/overview` | What the account page shows of this server, in one answer: `{version, dest, state, transientExpires, usedBytes, quotaBytes, clients, devicesVersion}`. `dest` and `state` are those of `destination`, `transientExpires` the expiry of the delegation the server holds (`0` without one), `usedBytes` the bytes of pending content held for the account, `quotaBytes` the effective quota (the server service's grant, else the server default), `clients` the registered devices as `[{device, current, queue}]` - `current` marks the caller, `queue` counts the entries waiting for that device -, `devicesVersion` the version of the device list (`devices`, below). `version` is an opaque string that names this state: equal answers carry equal versions. With `hold=1&since=<version>` the server holds the request while the overview is still that version and answers when any field changes, with no time limit; a `since` that differs is answered at once. Any later `overview` request from the same device answers a held one, as does a broken link |

### Receiving

| Endpoint | Purpose |
|---|---|
| `GET /v1/messaging/events` | Answer in the same shape as `pending` plus the mailbox's `usedBytes` and `quotaBytes` and the `devicesVersion` (as in `overview`), at once. With `devices=<version>`, naming the version of the device list the device holds, a list of another version answers at once. With `hold=1` and nothing pending, hold the request open until something is pending for this device or the device list changes, with no time limit: the wait ends when mail arrives, when a device registers, leaves or is dropped, when the same device asks again (the earlier wait is answered with what is pending, which may be nothing) or when the link breaks. How a server holds it open is an implementation matter; what the protocol requires is that a delivery ends the wait, at once. With `deferred=<id>,<id>,...` the request names at most 32 entries (`kMaxDeferredMailItems`) this device holds without acknowledging them (`../Messages.md`): pending entries named there do not answer a held request, which is answered at once only when an entry not named there is pending, and otherwise held as above. An id that names no entry pending for this device is ignored, and a longer list is refused with `400`. This is the **only** way a client learns that mail has arrived, and a wait that ends in an error is repeated. A facade does not hold a wait for a server it cannot reach: it answers `502 UPSTREAM_UNAVAILABLE`, and since nothing announces the server's return, a client repeats that wait after a pause |
| `GET /v1/messaging/pending` | List pending entries for this device: `{pending: [{id, class, bytes}]}` where `class` is the delivery class `content`, `contact` or `device` (the admission selector, not the content type - see `../Messages.md`) and `bytes` the size of the stored item, so a client can cut its fetches and show how much is left. **Oldest arrival first**, and `events` answers in the same order: a burst is fetched in the order the mailbox took it. The order is arrival only - the position of each message in the conversation is decided by the `sentAt` inside the sealed envelope, which no server sees. The list is not paged: one answer lists every entry this device has not acknowledged, and the mailbox quota bounds its length. A client takes in as many entries per pass as it chooses and acknowledges them; while any remain unacknowledged and not named in `deferred`, `events` answers at once |
| `POST /v1/messaging/fetch` | Fetch several pending items in one answer: `{ids: [...]}` in, `application/cbor` out - `{items: [{id, blob}]}` with each `blob` a CBOR byte string holding the raw E2E payload, in the order asked. An item this device has already acknowledged, or that is no longer held, is left out. The items named may add up to at most `kMaxFetchBytes` (1 MiB, twice the largest item any delivery or self copy may be), otherwise the request is refused with `400`; the `bytes` of the listing is what a client cuts by. A client coming online takes its backlog as one stream: the next fetch is under way while it takes in the previous answer |
| `POST /v1/messaging/ack` | Confirm receipt of `{ids: [...]}` for this device, at most `kMaxAckIdsPerRequest` (256) at once; when all registered devices of the user have acked an item, or the one device a self-message names has acked it, the server deletes it immediately |

### Files (see `FileTransfer.md`)

A file is announced in the message and transferred directly between the two clients over a one-time I2P
destination when the recipient asks for it.

### Reseed

Every call in this API goes to a facade over I2P. For a client that has no
transport yet, a server publishes one file outside the API:

| Where | What |
|---|---|
| `GET <portal>/i2pseeds.su3` | This server's netDb as an **su3 reseed archive** - the ordinary I2P format, under the name that format gives it, read by any router. Unauthenticated by necessity (a client fetching its first netDb holds nothing yet) and safe by construction (every RouterInfo inside carries its own router's signature). Packed from a random sample of about sixty routers and rebuilt no more than once an hour: one archive for everyone who asks within that hour, so repeated requests learn no more of the netDb than a single one |

It is published by the **portal**, which is on the clearnet: a facade is an I2P
address, and a client needs this file *before* it has I2P.

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
carries. That is the only case in which the bootstrap leaves the network the
user chose.

### Delivery passes (see `../Contacts.md`)

| Endpoint | Purpose |
|---|---|
| `POST /v1/messaging/passes` | `{passes: [<hash>, ...]}` -> `{ok}`. Register the hashes of the passes this user's mailbox admits. Only hashes are held: a sender presents the pass, so what is on the server service's disk recognises a delivery and cannot produce one. Registering one already held is how a block is lifted |
| `POST /v1/messaging/passes/revoke` | `{passes}` -> `{ok}`. Drop exactly these passes; one that is not held is no error. Exact and immediate - the account names what it issued, so there is nothing to sweep for (`../Contacts.md`) |

### Device registry (see `../DeviceSync.md`)

| Endpoint | Purpose |
|---|---|
| `POST /v1/messaging/clients` | Register the device that signs the request, `{install, sealing}`: `install` is a random token made once for each copy of the account database, `sealing` the device's public sealing key (base64 SPKI), which must match the digest in its certificate (`400` otherwise). The server keeps the certificate the tunnel's hello presented. The same device registered again from the same copy is answered as registered; from another copy, `409 DEVICE_TAKEN`; a device the card revokes, `403 DEVICE_REVOKED` |
| `GET /v1/messaging/devices` | The device list: `{version, devices: [{cert, sealing, registered, departure}], card}` - every device of the account this server knows and the card does not revoke, with its certificate and public sealing key (base64), `registered` false for a device that left or fell silent, and `departure` its signed departure (base64) when it left; `card` is the account's published card (base64), from which the account's devices take its revocation. `version` changes whenever the list does |
| `DELETE /v1/messaging/clients/{id}` | Clear the queue of another device of the account: the server unregisters it, deletes what was held for it alone and keeps it on the list. Only a primary device may (`403 DEVICE_NOT_PRIMARY`), and not for itself (`400`) |
| `POST /v1/messaging/departure` | `{departure}`: the calling device's signed departure (`../Identity.md`, Revocation), which must name that device and verify under its certificate. The server unregisters the device - what was held for it alone is deleted - and keeps it on the list with the departure until the card revokes it |
| `POST /v1/messaging/self` | Store `{deliveryId, payload}` or `{deliveryId, to, payload}` (payload base64) in the account's own mailbox: without `to`, for every registered device but the caller; with it, for that one registered device, which must not be the caller, else `404 CLIENT_UNREGISTERED`. Answers `{ok}` (`../DeviceSync.md`) |

The device is the one the request's certificate names; the "delete when all devices received" rule operates on the registered set, and a self-message waits for the devices it was stored for. Requests to `events`, `pending`, `fetch`, `ack`, `passes`, `devices` and `self` from an unregistered device fail with `CLIENT_UNREGISTERED`; a departure is taken from any device the card does not revoke. A device leaves the registered set by leaving, by being revoked, by a primary device clearing its queue, or after 30 days without asking for mail (`kMailRetentionDays`); the last two keep it on the list as inactive, and such a device registers again when it next connects. A successful write that has nothing to report answers `{ok: true}`.
