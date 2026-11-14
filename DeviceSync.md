# Multi-Device Synchronization

A user runs several clients (devices); each has keys of its own, certified by the account's root, and is named by its **device id** (`Identity.md`). Contact bookkeeping, delivery passes, and message history must converge across them without giving servers anything new to read.

## Model

- **Own keys per device.** Every device holds its own signing and sealing pair and the certificate the root issued for them, and signs everything it sends with its own keys. Contacts check a device against the account's device list, which the card carries (`Identity.md`, Device certificate). Senders seal to the account's epoch key, which every active device holds, so they never need to know which device will read a message.
- **Primary and secondary.** A primary device holds the root as well; a secondary one does not. Only a primary device adds a device, revokes one, exports the account, makes epoch keys, raises a new address, signs in to a site with the key, changes an alias or deletes the account. A secondary device reads and writes, keeps contacts, presence and the profile, and holds the epoch keys. The service node checks the role for deleting the account and for a new address (`api/ClientApi.md`).
- **Explicit device registry.** Devices are registered via the API (see `api/ClientApi.md`), and leave it only by being revoked. The "delete when all devices received" rule operates on this definite set.
- **Sync channel = own mailbox, one item per device.** Sync data travels as ordinary E2E deliveries to the user's own mailbox, written through `POST /v1/messaging/self` and read back through the standard pending/ack path. Each self-message is written once for every other active device of the account, sealed to that device's own key and addressed to its id; the server lists it to that device alone and deletes it once that device acknowledges it. The server sees a delivery class and a device id and nothing else about it.
- With a **single device**, there is no other device to write to, and nothing is sent.

## What is synchronized

The mechanism is the flat list of `device.*` self-messages below, and the
address-book transfer (`device.contacts-request` / `device.contacts`) is the
full-state handover.

Sync data travels as ordinary E2E deliveries to the user's own mailbox, through
its own endpoint, `POST /v1/messaging/self`, with its own delivery class
(`device`), outside the budget a stranger's contact request is counted against.
An item to self is sealed to the own key of the device it is for, never to an
epoch key: a revoked device keeps the epoch keys it held, and it must not read
what the account's devices tell each other after it is revoked.

## Enrollment (adding a device)

Only a primary device adds one, and the new device's role - primary or secondary - is chosen before pairing starts.

1. An existing device publishes a one-shot blinded destination and shows two things: a link naming it, `https://bazari.sh/#pair?v=1&dest=<b33>[&reseed=<https url>]&key=<key>`, and a four-digit code. The link is a QR code and a string; the code is only on the screen. The destination is blinded so a floodfill cannot disclose it to anyone who did not receive the link. `key` is 32 random bytes as base64url without padding, made for this one showing and kept in memory until pairing ends; a link without it is refused.
2. The new device makes its own signing and sealing keys, dials that destination and sends `POST /hello` whose body is, sealed under the key, the CBOR `{code, c, pq, s}`: the four digits and the public halves of its Ed25519, ML-DSA-65 and sealing keys. The existing device opens the hello and compares the code in constant time. Only when the code matches does it issue the new device's certificate for those keys, in the role chosen, put it in the account bundle, seal the bundle under the key and stream it in the response body. A request with a wrong code under the right key is refused with `403` and counted; after ten wrong codes the destination is taken down. A request whose body does not open under the key is refused with `400` and not counted: without the key there is no guess to count, so a port scan cannot spend the budget. The code is the second factor - the key alone gets no bundle - so the link may travel over any channel.

   ```
   seal(label, m) = nonce(12, random) || ChaCha20-Poly1305(HKDF-SHA256(key, salt = "", info = label), nonce, m)
   hello:   seal("bazarish pairing hello v1", CBOR {code, c, pq, s})
   bundle:  seal("bazarish pairing bundle v1", <the account bundle>)
   ```

   The key reaches the new device in the link or the QR code, so a gateway that carries either device's streams sees neither the code, the keys nor the bundle, and pairing works through one. The private halves of the new device's keys never leave it: it adds them to the bundle it received.
