# Multi-Device Synchronization

A user runs several clients (devices); each has a unique random **client ID**. Contact bookkeeping, delivery passes, and message history must converge across them without giving servers anything new to read.

## Model

- **Shared identity keys.** All clients of a user hold the same hybrid identity keypair set (see `Identity.md`); a new device receives it during enrollment. Senders never need to know how many devices a user has.
- **Explicit client registry.** Client IDs are registered and retired via the API (see `api/ClientApi.md`). The "delete when all clients received" rule operates on this definite set; implicit discovery would admit races and leave retired identifiers holding quota.
- **Sync channel = own mailbox.** Sync data travels as ordinary E2E deliveries to the user's own mailbox, written through `POST /v1/messaging/self` and read back through the standard pending/ack path. The server sees a delivery class and nothing else about it, and the "all clients acked -> delete immediately" rule performs the cleanup.
- With a **single registered client**, the server stores no sync traffic: it answers a self-message with `stored: false`.

## What is synchronized

There is no event journal, no per-device sequence and no CRDT: the mechanism is
the flat list of `device.*` self-messages below, and the address-book transfer
(`device.contacts-request` / `device.contacts`) is the full-state handover.

Sync data travels as ordinary E2E deliveries to the user's own mailbox, but not
through the same entry point as mail: `POST /v1/messaging/self` is its own
endpoint, with its own delivery class (`device`), and it is deliberately outside
the budget a stranger's contact request is counted against. An account with only
one registered device stores nothing at all - the answer says so in `stored`.

## Enrollment (adding a device)

1. An existing device publishes a one-shot blinded destination and shows two things: a link naming it, `bazarish://pair?v=1&dest=<b33>[&reseed=<https url>]`, and a four-digit code. The link is a QR code and a string; the code is only on the screen. The destination is blinded so a floodfill cannot disclose it to anyone who did not receive the link.
2. The new device dials that destination and sends `GET /hello`, carrying the code in `X-Bazarish-Pair`. The existing device seals the account bundle with that code when pairing starts and holds it in memory; it compares the code in constant time and streams the bundle in the response body only when the code matches. A request with a wrong code is refused with `403` and counted; after ten wrong codes the destination is taken down. A request with no code header is refused with `400` and not counted, so a port scan cannot spend the budget. Nothing leaves the device without the code, so the link may travel over any channel.
3. The bundle is the account backup: the account's kind (`user`, `bot` or `chat`), identity private keys (classical + post-quantum), the account's sealing key, the destination key with its serving sealing key and signing key (`Identity.md`), the delivery secret, the server endpoint, the account's name, settings and avatar, the address book with avatars, and the block list; a `user` account's also carries its presence key and presence destination keys (`Presence.md`). It is never written to disk on either device. Message history is not in it. The kind is fixed when the account is made: the client takes in only a `user` bundle and `bazarish-bot` only a `bot` or `chat` one, and a bundle of another kind is refused before anything is written.
4. A reseed address in the link is what a new device uses to start its own router when it has no network database yet. It is the only coordinate a device can be given before it has an account, so it is an `https` URL and nothing else.
5. The new device authenticates as the user and **registers its client ID**.
6. It asks for the address book (`device.contacts-request`); every other device that has contacts answers.
7. Content delivered and fully acked *before* registration is gone by design and is not replayed. Prior message history does not arrive either: the address book carries no messages, and the message echo is sent as messages are written, not replayed.

## Retirement (removing a device)

- Any device can retire a client ID (lost/stolen device included). The server immediately **re-evaluates ack-completeness** of pending content: blobs that were waiting only on the retired client are deleted.
- **Limitation:** with a shared identity key, a stolen device retains the identity indefinitely. Retirement stops deliveries to it but does not revoke the key. A correspondent can revoke the pass the account holds, which stops the stolen device writing to that one mailbox; the identity itself is unrecoverable.

## Account sync (avatar & contact names)

Account state propagates to the user's other devices as dedicated **self-messages**, sealed to the user's own key and written to their own mailbox (`POST /v1/messaging/self`):

