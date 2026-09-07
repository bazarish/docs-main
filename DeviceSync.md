# Multi-Device Synchronization

Status: design draft (decided direction).

A user runs several clients (devices); each has a unique random **client ID**. Contact bookkeeping, delivery passes, and message history must converge across them without giving servers anything new to read.

## Model

- **Shared identity keys.** All clients of a user hold the same hybrid identity keypair set (see `Identity.md`); a new device receives it during enrollment. Senders never need to know how many devices a user has.
- **Explicit client registry.** Client IDs are registered and retired via the API (see `api/ClientApi.md`). The "delete when all clients received" rule operates on this definite set; implicit discovery would admit races and leave retired identifiers holding quota.
- **Sync channel = own mailbox.** Sync data travels as ordinary E2E deliveries to the user's own mailbox, written through `POST /v1/messaging/self` and read back through the standard pending/ack path. The server sees a delivery class and nothing else about it, and the "all clients acked -> delete immediately" rule performs the cleanup.
- With a **single registered client**, no sync traffic is produced at all.

## What actually syncs

There is no event journal, no per-device sequence and no CRDT: the mechanism is
the flat list of `device.*` self-messages below, and the address-book transfer
(`device.contacts-request` / `device.contacts`) is what a full-state handover
looks like. An earlier draft of this file specified a journal with
`message-echo`, `snapshot` and gap detection; none of it was built, and it is
gone from here rather than left standing as a design somebody might implement
against.

Sync data travels as ordinary E2E deliveries to the user's own mailbox, but not
through the same door as mail: `POST /v1/messaging/self` is its own endpoint,
with its own delivery class (`device`), and it is deliberately outside the budget
a stranger's contact request is counted against. A server that had only one
registered device stores nothing at all.

## Enrollment (adding a device)

1. An existing device displays a QR code containing: the identity private keys (classical + post-quantum), the home server fingerprint + facade secret URI, and the user-owned destination key. The new device re-fetches the user's current routing (destination + serving sealing key) from the server (`GET /v1/messaging/destination`). The ML-DSA private key (~4 KB) exceeds practical QR capacity; the enrollment bundle may therefore require a QR-carried one-time secret together with a transfer over a local channel. To be determined in client design.
2. The new device authenticates as the user and **registers its client ID**.
3. It asks for the address book (`device.contacts-request`); every other device that has contacts answers.
4. Content delivered and fully acked *before* registration is gone by design and is not replayed; prior history arrives with the address book and the message echo.

## Retirement (removing a device)

- Any device can retire a client ID (lost/stolen device included). The server immediately **re-evaluates ack-completeness** of pending content: blobs that were waiting only on the retired client are deleted.
- **Limitation:** with a shared identity key, a stolen device retains the identity indefinitely. Retirement stops deliveries to it but does not revoke the key. A correspondent can revoke the pass the account holds, which stops the stolen device writing to that one mailbox; the identity itself is unrecoverable. Per-device keys with cross-signing are the known evolution path if this risk proves unacceptable in practice.

## Account sync (avatar & contact names)

Two pieces of account state propagate to the user's other devices as dedicated **self-messages** - sealed to the user's own key and delivered to their own destination - independent of the journal above:

- `device.avatar` - the user's own avatar, re-sent whenever it changes.
- `device.contact-name` - a local contact rename, carrying `peer` (fingerprint) and the new `name`.
- `device.message` - what this device just sent a contact, so the conversation reads the same on all of them.
- `device.delegation-term` - the term this account delegates its destination for; the shortest choice wins.
- `device.saved` - a message the user kept for themselves, carrying it in `message`. The chat it belongs to is the account's own fingerprint, which no contact can hold.
- `device.saved-clear` - that chat, emptied on every device.
- `device.contact-remove` - a contact removed, carrying `peer`. Every device drops the contact and its conversation; the device that sent it is also the one that revoked their pass.
- `device.contact-block` - `peer` and `blocked`: mail and contact requests from them are dropped as they are read, on every device. The conversation is not touched.
- `device.contact-prefs` - `peer`, `notifications` and `allowCalls`: what one contact may do here.
- `device.contact-accepted` - `peer`: a contact request agreed to. Every device holds the same request, so without this the others go on offering an Agree that has been given - and, worse, go on treating the contact as one nothing may be sent to. Receiving it sends the correspondent nothing: the device that agreed carries the exchange.
- `device.chat-clear` - `peer`: this account's copy of one conversation, emptied. Distinct from the `chat.clear` sent to a correspondent, which asks *them* to empty theirs.
- `device.read` - `peer` and `ts`: this conversation has been read through the
  message sent at that moment. Reading is the account's act, not the device's, so
  a message read on one device is not offered as new on the next one. The mark is
  the message's own **sent-at**, because a local row id is a different number on
  every device, while the sender's stamp is the same everywhere.

  A receiver folds it into its own read high-water **once, on arrival**: it takes
  the newest incoming message it holds at or before that moment and marks through
  it. It is deliberately not kept as a standing rule that anything older counts as
  read - a `sentAt` is written by the sender, and a correspondent who back-dates a
  message would otherwise have it arrive already read and never be seen. A message
  that turns up after the mark is unread whatever it claims about when it was sent.

  A device sends it while the user reads, coalesced rather than per message: one
  notice per conversation after reading settles, and one when the conversation is
  left.
