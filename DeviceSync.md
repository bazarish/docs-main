# Multi-Device Synchronization

Status: design draft (decided direction).

A user runs several clients (devices); each has a unique random **client ID**. Contact bookkeeping, delivery-token state, and message history must converge across them without giving servers anything new to read.

## Model

- **Shared identity keys.** All clients of a user hold the same hybrid identity keypair set (see `Identity.md`); a new device receives it during enrollment. Senders never need to know how many devices a user has.
- **Explicit client registry.** Client IDs are registered and retired via the API (see `api/ClientApi.md`). The "delete when all clients received" rule operates on this definite set; implicit discovery would admit races and leave retired identifiers holding quota.
- **Sync channel = own mailbox.** Sync data travels as ordinary E2E deliveries to the user's own mailbox through the standard send/pending/ack path. No dedicated server mechanism; the server cannot distinguish sync traffic from ordinary messages, and the "all clients acked -> delete immediately" rule performs the cleanup without an additional mechanism.
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
3. deliveryId dedup on the recipient's server guarantees a successfully delivered message is never duplicated.

`token-spent` events converge the known-unspent sets, so collisions are infrequent; strict partitioning of the stash is not required.

## Enrollment (adding a device)

1. An existing device displays a QR code containing: the identity private keys (classical + post-quantum), the home server fingerprint + facade secret URI, and the user-owned destination key. The new device re-fetches the user's current routing (destination + serving sealing key) from the server (`GET /v1/messaging/destination`). The ML-DSA private key (~4 KB) exceeds practical QR capacity; the enrollment bundle may therefore require a QR-carried one-time secret together with a transfer over a local channel. To be determined in client design.
2. The new device authenticates as the user and **registers its client ID**.
3. It announces itself with a sync event; an existing device responds with a `snapshot` (contacts, token bookkeeping, settings, recent history).
4. Content delivered and fully acked *before* registration is gone by design and is not replayed; prior history arrives via the snapshot.

## Retirement (removing a device)

- Any device can retire a client ID (lost/stolen device included). The server immediately **re-evaluates ack-completeness** of pending content: blobs that were waiting only on the retired client are deleted.
- **Limitation:** with a shared identity key, a stolen device retains the identity indefinitely. Retirement stops deliveries to it but does not revoke the key. A token epoch reset limits mailbox abuse; the identity itself is unrecoverable. Per-device keys with cross-signing are the known evolution path if this risk proves unacceptable in practice.

## Account sync (avatar & contact names)

Two pieces of account state propagate to the user's other devices as dedicated **self-messages** - sealed to the user's own key and delivered to their own destination - independent of the journal above:

- `device.avatar` - the user's own avatar, re-sent whenever it changes.
- `device.contact-name` - a local contact rename, carrying `peer` (fingerprint) and the new `name`.
- `device.message` - what this device just sent a contact, so the conversation reads the same on all of them.
- `device.delegation-term` - the term this account delegates its destination for; the shortest choice wins.
- `device.saved` - a message the user kept for themselves, carrying it in `message`. The chat it belongs to is the account's own fingerprint, which no contact can hold.
- `device.saved-clear` - that chat, emptied on every device.
- `device.contact-remove` - a contact removed, carrying `peer`. Every device drops the contact and its conversation; the device that sent it is also the one that revoked their tokens.
- `device.contact-block` - `peer` and `blocked`: mail and contact requests from them are dropped as they are read, on every device. The conversation is not touched.
- `device.contact-prefs` - `peer`, `notifications` and `allowCalls`: what one contact may do here.
- `device.chat-clear` - `peer`: this account's copy of one conversation, emptied. Distinct from the `chat.clear` sent to a correspondent, which asks *them* to empty theirs.
- `device.account-name` - `name`: the account renamed itself.
- `device.account-prefs` - `acceptCalls` and `sendReceipts`: the account's own answers, which must be the same wherever it is reached from. A notice carries what it knows about; a field it omits is left as it was.
- `call.taken` - the device that answered or refused an incoming call, stopping the others ringing (`api/Calls.md`).
- `device.contacts-request` - `device`: the asking device wants the address book. Sent once, automatically, on a device's first sync that reaches the server, and by hand from the account window. A device ignores its own request; with no other device the mailbox simply holds it.
- `device.contacts` - `forDevice`, `seq`, `last`, `contacts`: the book, or a part of it. Every other device that has contacts answers, so several answers may arrive; each is taken in turn.

