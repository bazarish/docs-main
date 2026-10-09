# Server <-> Service Node API

Status: implemented.

Internal API over the operator private channel - mutually pinned TLS, each side holding the hash of the other's key, and every call signed by the identity that makes it. What the node asks of the server rides the server's operator socket; the one call the other way, a card lookup by fingerprint, rides the node's own listener and is authorised to the server's fingerprint alone. The service node owns per-operator user-management state (accounts: the quota granted and the card published); the server owns content and delivery, and consults the service node to make admission decisions. The global **alias** namespace is **not** here - it is a separate central service (`api/FederatedResolve.md`), not the operator service node.

An account has **no term**. Registration opens an unbounded channel; the only clock is the delegated destination, and an account that has none for the node's idle window is deleted (`idleDeleteDays`, 180 by default).

## Server -> Service node

| Request | Purpose |
|---|---|
| `get-card` | fingerprint -> the user's **contact card** (sealing key + destination + serving sealing key). Backs the federation `card` request - the first-contact card fetch (`api/FederatedResolve.md`). The query arrives on that user's own destination and is answered for that user alone, so it cannot be used to ask a server whether it hosts some other fingerprint |

## Service node -> Server (implemented)

Requests are signed with the node's own identity and authorised by the server on
that fingerprint; no shared secret is used.

| Request | Purpose |
|---|---|
| `PUT /v1/messaging/i2p-dest` | Supply the user's signed offline transient; an empty transient revokes the destination. Answers `{dest, servingKey}` - the address derived from the transient (known before any tunnel is built) and the sealing key that destination serves with, which the client cannot derive. A transient over a **different** address replaces the destination at once: the old one is stopped and dropped inside the same call, so nothing is served on it afterwards and its lease set stops being published. The serving sealing key is kept only while the address is unchanged |
| `PUT /v1/messaging/quota` | Set the mailbox quota granted by the account, in place of the server's default |
| `GET /v1/messaging/account?user=` | State held for one account: registered devices, bytes in use, effective quota; displayed by the administration panel |
| `DELETE /v1/messaging/account?user=` | The account has been deleted: discard the mailbox, the replay markers, the delivery passes, the device registry and the quota override. Issued whenever an account is deleted, by its owner or by an operator. Idempotent: a fingerprint for which the server holds no state is already in the requested condition |

## Client-ID registry

Owned by the **server** (it drives the delete-when-all-acked storage rule, see `../DeviceSync.md`); the service node never learns the user's device count.

## What the split settled on

- **Push, not pull.** The node pushes what it owns (the delegated transient, the
  quota, a deletion) and reconciles the quotas once at
  start, so the messaging server needs the node for nothing on the delivery path.
  The one thing the server asks the node for is a user's contact card, and it holds
  the answer briefly rather than calling per delivery.
- **The quota is the node's, the usage is the server's.** The node grants and
  stores the number; only the filesystem under the server knows what is in a
  mailbox, so the panel and the portal read usage from the server
  (`GET /v1/messaging/account`).
- **Localhost HTTP, signed per request.** Channel isolation is not the
  authentication: every call carries the caller's hybrid signature and each side
  authorises it by fingerprint - the node by the server card it is bound to, the
  server by its configured `operators` list. No shared secret.