- `device.avatar` - the user's own avatar, re-sent whenever it changes.
- `device.contact-name` - a local contact rename, carrying `peer` (fingerprint) and the new `name`.
- `device.message` - what this device has sent to a contact, so the conversation reads the same on all of them.
- `device.delegation-term` - the term this account delegates its destination for; the shortest choice takes precedence.
- `device.saved` - a message the user kept for themselves, carrying it in `message`. The chat it belongs to is the account's own fingerprint, which no contact can hold.
- `device.saved-clear` - that chat, emptied on every device.
- `device.contact-remove` - a contact removed, carrying `peer`. Every device drops the contact and its conversation; the device that sent it is also the one that revoked their pass.
- `device.contact-block` - `peer` and `blocked`: mail and contact requests from them are dropped as they are read, on every device. The conversation is not touched.
- `device.contact-prefs` - `peer`, `notifications` and `allowCalls`: what one contact may do here.
- `device.contact-accepted` - `peer`: a contact request agreed to. Every device holds the same request, so without this the others continue offering an Agree that has been given - and, more seriously, continue treating the contact as one nothing may be sent to. Receiving it sends the correspondent nothing: the device that agreed carries the exchange.
- `device.chat-pin` - `peer` and `pinned`: a conversation pinned or unpinned, on every device.
- `device.chat-clear` - `peer`: this account's copy of one conversation, emptied. Distinct from the `chat.clear` sent to a correspondent, which asks *them* to empty theirs.
- `device.read` - `peer` and `ts`: this conversation has been read through the
  message sent at that moment. Reading is the account's act, not the device's, so
  a message read on one device is not offered as new on the next one. The mark is
  the message's own **sent-at**, because a local row id is a different number on
  every device, while the sender's stamp is the same everywhere.

  A receiver applies it to its own read high-water **once, on arrival**: it takes
  the newest incoming message it holds at or before that moment and marks through
  it. It is deliberately not kept as a standing rule that anything older counts as
  read - a `sentAt` is written by the sender, and a correspondent who back-dates a
  message would otherwise have it arrive already read and never be seen. A message
  that arrives after the mark is unread regardless of the send time it claims.

  A device sends it while the user reads, coalesced rather than per message: one
  notice per conversation after reading settles, and one when the conversation is
  left.
- `device.account-name` - `name`: the account renamed itself.
- `device.account-prefs` - `acceptCalls` and `sendReceipts`: the account's own answers, which must be the same wherever it is reached from. A notice carries what it knows about; a field it omits is left as it was.
- `device.alias-status` - `status` and `delegation`: the registry's signed answer about the names this account holds, passed on as fetched. A receiver checks it against the built-in resolver root and adopts it only when it is about this account and not older than the answer it holds (`api/FederatedResolve.md`).
- `call.taken` - the device that answered or refused an incoming call, stopping the others ringing (`api/Calls.md`).
- `device.contacts-request` - `device`: the asking device wants the address book. Sent once, automatically, on a device's first sync that reaches the server, and manually from the account window. A device ignores its own request; an account with no other device stores nothing.
- `device.contacts` - `forDevice`, `contacts`: the book, or a part of it. Every other device that has contacts answers, so several answers may arrive; each is taken in turn.
- `device.contact` - `device`, `contact`: one entry of the book, in the same form, sent by the device that has just added a contact (it sent the contact request), before the request leaves. The others hold no copy of that request, so without it they would drop the contact's answer and every later message as from a stranger. It is merged as an answer is.

### The address book

A device that has just joined cannot learn a contact from the network: a correspondent is reached by destination, and nothing turns a fingerprint into one. The account's other devices are the only place the book can come from - so it is asked for, over the same self-channel.

One entry carries what a contact **is**: their fingerprint, their identity keys (what every message of theirs is checked against, `Messages.md`), their sealing key, destination and serving key, the local display name, the per-contact switches, whether this account has issued a delivery pass to them, whether they are blocked, and their avatar.

It carries the contact's **delivery pass** along with everything else about them: a pass is not spent, so every device holds the same one and a device that learns a contact can write to them at once. It still carries nothing that belongs to one device or to one exchange in flight.