3. The bundle is the account backup: the account's kind (`user`, `bot` or `chat`), the root's private keys for a primary device and not for a secondary one, the new device's certificate, the account's device list with every active device's public sealing key, the ring of epoch keys, the destination key with its serving sealing key and signing key (`Identity.md`), the delivery secret, the server endpoint, the account's name, settings and avatar, the address book with avatars, and the block list; a `user` account's also carries its presence seed (`Presence.md`). It is never written to disk on either device. Message history is not in it. The kind is fixed when the account is made: the client takes in only a `user` bundle and `bazarish-bot` only a `bot` or `chat` one, and a bundle of another kind is refused before anything is written.
4. A reseed address in the link is what a new device uses to start its own router when it has no network database yet. It is the only coordinate a device can be given before it has an account, so it is an `https` URL and nothing else.
5. The new device publishes the card with itself in the device list, **registers itself** on the server, and tells the account's other devices (`device.devices`) and every contact (`account.devices`) about itself.
6. It asks for the address book (`device.contacts-request`); every other device that has contacts answers, and a primary device also sends it every epoch key it holds (`device.epochs`).
7. Content delivered and fully acked *before* registration is gone by design and is not replayed. Prior message history does not arrive either: the address book carries no messages, and the message echo is sent as messages are written, not replayed.

**From a file.** An exported backup is the same bundle sealed with a password (the CMS password envelope, `Identity.md`), and only a primary device exports one, for a role chosen the same way. A **primary** backup carries the root; the device that imports it makes its own keys and certifies them as primary itself. A **secondary** backup cannot do that, so the exporting device makes the new device's keys and certificate itself and puts them, private halves included, in the backup. Such a backup is for one device: its id is taken by the first copy that registers, and a second import of it is refused by the server with `409 DEVICE_TAKEN` (`api/ClientApi.md`).

## Revocation (removing a device)

- Only a primary device revokes, and never itself; a lost or stolen device is revoked from any primary device that remains. Revocation is the root's signed revocation list (`Identity.md`, Revocation list): the revoking device signs the next list, makes new epoch keys under it and hands them only to the devices that stay, withdraws what its outbox still held for the revoked device, publishes the card, and sends the new device list and the new epoch keys to the account's other devices and to every contact.
- The server reads the list from the published card. It refuses the revoked device with `403 DEVICE_REVOKED`, unregisters it and **re-evaluates ack-completeness** of pending content: blobs that were waiting only on the revoked device, and every item to self for it, are deleted.
- A contact refuses whatever a revoked device signs once it holds the new list: from the `account.devices` letter in its mailbox, or from the card. Until then it accepts that device's messages.
- A revoked device keeps the epoch keys it held, and with them it can open mail sealed to those epochs - mail the server refuses to hand it. It does not receive the keys made at the revocation or after it.
- A revoked device is not readmitted. The same hardware joins again as a new device, with new keys.

## Account sync (avatar & contact names)

Account state propagates to the user's other devices as dedicated **self-messages**, sealed to each device's own key and written to their own mailbox (`POST /v1/messaging/self`):

- `device.avatar` - the user's own avatar, re-sent whenever it changes.
- `device.contact-name` - a local contact rename, carrying `peer` (fingerprint) and the new `name`.
- `device.message` - what this device has sent to a contact, so the conversation reads the same on all of them.
- `device.delegation-term` - the term this account delegates its destination for; the shortest choice takes precedence.
- `device.saved` - a message the user kept for themselves, carrying it in `message`. The chat it belongs to is the account's own fingerprint, which no contact can hold.
- `device.saved-clear` - that chat, emptied on every device.
- `device.contact-remove` - a contact removed, carrying `peer`. Every device drops the contact and its conversation; the device that sent it is also the one that revoked their pass.
- `device.contact-block` - `peer` and `blocked`: mail and contact requests from them are dropped as they are read, on every device. The conversation is not touched.
- `device.contact-prefs` - `peer`, `notifications` and `allowCalls`: what one contact may do here.
- `device.contact-accepted` - `peer`: a contact request agreed to. Every device holds the same request; on this notice the others stop offering Agree and treat the contact as one that may be written to. Receiving it sends the correspondent nothing: the device that agreed carries the exchange.
- `device.chat-pin` - `peer` and `pinned`: a conversation pinned or unpinned, on every device.
- `device.chat-clear` - `peer`: this account's copy of one conversation, emptied. Distinct from the `chat.clear` sent to a correspondent, which asks *them* to empty theirs.
- `device.read` - `peer` and `ts`: this conversation has been read through the
  message sent at that moment. Reading is the account's act, not the device's, so
  a message read on one device is not offered as new on the next one. The mark is
  the message's own **sent-at**, the sender's stamp, which is the same on every
  device.

  A receiver applies it to its own read high-water **once, on arrival**: it takes
  the newest incoming message it holds at or before that moment and marks through
  it. A message that arrives after the mark is unread regardless of the send time
  it claims, so a correspondent cannot back-date a message into the read state.

  A device sends it while the user reads, coalesced: one
  notice per conversation after reading settles, and one when the conversation is
  left.
