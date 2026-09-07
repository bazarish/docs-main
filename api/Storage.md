# Storage Backend Interface

Status: draft skeleton.

Internal abstraction inside the server for keeping users' pending encrypted content. Two backends from the start:

- **Disk** - plain filesystem layout, the simple-deploy default.
- **Object storage** - S3-compatible API.

The server code depends only on the abstract interface; backend choice is server configuration.

## Stored data

- Pending encrypted blobs, keyed per recipient user.
- Per-blob delivery state: which of the user's client IDs have acked receipt.
- Per-user usage counters for quota enforcement.

Content is opaque ciphertext; the interface never interprets blob contents.

## Lifecycle rules

1. A blob is **deleted immediately** once all **registered** client IDs of the user (see `../DeviceSync.md`) have acked it - before any quota or TTL pressure. Retiring a client ID triggers re-evaluation: blobs that were waiting only on the retired client are deleted.
2. Maximum retention is the server's **mailbox retention** setting (`mailboxRetentionDays`); blobs older than that are purged.
3. When the service node deletes an account (idle, ended by its owner, or removed by an operator), it notifies the messaging server, which discards **everything** held for that fingerprint: pending content, replay markers, delivery passes, the device registry and any quota override. No mailbox may remain in service for an account that no longer exists.
4. Admission is refused (typed errors, see `../Errors.md`) when a blob exceeds the per-message limit or the user's quota; nothing is partially stored.

## Interface sketch (draft)

| Operation | Purpose |
|---|---|
| `put(user, blobId, data)` | Store a pending blob; fails atomically on quota/limit violations |
| `list(user, clientId)` | Enumerate blobs not yet acked by this client |
| `get(user, blobId)` | Read blob data |
| `ack(user, blobId, clientId)` | Record receipt; triggers deletion when the ack set is complete |
| `purgeUser(user)` | Drop everything for a user (account deleted) |
| `purgeExpired()` | TTL sweep |
| `usage(user)` | Current usage in bytes |

## Open questions

- Where delivery-state metadata lives for the object-storage backend (object tags vs sidecar objects vs a small local index).
- Encryption at rest: content is already end-to-end ciphertext; to be decided whether backend-level encryption adds sufficient value to justify the complexity.
- Crash consistency of the ack-set -> delete rule (avoid resurrecting acked content after restart).