- `device.account-name` - `name`: the account renamed itself.
- `device.account-prefs` - `acceptCalls` and `sendReceipts`: the account's own answers, which must be the same wherever it is reached from. A notice carries what it knows about; a field it omits is left as it was.
- `call.taken` - the device that answered or refused an incoming call, stopping the others ringing (`api/Calls.md`).
- `device.contacts-request` - `device`: the asking device wants the address book. Sent once, automatically, on a device's first sync that reaches the server, and by hand from the account window. A device ignores its own request; with no other device the mailbox simply holds it.
- `device.contacts` - `forDevice`, `contacts`: the book, or a part of it. Every other device that has contacts answers, so several answers may arrive; each is taken in turn.

### The address book

A device that has just joined cannot learn a contact from the network: a correspondent is reached by destination and their card is read with a capability, and nothing turns a fingerprint into either. The account's other devices are the only place the book can come from - so it is asked for, over the same self-channel.

One entry carries what a contact **is**: their fingerprint, their identity keys (what every message of theirs is checked against, `Messages.md`), their sealing key, destination, serving key and view capability, whether they refused to be passed on, the local display name, the per-contact switches, whether this account has issued to them, whether they are blocked, and their avatar.

It carries the contact's **delivery pass** along with everything else about them, and this is the one thing it deliberately did not carry before: a one-time token could not be in two places, so a device that learned a contact this way had to buy its own before it could write. A pass is not spent, so every device holds the same one and a device that learns a contact can write to them at once. What it still carries nothing of is anything that belongs to one device or one exchange in flight.

Answers are **chunked**: one contact is never split across two messages, and an entry whose avatar would not fit is sent without it (the correspondent sends their face again the next time they write; the address they are reached at is what cannot be re-derived). The chunks carry no sequence number and no end marker: the merge is per message and fills gaps, so the order they arrive in does not matter and there is nothing for the asker to wait for. An empty chunk is not sent.

Merging **fills gaps only**. A contact this device already has is not overwritten: routing learned from the correspondent themselves is fresher than a copy of it, and a second answer from another device must not undo a rename or a move. Missing fields - keys, avatar, name, switches - are taken from the book.

`device.message` is sent **after** the recipient's server has signed for the delivery it echoes, not when the message is written. An echo that goes out first puts on every other device a message that may never arrive - and one copy per retry, all of them shown as still sending - which is a desync with no way back. What every device shows is therefore what at least one server holds. `device.message` carries what this device did in a conversation, so every device shows the same one: a message sent, a picture, a voice message, an edit, a deletion, a reaction, and a request to a correspondent to clear the chat. What a device does *not* echo is its own errands - read receipts, call signalling and the file-transfer handshake - which are between it and the correspondent. The read state is not among them: it is the account's, not the device's, and it travels as `device.read` above. Where a message has been read from is nobody's business, but *that* it has been read is the same fact on every device the account is open on - a second device that offered everything again as new was reporting something that is not true.

All are local-only facts that must never reach a contact, so they travel the self-channel exclusively. The routing-key sibling `device.i2p-master` (preserving the user's I2P address across devices, see `Identity.md`) uses the same mechanism. Wire shapes are in `Messages.md`.

**`device.i2p-master-request` (`{device, host}`)** is the other half of that pair. An account has **one** address, and it is the address every contact holds; a device that finds its server serving an address it has no keys for asks for them here, naming the address it needs (`host`) and itself (`device`). A device answers with `device.i2p-master` only when it holds the address that was named (an unnamed `host` asks for whatever the answerer has), and the asker takes the one whose address matches what the server serves. A device answers its own request never, and a device that already holds the served address has nothing to ask.

A `device.i2p-master` that carries a **different** master than the receiving device holds is not ignored either: the account's address has moved, and a device still announcing the old one in its outgoing routing keeps its contacts writing to an address nobody serves. It is adopted - replacing the stale master and the delegation signed by it - **only when the server confirms that address is the one it serves**, so a stale or replayed copy cannot take a working address away from a device.

Until that question is settled a device **publishes nothing**: minting an address of its own and delegating it would take the account's address away from every contact holding it, and none of them would learn a replacement until the user wrote to them again. What a client does when nobody answers is its own choice (this implementation puts it to the user); what the protocol requires is that it not publish silently.

**Every `device.*` type is a message from one of the account's own devices, and only that.** The channel is the account's own mailbox, but a mailbox is where correspondents write too: admission (a delivery pass) is not authorship, so a receiver checks the signer of a device message against its own identity and drops it otherwise - before any field of it is read. Without that check an admitted contact could change an account setting or write into its address book, both of which these types do.

A self-message is named by a **delivery id of its own**, unrelated to the one the same message carries to its recipient (`Messages.md`): the copy kept for the account's devices and the copy the contact receives share no name, so the two servers holding them cannot be matched against each other.

## Quota interplay

`device.message` with attachments doubles outgoing traffic and temporarily occupies the user's own quota: echo blobs are retained only until every registered client has fetched them, so for devices that are normally online the occupancy is brief. An offline device accumulates echoes for the retention period; substantial attachment traffic on the free tier (50 MB) will exhaust the quota. This is an accepted trade-off, to be reviewed against operational experience.

## Open questions

- **Address-book size vs quota** - a long-lived account's book is sent in as many messages as it takes, one contact never spanning two; whether that needs an explicit bound is open.
- Client-side policy for `device.message` (for example, omitting attachments above a threshold). The protocol permits a partial echo carrying an attachment stub; defaults to be determined in client design.
- Per-device keys as a future identity model (see Retirement).