### The address book

A device that has just joined cannot learn a contact from the network: a correspondent is reached by destination and their card is read with a capability, and nothing turns a fingerprint into either. The account's other devices are the only place the book can come from - so it is asked for, over the same self-channel.

One entry carries what a contact **is**: their fingerprint, their identity keys (what every message of theirs is checked against, `Messages.md`), their sealing key, destination, serving key and view capability, whether they refused to be passed on, the local display name, the per-contact switches, whether this account has issued to them, whether they are blocked, and their avatar.

It carries nothing that belongs to one device or one exchange in flight - above all **no delivery tokens**: they are one-time, and two devices spending the same one is a message lost. A device that receives a contact this way asks for its own batch through the ordinary path when it first needs to write.

Answers are **chunked**: one contact is never split across two messages, and an entry whose avatar would not fit is sent without it (the correspondent sends their face again the next time they write; the address they are reached at is what cannot be re-derived). `last` marks the final chunk of one device's answer.

Merging **fills gaps only**. A contact this device already has is not overwritten: routing learned from the correspondent themselves is fresher than a copy of it, and a second answer from another device must not undo a rename or a move. Missing fields - keys, avatar, name, switches - are taken from the book.

`device.message` carries what this device did in a conversation, so every device shows the same one: a message sent, a picture, a voice message, an edit, a deletion, a reaction, and a request to a correspondent to clear the chat. What a device does *not* echo is its own errands - read receipts, call signalling, token refills and the file-transfer handshake - which are between it and the correspondent, and the read state, which is per device by design: a message read here is not read there.

All are local-only facts that must never reach a contact, so they travel the self-channel exclusively. The routing-key sibling `device.i2p-master` (preserving the user's I2P address across devices, see `Identity.md`) uses the same mechanism. Wire shapes are in `Messages.md`.

**`device.i2p-master-request` (`{device, host}`)** is the other half of that pair. An account has **one** address, and it is the address every contact holds; a device that finds its server serving an address it has no keys for asks for them here, naming the address it needs (`host`) and itself (`device`). Every device that holds a master answers with `device.i2p-master`; the asker takes the one whose address matches what the server serves and ignores the rest. A device answers its own request never, and a device that already holds the served address has nothing to ask.

A `device.i2p-master` that carries a **different** master than the receiving device holds is not ignored either: the account's address has moved, and a device still announcing the old one in its outgoing routing keeps its contacts writing to an address nobody serves. It is adopted - replacing the stale master and the delegation signed by it - **only when the server confirms that address is the one it serves**, so a stale or replayed copy cannot take a working address away from a device.

Until that question is settled a device **publishes nothing**: minting an address of its own and delegating it would take the account's address away from every contact holding it, and none of them would learn a replacement until the user wrote to them again. What a client does when nobody answers is its own choice (this implementation puts it to the user); what the protocol requires is that it not publish silently.

A self-message is named by a **delivery id of its own**, unrelated to the one the same message carries to its recipient (`Messages.md`): the copy kept for the account's devices and the copy the contact receives share no name, so the two servers holding them cannot be matched against each other.

## Quota interplay

`message-echo` with attachments doubles outgoing traffic and temporarily occupies the user's own quota: echo blobs are retained only until every registered client has fetched them, so for devices that are normally online the occupancy is brief. An offline device accumulates echoes for the retention period; substantial attachment traffic on the free tier (20 MB) will exhaust the quota. This is an accepted trade-off, to be reviewed against operational experience.

## Open questions

- **Snapshot size vs quota** - a long-lived account's snapshot may exceed the free quota; chunked snapshots or an explicit size bound needed.
- Client-side policy for `message-echo` (for example, omitting attachments above a threshold). The protocol permits a partial echo carrying an attachment stub; defaults to be determined in client design.
- Per-device keys as a future identity model (see Retirement).
