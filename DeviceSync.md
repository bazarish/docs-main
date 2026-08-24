# Multi-Device Synchronization

Status: design draft (decided direction).

A user runs several clients (devices); each has a unique random **client ID**. Contact bookkeeping, delivery-token state, and message history must converge across them without giving servers anything new to read.

## Model

- **Shared identity keys.** All clients of a user hold the same hybrid identity keypair set (see `Identity.md`); a new device receives it during enrollment. Senders never need to know how many devices a user has.
- **Explicit client registry.** Client IDs are registered and retired via the API (see `api/ClientApi.md`). The "delete when all clients received" rule operates on this definite set - no races from implicit discovery, no dead IDs pinning quota.
- **Sync channel = own mailbox.** Sync data travels as ordinary E2E deliveries to the user's own mailbox through the standard send/pending/ack path. No dedicated server mechanism; the server cannot distinguish sync from regular messages, and the "all clients acked -> delete immediately" rule provides cleanup for free.
- With a **single registered client**, no sync traffic is produced at all.

## Sync journal

Each sync message carries a batch of events stamped `(deviceId, seq)` - a per-device monotonically increasing sequence.

| Event | Content |
|---|---|
| `contact-added` / `contact-revoked` | Contact identity, contact card (destination + serving sealing key) |
| `token-batch-issued` | Which tokens (and their hashes) went to which contact |
| `token-hashes-registered` | Hashes registered with the own server |
| `token-spent` | Outgoing token consumed for a peer |
| `message-echo` | Full copy of an outgoing message, **including attachments**, so every device shows the complete conversation |
| `snapshot-request` / `snapshot` | Full-state transfer (bootstrap or gap recovery) |

- Events have **CRDT set semantics** (grow-only sets plus tombstones for revocations) - no global ordering is needed; applying them in any order converges.
- A **gap in a peer device's `seq`** (events expired from the mailbox while this device was offline longer than the retention period) means state may be incomplete -> the device emits `snapshot-request` and a live device answers with `snapshot`.

## Outgoing token contention

Two devices may race to spend the same token from the shared per-contact stash. Handled optimistically:

1. A device picks a random token it believes unspent.
2. If another device already burned it, delivery fails with `DELIVERY_REJECTED` -> retry with another token.
3. messageId dedup on the recipient's server guarantees a successfully delivered message is never duplicated.

`token-spent` events keep the known-unspent sets converging, making collisions rare; no strict stash partitioning is needed.

## Enrollment (adding a device)

1. An existing device displays a QR code containing: the identity private keys (classical + post-quantum), the home server fingerprint + facade secret URI, and the user-owned destination key. The new device re-fetches the user's current routing (destination + serving sealing key) from the server (`GET /v1/messaging/destination`). The ML-DSA private key (~4 KB) pushes the payload beyond comfortable QR capacity - the enrollment bundle may need a QR-carried one-time secret plus a local-channel transfer; settle at client design time.
2. The new device authenticates as the user and **registers its client ID**.
3. It announces itself with a sync event; an existing device responds with a `snapshot` (contacts, token bookkeeping, settings, recent history).
4. Content delivered and fully acked *before* registration is gone by design and is not replayed; prior history arrives via the snapshot.

## Retirement (removing a device)

- Any device can retire a client ID (lost/stolen device included). The server immediately **re-evaluates ack-completeness** of pending content: blobs that were waiting only on the retired client are deleted.
- **Honest limitation:** with a shared identity key, a stolen device keeps the identity forever - retirement stops deliveries to it but cannot revoke the key. A token epoch reset limits mailbox abuse; the identity itself is unrecoverable. Per-device keys with cross-signing are the known evolution path if this risk proves unacceptable in practice.

## Account sync (avatar & contact names)

Two pieces of account state propagate to the user's other devices as dedicated **self-messages** - sealed to the user's own key and delivered to their own destination - independent of the journal above:

- `device.avatar` - the user's own avatar, re-sent whenever it changes.
- `device.contact-name` - a local contact rename, carrying `peer` (fingerprint) and the new `name`.

Both are local-only facts that must never reach a contact, so they travel the self-channel exclusively. The routing-key sibling `device.i2p-master` (preserving the user's I2P address across devices, see `Identity.md`) uses the same mechanism. Wire shapes are in `Messages.md`.

## Quota interplay

`message-echo` with attachments doubles outgoing traffic and transiently occupies the user's own quota: echo blobs live only until all registered clients fetch them, so with normally-online devices the pressure is brief. An offline device accumulates echoes up to the retention period - heavy attachment use on the free tier (20 MB) will hit the quota; this is an accepted tradeoff, to be revisited if practice shows pain.

## Open questions

- **Snapshot size vs quota** - a long-lived account's snapshot may exceed the free quota; chunked snapshots or an explicit size bound needed.
- Client-side policy knobs for `message-echo` (e.g. skip attachments above a threshold) - protocol allows partial echo with an attachment stub; decide defaults at client design time.
- Per-device keys as a future identity model (see Retirement).