- `device.account-name` - `name`: the account renamed itself.
- `device.account-prefs` - `acceptCalls` and `sendReceipts`: the account's own answers, which must be the same wherever it is reached from. A notice carries what it knows about; a field it omits is left as it was.
- `device.alias-status` - `status` and `delegation`: the registry's signed answer about the names this account holds, passed on as fetched. A receiver checks it against the built-in resolver root and adopts it only when it is about this account and not older than the answer it holds (`api/FederatedResolve.md`).
- `call.taken` - the device that answered or refused an incoming call, stopping the others ringing (`api/Calls.md`).
- `device.epochs` - `epochs`: the private halves of epoch keys, `[{n, g, by, pem}]`. Sent by a primary device when it makes a key, after a revocation and in answer to `device.contacts-request`; taken only from a primary device, and only keys a primary device made (`Identity.md`, Epoch keys).
- `device.devices` - `devices` and `revocation`: the account's device list, each active device's certificate with its public sealing key, and the newest revocation list. Sent by a device new to its server and after a revocation, carrying the author's own certificate. A receiver checks each sealing key against its certificate's digest, refuses a list older than the one it holds, and from then on writes only to the devices it names.
- `device.contacts-request` - no fields: the device that signed it wants the address book. Sent once, automatically, on a device's first sync that reaches the server, and manually from the account window; an account with no other device sends nothing. A primary device answers it with its epoch keys as well (`device.epochs`).
- `device.contacts` - `contacts`: the book, or a part of it, written to the asking device alone. Every other device that has contacts answers, so several answers may arrive; each is taken in turn.
- `device.contact` - `contact`: one entry of the book, in the same form, sent by the device that has just added a contact (it sent the contact request), before the request leaves. The others hold no copy of that request; this entry lets them accept the contact's answer and every later message. It is merged as an answer is.

### The address book

A device that has just joined takes the address book from the account's other devices, asking for it over the same self-channel: a correspondent is reached by destination, and a fingerprint does not lead to one.

One entry carries what a contact **is**: their fingerprint, their device list - the certificates and revocation list every message of theirs is checked against (`Messages.md`) -, their epoch keys and the last epoch this account sent them its own, their destination, serving key and card key, the local display name, the per-contact switches, whether this account has issued a delivery pass to them, whether they are blocked, their avatar, and for a `user` contact their presence schedule and the last day of the schedule sent to them (`Presence.md`).

It carries the contact's **delivery pass** along with everything else about them: a pass is not spent, so every device holds the same one and a device that learns a contact can write to them at once. It still carries nothing that belongs to one device or to one exchange in flight.

Answers are **chunked**: one contact is never split across two messages, and an entry whose avatar would not fit is sent without it (the correspondent sends their avatar again the next time they write; the address they are reached at is what cannot be re-derived). The chunks carry no sequence number and no end marker: the merge is per message and fills gaps, so the order they arrive in does not matter and there is nothing for the asker to wait for. An empty chunk is not sent. A chunk is at most 128 KiB, and the book has no bound of its own: it is stored against the account's one quota like any other self-message, and a chunk refused for quota ends that answer - the asking device keeps what arrived and can ask again from the account window.

