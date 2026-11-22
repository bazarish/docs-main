# server-core <-> server-service API

Internal API over the operator private channel - mutually pinned TLS, each side holding the hash of the other's key, and every call signed by the identity that makes it. What the server service asks of the server rides the server's operator socket; the one call the other way, a card lookup by fingerprint, rides the server service's own listener and is authorised to the server's fingerprint alone. The server reaches the server service only at an `https` `serverService.url` with the server service's pin, and takes a lookup's answer only when the card in it is the asked account's. The server service owns per-operator user-management state (accounts: the quota granted and the card published); the server owns content and delivery, and consults the server service to make admission decisions. The global **alias** namespace belongs to a separate central service (`api/FederatedResolve.md`).

Registration opens an account without a term; the only clock is the delegated destination, and an account that has none for the server service's idle window is deleted (`idleDeleteDays`, 180 by default).

## Server -> Server service

| Request | Purpose |
|---|---|
| `get-card` | fingerprint -> the user's **contact card** (epoch keys + devices + destination + serving sealing key) and its card key, which seals the exchange. Backs the federation `card` request - the first-contact card fetch (`api/FederatedResolve.md`). The query arrives on that user's own destination and is answered for that user alone, so it cannot be used to ask a server whether it hosts some other fingerprint |

## Server service -> Server

Requests are signed with the server service's own identity and authorised by the server on
that fingerprint.

| Request | Purpose |
|---|---|
| `PUT /v1/messaging/i2p-dest` | Supply the user's signed offline transient with the address's keys, `{user, transient, expiresUnix, sealingKey, signingKey}` (private PEM, as the client sent them); an empty transient takes the destination offline. The server operates the destination with exactly these keys and generates none: a request with a transient and without both keys, or with a sealing key that is not hybrid or a signing key that is not Ed25519, is refused with `400`. Offline, the address and its record are dropped; published again, the client hands over the same keys, so every contact's card stays valid; a delegation that runs out is dropped the same way. Answers `{dest}` - the address derived from the transient, known before any tunnel is built. A transient over a **different** address replaces the destination at once: the old one is stopped and dropped inside the same call, so nothing is served on it afterwards and its lease set stops being published |
| `PUT /v1/messaging/card` | `{user}`: the account published a new card. The server drops the copy it holds, so the next `get-card` reads the new one |
| `PUT /v1/messaging/quota` | Set the mailbox quota granted by the account, in place of the server's default |
| `GET /v1/messaging/account?user=` | State held for one account: registered devices, bytes in use, effective quota; displayed by the administration panel |
| `POST /v1/messaging/node-card` | `{sealingKey}`, the server service's public sealing key (DER, base64) -> `{card}`: the **node card**, `{ v, t: "node-card", server, sealingKey }`, signed with the server root like the server card. The server service asks for it at every start over its pinned link to the server, which vouches for the signer: the `server` in the card is the fingerprint the server service then authorises the server by and serves it at `GET /v1/node/card`; a client seals the account tunnel to the key in it and trusts it by the same fingerprint as the server card (`ClientApi.md`). A server service that cannot get one does not start |
| `DELETE /v1/messaging/account?user=` | The account has been deleted: discard the mailbox, the replay markers, the delivery passes, the device list, the quota override and the destination record with its keys. Issued whenever an account is deleted, by its owner or by an operator. Idempotent: a fingerprint for which the server holds no state is already in the requested condition |

## Client-ID registry

Owned by the **server** (it drives the delete-when-all-acked storage rule, see `../DeviceSync.md`); the server service never learns the user's device count.

## Split of responsibilities

- **Push.** The server service pushes what it owns (the delegated transient, the
  quota, a deletion), and what it could not push it pushes again once the server
  answers, so the messaging server needs the server service for nothing on the delivery path.
  The one thing the server asks the server service for is a user's contact card, and it holds
  the answer briefly; a new card drops that copy at once (`PUT /v1/messaging/card`).
- **The quota is the server service's, the usage is the server's.** The server service grants and
  stores the number; only the filesystem under the server knows what is in a
  mailbox, so the panel and the portal read usage from the server
  (`GET /v1/messaging/account`).
- **Signed per request.** Every call carries the caller's hybrid signature and
  each side authorises it by fingerprint - the server service by the `server` named in its
  node card, the server by its configured `operators` list.
