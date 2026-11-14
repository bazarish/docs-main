# Mailbox Storage

How the messaging server keeps users' pending encrypted content. There is one
implementation, on disk, behind the server's `Storage` interface.

## Stored data

- Pending encrypted blobs, keyed per recipient user.
- Per blob: its delivery class, which of the user's devices have acked it, the
  one device it is for when it is an item to self, and its **arrival stamp** in
  milliseconds. The stamp is what the mailbox is ordered
  by, and it is the one thing a mailbox knows about a delivery beyond its size and
  its class - when it received it, never when it was written.
- Per user: the registered devices, each with the install token it registered
  from (`ClientApi.md`), the hashes of the delivery passes it
  issued, the replay markers (seen delivery ids, kept 30 days) and a quota
  override when the operator set one.

Usage is the sum of the stored blobs' sizes. Content is opaque ciphertext; storage
never interprets blob contents.

## Lifecycle rules

1. A blob is **deleted immediately** once all **registered** devices of the user (see `../DeviceSync.md`) have acked it - before any quota pressure. An item to self is listed only to the one device it is for and deleted as soon as that device acks it. Unregistering a revoked device triggers re-evaluation: blobs that were waiting only on it, and every item to self for it, are deleted.
2. A blob is deleted **30 days after it arrived** (`kMailRetentionDays`), acked or not. The account's quota bounds a mailbox in the meantime, and once it is reached admission is refused (rule 4). The same 30 days are how long a device keeps an epoch key after its epoch ends (`../Identity.md`, Epoch keys).
3. When the service node deletes an account (idle, ended by its owner, or removed by an operator), it notifies the messaging server, which discards **everything** held for that fingerprint: pending content, replay markers, delivery passes, the device registry and any quota override. No mailbox may remain in service for an account that no longer exists.
4. Admission is refused (typed errors, see `../Errors.md`) when a blob exceeds the per-message limit or the user's quota - a contact request over the quota gets `CONTACT_RATE_LIMITED`; nothing is partially stored.
5. A listing is returned **oldest arrival first**. A mailbox is a queue, and its
   arrival order is the only order this side can give: the time inside an
   envelope is sealed to the recipient, and this side never sees it.

## Operations

| Operation | Purpose |
|---|---|
| `putBlob(user, blobId, data, class, quota, receivedAt)` | Store a pending blob and its replay marker, or report it a duplicate or over the quota; the per-message limit is checked before the call |
| `putAddressedBlob(user, blobId, data, class, quota, receivedAt, device)` | The same for an item to self, pending for that one device |
| `listPending(user, device)` | The blobs this device has not acked and that are not for another device, with their class and size, oldest arrival first |
| `isPendingFor(user, blobId, device)` | Whether the blob is held, is not for another device and this device has not acked it; a fetch serves only those |
| `getBlob(user, blobId)` | Read blob data |
| `ackBlob(user, blobId, device)` | Record receipt; deletes the blob once every registered device has acked it, or at once when it is an item to self for this device |
| `forgetUser(user)` | Drop everything held for a user (account deleted) |
| `purgeExpired(receivedBefore, storedBefore)` | Drop replay markers and contact-request rate windows older than the first cut-off and blobs that arrived before the second. The server runs it hourly with both cut-offs 30 days back |
| `usage(user)` | Current usage in bytes |

Beside these, storage keeps the device registry, the delivery passes, the replay
markers, the contact-request windows and the per-user quota.

## At rest and after a crash

The server does not encrypt content at rest. What a mailbox holds is already
sealed end to end to its owner, and the server's state directory is trusted to
its operator.

Every file is written whole under a temporary name, flushed to disk, and renamed
into place, and the directory is flushed after every rename and removal; an
operation returns only once its change is on disk. A crash or a power loss
therefore keeps everything that was answered:

- a delivery is stored as its record, its blob and its replay marker, in that
  order and under one lock, before the sender is answered, so no client can take
  and ack it before the marker that recognises a retry is on disk;
- the ack that completes a delivery removes it, blob first, without rewriting its
  record; a record left without its blob is never listed and counts for nothing;
- unregistering a device removes what only that device still owed before the
  device itself;
- a temporary left by an interrupted write is not a blob and counts for nothing.