Merging **fills gaps only**. A contact this device already has is not overwritten: routing learned from the correspondent themselves is fresher than a copy of it, and a second answer from another device must not undo a rename or a move. For a contact it already has, a device fills only what is empty - routing, name and avatar; the switches, the pass and the block state come only with a contact new to it. Keys follow their own rules: epoch keys of both copies are kept while their epochs last, the later of the two last epochs sent is kept, and the device lists are merged as any device list is (`Identity.md`, Revocation list). So is the presence schedule: the copy that starts no earlier wins, and the later of the two last days sent is kept.

`device.message` is sent **after** the recipient's server has signed for the delivery it echoes, not when the message is written, so what every device shows is what at least one server holds. `device.message` carries what this device did in a conversation, so every device shows the same one: a message sent, a picture, a voice message, an edit, a deletion, a reaction, a request to a correspondent to clear the chat, and a conversation's auto-delete timer. What a device does *not* echo is its own service messages - read receipts, call signalling and the file-transfer handshake - which are between it and the correspondent. The read state is not among them: it is the account's, not the device's, and it travels as `device.read` above. `device.message` carries the message whole: a picture or a voice message is echoed with its bytes, and no echo is partial. A file carried inside its message is echoed the same way, with its bytes; a file announcement is not, because its bytes stay on the device that sent it. Which device a message was read on is not disclosed, but *that* it has been read is the same fact on every device the account is open on.

All are local-only facts that must never reach a contact, so they travel the self-channel exclusively. The routing-key sibling `device.i2p-master` (`{i2pMaster, addressSealing, addressSigning}`: the destination's master key and the private PEM of the serving sealing key and signing key the account issued with it, `Identity.md`) uses the same mechanism. A device sends it after publishing its routing - on raising a new address and on every publication - so the account's other devices hold the address and the exact keys the server operates. The fields of each type are listed above; `device.message` carries `peer` and `message`, and the device that sent it is its author.

**`device.i2p-master-request` (`{host}`)** is the other half of that pair. An account has **one** address, and it is the address every contact holds; a device that finds its server serving an address it has no keys for asks for them here, naming the address it needs (`host`). A device answers with `device.i2p-master` only when it holds the address that was named (an unnamed `host` asks for whatever the answerer has), and the asker takes the one whose address matches what the server serves. A device never answers its own request, and a device that already holds the served address has nothing to ask.

A `device.i2p-master` with the **same** master the device holds brings that address's keys as last published, and the device takes them. One that carries a **different** master than the receiving device holds means the account's address has moved, and a device still announcing the old one in its outgoing routing keeps its contacts writing to an address nobody serves. It is adopted with its keys - replacing the stale master, its keys and the delegation signed by it - **only when the server confirms that address is the one it serves**, so a stale or replayed copy cannot take a working address away from a device.

Until that question is settled a device **publishes nothing**: creating an address of its own and delegating it would take the account's address away from every contact holding it, and none of them would learn a replacement until the user wrote to them again. What a client does when no device answers is its own choice (this implementation asks the user); what the protocol requires is that it not publish silently.

**Every `device.*` type is a message from one of the account's own devices, and only that.** The channel is the account's own mailbox, but a mailbox is where correspondents write too: admission (a delivery pass) is not authorship, so a receiver checks that a device message is signed by an active device of its own account and drops it otherwise - before any field of it is read. Without that check an admitted contact could change an account setting or write into its address book, both of which these types do.

A self-message leaves through the same outbox as a message to a contact (`Messages.md`): while the account's own server does not answer it waits, and it goes when that server answers a request again.

A self-message is named by a **delivery id of its own** for each device it is written to, unrelated to the one the same message carries to its recipient (`Messages.md`): the copy kept for the account's devices and the copy the contact receives share no name, so the two servers holding them cannot be matched against each other.

## Quota interplay

An echoed picture or voice message is stored again in the account's own mailbox once for every other device, each copy until its device has fetched it, so for devices that are normally online the occupancy is brief. A device that stays offline keeps its echoes there for up to 30 days (`api/Storage.md`), and enough of them fill the quota (50 MB unless the operator sets another).
