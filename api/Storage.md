# Storage Backend Interface

Status: draft skeleton.

Internal abstraction inside the server for keeping users' pending encrypted content. One backend:

- **Disk** - plain filesystem layout.

The server code depends only on the abstract interface, so another backend could be added; none exists, and there is no configuration key that would select one.

## Stored data

- Pending encrypted blobs, keyed per recipient user.
- Per-blob delivery state: which of the user's client IDs have acked receipt.
- Per-blob **arrival stamp**, in milliseconds. It is what the mailbox is ordered
  by, and it is the one thing a mailbox knows about a delivery beyond its size
  and its class - when it took it, never when it was written.
- Per-user usage counters for quota enforcement.

Content is opaque ciphertext; the interface never interprets blob contents.

## Lifecycle rules

1. A blob is **deleted immediately** once all **registered** client IDs of the user (see `../DeviceSync.md`) have acked it - before any quota or TTL pressure. Retiring a client ID triggers re-evaluation: blobs that were waiting only on the retired client are deleted.
2. Maximum retention is the server's **mailbox retention** setting (`mailboxRetentionDays`); blobs older than that are purged.
3. When the service node deletes an account (idle, ended by its owner, or removed by an operator), it notifies the messaging server, which discards **everything** held for that fingerprint: pending content, replay markers, delivery passes, the device registry and any quota override. No mailbox may remain in service for an account that no longer exists.
4. Admission is refused (typed errors, see `../Errors.md`) when a blob exceeds the per-message limit or the user's quota; nothing is partially stored.
5. A listing is returned **oldest arrival first**. A mailbox is a queue and has to
   read like one: a backend that enumerated its own storage order would hand a
   burst of messages to the client shuffled, and the client has nothing else to
   put them back in order with - the time inside an envelope is sealed to the
   recipient, and this side never sees it.

## Interface sketch (draft)

| Operation | Purpose |
|---|---|
| `put(user, blobId, data)` | Store a pending blob; fails atomically on quota/limit violations |
| `list(user, clientId)` | Enumerate blobs not yet acked by this client, oldest arrival first |
| `get(user, blobId)` | Read blob data |
| `ack(user, blobId, clientId)` | Record receipt; triggers deletion when the ack set is complete |
| `purgeUser(user)` | Drop everything for a user (account deleted) |
| `purgeExpired()` | TTL sweep |
| `usage(user)` | Current usage in bytes |

## Open questions

- Encryption at rest: content is already end-to-end ciphertext; to be decided whether backend-level encryption adds sufficient value to justify the complexity.
- Crash consistency of the ack-set -> delete rule (avoid resurrecting acked content after restart).