Answers are **chunked**: one contact is never split across two messages, and an entry whose avatar would not fit is sent without it (the correspondent sends their avatar again the next time they write; the address they are reached at is what cannot be re-derived). The chunks carry no sequence number and no end marker: the merge is per message and fills gaps, so the order they arrive in does not matter and there is nothing for the asker to wait for. An empty chunk is not sent. A chunk is at most 128 KiB, and the book has no bound of its own: it is stored against the account's one quota like any other self-message, and a chunk refused for quota ends that answer - the asking device keeps what arrived and can ask again from the account window.

Merging **fills gaps only**. A contact this device already has is not overwritten: routing learned from the correspondent themselves is fresher than a copy of it, and a second answer from another device must not undo a rename or a move. For a contact it already has, a device fills only what is empty - keys, routing, name and avatar; the switches, the pass and the block state come only with a contact new to it.

`device.message` is sent **after** the recipient's server has signed for the delivery it echoes, not when the message is written. An echo that goes out first puts on every other device a message that may never arrive - and one copy per retry, all of them shown as still sending - which is an unrecoverable desynchronization. What every device shows is therefore what at least one server holds. `device.message` carries what this device did in a conversation, so every device shows the same one: a message sent, a picture, a voice message, an edit, a deletion, a reaction, a request to a correspondent to clear the chat, and a conversation's auto-delete timer. What a device does *not* echo is its own service messages - read receipts, call signalling and the file-transfer handshake - which are between it and the correspondent. The read state is not among them: it is the account's, not the device's, and it travels as `device.read` above. `device.message` carries the message whole: a picture or a voice message is echoed with its bytes, and no echo is partial. A file announcement is not echoed, because its bytes stay on the device that sent it. Which device a message was read on is not disclosed, but *that* it has been read is the same fact on every device the account is open on - a second device that offered everything again as new was reporting something that is not true.

All are local-only facts that must never reach a contact, so they travel the self-channel exclusively. The routing-key sibling `device.i2p-master` (`{i2pMaster, addressSealing, addressSigning}`: the destination's master key and the private PEM of the serving sealing key and signing key the account issued with it, `Identity.md`) uses the same mechanism. A device sends it after publishing its routing - on raising a new address and on every publication - so the account's other devices hold the address and the exact keys the server operates. The fields of each type are listed above; `device.message` carries `peer`, `device` and `message`.

**`device.i2p-master-request` (`{device, host}`)** is the other half of that pair. An account has **one** address, and it is the address every contact holds; a device that finds its server serving an address it has no keys for asks for them here, naming the address it needs (`host`) and itself (`device`). A device answers with `device.i2p-master` only when it holds the address that was named (an unnamed `host` asks for whatever the answerer has), and the asker takes the one whose address matches what the server serves. A device never answers its own request, and a device that already holds the served address has nothing to ask.

A `device.i2p-master` with the **same** master the device holds brings that address's keys as last published, and the device takes them. One that carries a **different** master than the receiving device holds is not ignored either: the account's address has moved, and a device still announcing the old one in its outgoing routing keeps its contacts writing to an address nobody serves. It is adopted with its keys - replacing the stale master, its keys and the delegation signed by it - **only when the server confirms that address is the one it serves**, so a stale or replayed copy cannot take a working address away from a device.

Until that question is settled a device **publishes nothing**: creating an address of its own and delegating it would take the account's address away from every contact holding it, and none of them would learn a replacement until the user wrote to them again. What a client does when no device answers is its own choice (this implementation asks the user); what the protocol requires is that it not publish silently.

**Every `device.*` type is a message from one of the account's own devices, and only that.** The channel is the account's own mailbox, but a mailbox is where correspondents write too: admission (a delivery pass) is not authorship, so a receiver checks the signer of a device message against its own identity and drops it otherwise - before any field of it is read. Without that check an admitted contact could change an account setting or write into its address book, both of which these types do.

A self-message is named by a **delivery id of its own**, unrelated to the one the same message carries to its recipient (`Messages.md`): the copy kept for the account's devices and the copy the contact receives share no name, so the two servers holding them cannot be matched against each other.

## Quota interplay

An echoed picture or voice message is stored again in the account's own mailbox until every registered device has fetched it, so for devices that are normally online the occupancy is brief. A device that stays offline keeps its echoes there, and enough of them fill the quota (50 MB unless the operator sets another).
