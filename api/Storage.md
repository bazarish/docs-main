# Mailbox Storage

How the messaging server keeps users' pending encrypted content. There is one
implementation, on disk, behind the server's `Storage` interface.

## Stored data

- Pending encrypted blobs, keyed per recipient user.
- Per blob: its delivery class, which of the user's client IDs have acked it, and
  its **arrival stamp** in milliseconds. The stamp is what the mailbox is ordered
  by, and it is the one thing a mailbox knows about a delivery beyond its size and
  its class - when it received it, never when it was written.
- Per user: the registered client IDs, the hashes of the delivery passes it
  issued, the replay markers (seen delivery ids, kept 30 days) and a quota
  override when the operator set one.

Usage is the sum of the stored blobs' sizes. Content is opaque ciphertext; storage
never interprets blob contents.

## Lifecycle rules

1. A blob is **deleted immediately** once all **registered** client IDs of the user (see `../DeviceSync.md`) have acked it - before any quota pressure. Retiring a client ID triggers re-evaluation: blobs that were waiting only on the retired client are deleted.
2. A blob nobody acks is kept: the account's quota is what bounds a mailbox, and admission is refused (rule 4) rather than old content dropped.
3. When the service node deletes an account (idle, ended by its owner, or removed by an operator), it notifies the messaging server, which discards **everything** held for that fingerprint: pending content, replay markers, delivery passes, the device registry and any quota override. No mailbox may remain in service for an account that no longer exists.
4. Admission is refused (typed errors, see `../Errors.md`) when a blob exceeds the per-message limit or the user's quota - a contact request over the quota gets `DELIVERY_REJECTED`; nothing is partially stored.
5. A listing is returned **oldest arrival first**. A mailbox is a queue and must
   behave as one: a backend that enumerated its own storage order would return a
   burst of messages to the client out of order, and the client has no other
   means of restoring their order - the time inside an envelope is sealed to the
   recipient, and this side never sees it.

## Operations

| Operation | Purpose |
|---|---|
| `putBlob(user, blobId, data, class, quota, alreadyAcked)` | Store a pending blob, or report it a duplicate or over the quota; the per-message limit is checked before the call, and `alreadyAcked` is the sending device of a self-delivery |
| `listPending(user, clientId)` | The blobs this client has not acked, with their class and size, oldest arrival first |
| `isPendingFor(user, blobId, clientId)` | Whether the blob is held and this client has not acked it; a fetch serves only those |
| `getBlob(user, blobId)` | Read blob data |
| `ackBlob(user, blobId, clientId)` | Record receipt; deletes the blob once every registered client has acked it |
| `forgetUser(user)` | Drop everything held for a user (account deleted) |
| `purgeExpired(receivedBefore)` | Drop replay markers and contact-request rate windows older than the cut-off; pending blobs are not touched. The server runs it hourly with a cut-off of 30 days |
| `usage(user)` | Current usage in bytes |

Beside these, storage keeps the client registry, the delivery passes, the replay
markers, the contact-request windows and the per-user quota.

## At rest and after a crash

The server does not encrypt content at rest. What a mailbox holds is already
sealed end to end to its owner, and the server's state directory is trusted to
its operator.

Every file is written whole under a temporary name and renamed into place, and an
ack is recorded before a delivery's files are removed, so a server that stops at
any point never lists a delivery again to a client whose ack it recorded. Writes
are not flushed to disk: after a power loss a recorded ack can be lost and the
delivery listed again, and the client keeps one copy per message id.
