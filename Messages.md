# Messages & Content Types

## Types

A message declares its type, and the type determines the recipient's handling of
it; neither the file name nor a sender-supplied MIME value has that role:

| `type` | What it carries | What the recipient does |
|---|---|---|
| `text` | the text | shows it |
| `file` | a transfer's metadata (name, size, sha256, mime) | offers to fetch it; the bytes travel client to client |
| `image` | the picture itself, bytes in the envelope | decodes it, keeps it in the account, draws it |
| `audio` | Opus audio, bytes in the envelope | keeps it in the account, plays it on a press |
| `contact.request`, `reaction`, `receipt`, `call.*`, `device.*` | as their names say | as documented elsewhere |

`image` carries the picture **inside the message**: there is no announcement and
nothing to request, so it arrives once, with the message, and it arrives even if
the sender goes offline the moment after sending - the mailbox holds it like any
other message. The sender shrinks it first (long edge 1600, re-encoded, at most
256 KiB) so that it and the envelope around it stay inside the 512 KiB message
limit.

Before it is rendered it must decode as an image; a rejection by the decoder is
final. Such a message is reported as broken. It is not reclassified as a file:
that would be a fallback, the client reinterpreting what it received. The bytes
are held in the account database, encrypted with the remainder of its contents,
and are exported by an explicit save action.

A `file` is the other path and keeps the announcement: the metadata travels in
the message, the bytes travel client to client on demand, and the size has no
limit other than the time the transfer takes.

The message that announces a transfer is a small message like any other: the
progress bar belongs to the transfer that follows it, not to the announcement.

## Encoding

The body inside the seal is **CBOR**, not JSON text: the same document with the
same field names, but binary values travel as themselves. That is what lets a
picture or a voice message travel inside the message rather than being announced
and fetched. Base64 would add a third to every such payload (measured at 1.333x),
and compression yields nothing, both JPEG/PNG and Opus being already
entropy-coded (gzip on a JPEG: 0.986x; on a PNG: 1.000x).

A voice message is Opus in the call format - 48 kHz mono, 20 ms frames, each
length-prefixed - and nothing further. No container is used: both ends implement
this protocol, so an Ogg header would carry only parameters already fixed by the
specification.

## Size

One message carries at most **512 KiB** of payload (`kMaxMessagePayloadBytes`).
The server checks it before storing anything - on a delivery, only after the
pass has been accepted - and answers `MESSAGE_TOO_LARGE`, a final refusal. The
client checks the same number on the same bytes before sending: the payload as
it leaves, signed and sealed. A message over it is refused locally, with a
request to send it as a file, and nothing is delivered.

Anything larger is a file: `sendFile` puts only its metadata (name, size,
sha256, mime) in the message and the bytes travel client to client over a
one-time destination, so a file never enters a mailbox and its size has no limit
other than the time the transfer takes. Pictures sent from the composer are
scaled and re-encoded to a few hundred KiB before they are sent.

## Principle: two independent layers

A message has **two** type discriminators that must never be conflated:

1. **Delivery class** - visible to the destination server: inside the sealed
   delivery envelope, or given by the endpoint a self-delivery is posted to. A
   closed set of exactly three values. It selects the server's *admission
   policy*, nothing else.
2. **Content type** - visible only to the recipient, inside the end-to-end
   encrypted payload. An **open** registry. The server, facades and federation
   relays never see it.

The resulting invariant:

> Adding a new kind of content (a file, a picture, a voice note, a call signal,
> a read receipt) touches **only the client protocol and the content-type
> registry below**. The server, the storage backend and the federation relay
> require **no change** and never learn what kind of content they carried.

A picture is not a server concept. It is a `content` delivery whose encrypted
payload happens to have `type: "image"`. The server stores and relays the
identical opaque envelope it would for a text line or a call invite.

## Layer 1 - delivery class (server-visible)

Carried as the `class` field of the sealed delivery envelope (see
`api/Federation.md`) for `contact` and `content`; `device` never travels in an
envelope - the server assigns it to what an account stores through
`POST /v1/messaging/self`. Three values, and this set is frozen - content types
never appear here:

| `class` | Admitted by | Server policy |
|---|---|---|
| `contact` | nothing | Bootstrap. Size-capped (`kMaxContactRequestBytes`), rate-limited per serving destination. The only path into a mailbox that presents nothing. |
| `content` | the recipient's delivery pass (required) | Looked up, not consumed (see `Contacts.md`). The path for every established-contact message, whatever its content type. |
| `device` | an authenticated request from one of the account's registered devices | An account writing to its own other devices, through `POST /v1/messaging/self` rather than through federation. Outside the budget a stranger's request is counted against, and stored only when the account has more than one device. |

There is no separate quota for any of them: a mailbox has one, and every class is
stored against it.

The server branches on `class` and nothing else: it validates the mailbox,
deduplicates by `deliveryId` (below), enforces the pass (for
`content`) or the size
cap (for `contact`), and stores the opaque payload. The pending list returns
the `class` back to the recipient - an admission-class hint the server already
knew, never the content type.

Sealed envelope as the server sees it:

```
{ "class": "content" | "contact",
  "mailbox": <recipient fingerprint>,
  "deliveryId": <delivery id>,
  "pass": "<base64 delivery pass>"      // present iff class == "content"
}
```

`pass` is the one value the recipient issued to this correspondent, whatever the
message's size (`Contacts.md`, pass lifecycle). The recipient's server hashes it
and looks it up; it never derives anything from the payload, whose only visible
property is its length - and that is padded to a step of the ladder before the
seal, so what a mailbox's server measures is the step.

### Two identifiers, and which is which

![How a message is wrapped, what travels on each hop, and where each identifier is created and read](img/identifiers.svg)

A message and an envelope are named separately, and the two names are known to
different sides. The **e2e id** is what the two *ends* call the message; the
**delivery id** is what the sending *client* and the recipient's *server* call one
envelope carrying it:

| | **e2e id** (`id` in the layer-2 envelope) | **delivery id** (`deliveryId` everywhere) |
|---|---|---|
| Names | the message | one envelope delivered to a server |
| Who knows it | the two clients, and the sender's other devices - it is sealed | the sending client and the recipient's server - it is in the clear |
| Used for | replies, reactions, edits, read receipts, deletes; the recipient's client recognising a redelivery | the mailbox blob name, the recipient server's dedup and seen-set, and what its confirmation is signed over |
| Chosen by | the sending client, at random, once per message | derived by the sending client: `HMAC(account secret, e2e id + mailbox)` |

The delivery id being **derived** is what makes sending the same message again
idempotent: the envelope keeps its name, so a recipient's server that already
stored it recognises it and stores nothing twice. Binding
it to the mailbox is what keeps it from linking anything: the copy a sender keeps
for their own devices and the copy the recipient gets are the same message under
two unrelated names, so the two servers holding them have nothing to match on.
Keying it with a secret held only by the sender means no other party can determine what
a message of theirs will be called - and so cannot poison a recipient's seen-set
with that name before the real envelope arrives.

## Layer 2 - content type (end-to-end only)

The encrypted payload (sealed to the recipient's user sealing key) is a
versioned envelope:

```
{ "v": 1,
  "type": <content-type string>,
  "id":   <e2e id: what this message is called between the two ends>,
  "from": <sender fingerprint>,
  "sentAt": <unix milliseconds>,   // client send clock; drives ordering and the displayed time
  "replyTo": <id>,            // optional
  "routing": { "dest": <sender b32>, "servingKey": <base64 SPKI> },   // contact.routing only
  "auth": { "c": <Ed25519 signature, CBOR bytes>,
            "p": <ML-DSA-65 signature, CBOR bytes>,
            "kc": <author's Ed25519 public key, SPKI DER - contact request only>,
            "kp": <author's ML-DSA public key, SPKI DER - contact request only> },
  ... type-specific fields ...
}
```

**`auth` - who wrote it.** Nothing else in the path answers that question. A
delivery pass is *admission*: it says somebody the recipient issued a pass to
put this in their mailbox, not which of their correspondents it was. The seal
says only who may read. `from` on its own is a claim by the sender, so every
envelope carries the author's own proof: their identity public keys and both
signatures over the CBOR of the envelope with the `auth` key removed. Hybrid,
like every identity statement here - both signatures must verify.

A receiver determines the author from the signature, not from `from`: the keys
give a fingerprint, that fingerprint must be what `from` claims, and an envelope
that fails either test is consumed before it reaches a chat, a contact entry or
the user. Without this an admitted contact could put another contact's name on a
message and have it filed in that other conversation.

The keys travel in one place only: the **contact request**. Adding a contact
goes through their card, so the adder holds the added party's keys before the
first word is sent, and the added party takes the adder's from the request that
started the dialog. Everywhere else the block is the two signatures alone, and the reader checks them against the keys it kept when the
correspondent introduced themselves - from their card when it added them, or
from the request that started the dialog. A message from somebody it has no keys
for is one it cannot check, and it is dropped exactly like a message from a
stranger.

Measured: **3388 bytes** for the two signatures, **5417** when the keys are
included. Every message pays the first figure - a reaction and a read receipt as
much as a letter. That is what the contact-request cap is
measured against (`api/ClientApi.md`): a request carries no pass and no card, so
its signature is the only thing that names the sender.

**`routing` - where to answer, sent once when it changes.** A destination is not
permanent: raising a new I2P key changes the address, and the account issues a
new serving sealing key with it. Whenever the pair a card carries changes, the
account sends every contact one
`contact.routing` message carrying `{dest, servingKey}`, and a receiver whose
stored pair differs adopts the new one and persists it. It is carried by that
message alone - an ordinary letter carries no `routing`, because the pair changes
rarely and adding 227 bytes to every reaction and read receipt to announce it
was not justified. The sender's queue keeps retrying a contact it could not reach,
so the update is not lost to one offline device.

The resulting rule is: **after changing an address, tell everyone.** No other
mechanism restores a pair whose stored routing is stale, because the only path to
a contact is the destination currently held for them, and a request for a new
address would be sent to the address that is no longer served. Sending outward
does not depend on one's own destination, so a moved account can always reach its
contacts; what it must not do is lose the obligation, which is why the fan-out is
a queue in the profile database that survives a restart and is cleared one
contact at a time, on success.

The queue is written once the server has taken the new address and keys and
the card naming them is published (`api/ClientApi.md`) - and it holds every
contact, plus the alias registry when
this account holds a name that resolves. It is
tried once immediately, once again whenever the account comes back online, and
manually. The two kinds of entry report differently, because the remedy differs:

- a **contact** that could not be reached leaves a notice in that contact's own
  chat, saying the new details did not arrive, with a **Try again** beside it.
  One notice per contact, not one global warning, and it is removed when the
  update is delivered;
- the **alias registry** is asked again every 10 seconds, silently, until it
  takes the new descriptor. It is one global service whose permanent absence is
  out of scope, and the person has nothing to decide about it.

A client with no chat window (the daemon) lists the pending entries on its own
face, with the same retry.

### Content-type registry (v1)

| `type` | Delivery class | Purpose / payload |
|---|---|---|
| `text` | content | `text` - a plain text line. |
| `contact.routing` | content | A routing change pushed to an established contact: the `routing` block, on a message with nothing else in it. The only message that carries one. |
| `device.chat-pin` | device | One conversation pinned or unpinned, on every device of the account. |
| `contact.request` | contact | Carries a `bootstrap` block (`kind`, `sealing` - the same account sealing key the card publishes -, `dest`, `servingKey`, `pass`, and for a `user` `presenceKey` and `presenceDest`) plus `text`. The first message; see `Contacts.md`. |
| `contact.accept` | content | Agreeing to a received `contact.request`: an empty-bodied message whose first-reply `bootstrap` delivers our descriptor + our pass to the requester, making the pair mutual contacts. The requester's client completes adding the contact automatically and surfaces a note. See `Contacts.md`. |
| `file` | content | An announcement: `name`, `size`, `sha256` and `mime`. The bytes stay on the sender's disk until the recipient asks for them. See `api/FileTransfer.md`. |
| `image` | content | A picture, carried inline: the bytes travel in the envelope rather than being fetched. Its body block is named `image`. |
| `audio` | content | A voice note, carried inline like `image`. Its body block is named `audio` and holds `durationMs`, `size` and the Opus bytes in `data`; no waveform is sent - the receiver draws it from the audio. |
| `receipt` | content | `ref` (the read message's `id`). A **read** receipt - the recipient's client sends it only on a genuine read (the chat is open, the app window focused, and the message scrolled into the visible area), confirming the "green" delivery state below. `ref` is the newest read incoming id, so it acknowledges that message and every earlier one **the sender got as far as the recipient's server** (a read high-water); one that never arrived is not carried up by it. While a device is taking in the backlog it found on coming online, it sends no receipt and no `device.read`; once the backlog is in, it sends at most one receipt per conversation, and only for a read newer than what the account's other devices reported (`device.read`), since those devices have answered for theirs. Optional/toggleable. **Never sent about a contact request**: reading a request is not answering it, and a requester who learns that a stranger's client is running and has opened their message has been told something they never agreed to be told. Until the request is accepted, the requester's own plate stops at *delivered*. |
| `reaction` | content | A reaction to a message: `ref` (the target message's `id`), `text` (**at most 4 characters and 20 bytes** - enough for an emoji with a modifier or a joined pair; **empty removes** the reactor's reaction). Both limits are checked when sending and again on receipt: a peer's client is not the receiver's to trust, and an oversized one is dropped rather than rendered on someone's message. One reaction per user per message - a new one overwrites the old at every recipient. A single sealed delivery to the peer. Never a chat bubble - clients render it as a chip on the target message. See *Reactions*. |
| `bot.command` | content | A command invocation aimed at a bot: `command` (name, no leading slash), `args` (raw argument string). See *Bots & interactive messages*. |
| `bot.callback` | content | An inline-keyboard button press: `data` (the button's payload), `ref` (the `id` of the keyboard message). See *Bots & interactive messages*. |
| `bot.menu` | content | The commands a bot offers a contact: `menuId` (a random unsigned 32-bit integer) and `items` of `{ title, command }`. A client names the menu it holds in the `menuId` of what it sends back. A silent service message. See `Bots.md`. |
| `edit` | content | An in-place edit of a message the sender previously sent: `ref` (the target message's `id`), `text` (new text), `keyboard` (new inline keyboard, `[]` to clear). The recipient replaces its copy. Applied only to a message sent by `from`. See *Editing and deleting messages*. |
| `delete` | content | A delete-for-everyone of a message the sender previously sent: `ref` (the target message's `id`). The recipient removes its copy with **no tombstone** left behind. Scoped, like `edit`, to a message sent by `from`. See *Editing and deleting messages*. |
| `chat.clear` | content | Asks the peer to clear the **whole** one-to-one conversation with the sender (no `ref`). The recipient's client wipes its transcript with `from` and leaves a single "cleared the chat" notice. The chat itself remains. The "clear for everyone" choice in the chat-info panel. |
| `chat.timer` | content | Sets the one-to-one conversation's auto-delete timer for both sides: `seconds` is one of 0 (off), 3600, 86400, 604800 or 2592000, and any other value is ignored. The recipient's client takes the same timer and leaves a "set the auto-delete timer" notice. See *Auto-delete timer*. |
| `avatar` | content | The sender's avatar pushed to a contact: `avatar` = `{ mime, data }`, `data` being the bytes of a square PNG/JPEG (a CBOR byte string, as in an `image`) compressed client-side to at most **500 KB**. A silent service message - stored as that contact's picture, never shown as a chat bubble. Sent once the dialog is mutually established and re-sent to every contact when the avatar changes. See *Avatars and display names* and `Contacts.md`. |
| `device.avatar` | device | Self-sync of the user's own avatar to their other devices: the same `avatar` block, sealed to the user's own key and stored through `POST /v1/messaging/self`. See `DeviceSync.md`. |
| `device.contact-name` | device | Self-sync of a local contact rename to the user's other devices: `peer` (fingerprint), `name`. A purely local label, never sent to the contact. See `DeviceSync.md`. |
| `device.i2p-master` | device | Self-sync of the user's own I2P master key to their other devices: `i2pMaster` (base64), sealed to the user's own key and stored through `POST /v1/messaging/self`, so every device keeps the same personal address. Applied only when `from` is the user themselves. See `DeviceSync.md`, `Identity.md`. |
| `call.invite` | content | Places an audio call: `callId`, `media` (`audio`), `codec` (`opus`), `dest` (the caller's media b32), `key` (base64 per-call media key). See `api/Calls.md`. |
| `call.accept` | content | Accepts a call: `callId`, `dest` (the callee's media b32). See `api/Calls.md`. |
| `call.decline` | content | Rejects a call: `callId`, `reason`. See `api/Calls.md`. |
| `call.end` | content | Hangs up / cancels a call: `callId`. See `api/Calls.md`. |
| `call.ring` | content | A device of the callee's is showing the call: `callId`. What turns the caller's "invitation delivered" into "their device is ringing". See `api/Calls.md`. |
| `group.state` | content | From a group host only: no body, only the `group` block, sent when the recipient's `write` or `admin` changes and the host has nothing else to send. See `Groups.md`. |
| `group.reactions` | content | From a group host only: `reactions` = `[{ref, counts: {emoji: n}}]`, the reaction counts of relayed posts, batched, leaving out the recipient's own reaction. See `Groups.md`. |
| `call.taken` | device | Self-sync: the device that answered or refused an incoming call, so the account's other devices stop ringing - `callId`, `device`. See `api/Calls.md`, `DeviceSync.md`. |

### Forwarding

A message can be passed on to another chat. The forward is a **new message of the
forwarder's own**: their id, their signature, their delivery - carrying the same
content and one additive field:

```
"forwarded": true
```

Nothing else is added. The forward names no original sender, carries no chain of
who passed it on before, and proves nothing about who wrote the content: a
client can set the field on anything it sends. Recipients render it as a
"Forwarded" mark on the message, which is the whole of its meaning: the content
was passed on rather than typed here.

What travels is what the forwarding device holds: text always; a voice message and
a picture, which travel inside the message; a file only while its bytes are still
on that device, since a file is announced from a path and served from it.

One further field is additive and may be carried by any content message:

- **`replyTo`** - the `id` of the message this one replies to. The recipient
  renders a quote of that message and, when it holds the original locally, a
  click jumps to it. A dead reference (the original was never received or was
  deleted) renders as a non-clickable quote. See *Replies*.

A group host (`Groups.md`) adds three more, honoured only from it:

- **`group`** - the block every envelope of a group host carries: `kind`,
  `members`, `write`, `admin`, `at`.
- **`author`** - on a relayed post, the author's name in the chat. Ignored on a
  message without a `group` block.
- **`contactable`** - on a relayed post, whether its author accepts contact
  requests.

The registry is **open**: new rows are added here and implemented in clients.
No server, facade, federation or storage change accompanies a new row.

### Bots & interactive messages

A bot is not a server concept either: it is an **ordinary identity** that
registers on a server like any client, accepts contacts and exchanges the same
`content`/`contact` envelopes. "Botness" lives entirely in Layer 2 - four
client-only pieces, all forward-compatible:

**Inline keyboard.** Any content message (typically `text`) may carry an
optional `keyboard` field: rows of buttons rendered under the message. A client
that does not understand it still renders the message body, so the field is
additive.

```
"keyboard": [
  [ { "text": "Ping", "data": "ping" }, { "text": "Time", "data": "time" } ],
  [ { "text": "Help", "command": "help" } ]
]
```

A button carries a label (`text`) and exactly one action:

| Field | Tap result |
|---|---|
| `data` | Sends a `bot.callback` to the message's sender with this `data` and `ref` = the keyboard message's `id`. |
| `command` | Sends a `bot.command` to the sender with this command name. |

**Commands.** A `bot.command` is the structured form of a `/name args` line;
clients dispatch both the same way. It lets a bot expose a command interface
without parsing free text (though a bot may do either).

**Callbacks.** A `bot.callback` reports a button press back to the bot, which
acts on the `data` and may reply (e.g. with a new message or a fresh keyboard).

**Menu.** A `bot.menu` lists the commands a bot offers, shown beside the
conversation, and every message a client answers it with names the menu it
holds.

Because all four are `content`-class deliveries, the server, facades,
federation and storage are unchanged and never learn that a conversation
involved a bot. The full model is in `Bots.md`.

### Editing and deleting messages

A sender can revise a message it already sent with an `edit` (above): the
recipient finds the message whose `id` equals `ref` and replaces its `text` and
`keyboard` in place, marking it edited. It is the same primitive for two cases:

- **Bots** edit their own keyboard message on a callback - the behaviour in which
  a message changes in response to a button press - by sending an `edit` whose
  `ref` is the callback's `ref`.
- **Users** edit their own line: the client updates its local copy and sends an
  `edit` so the peer updates theirs.

A `delete` is the sibling primitive: the recipient removes the message whose `id`
equals `ref` outright, leaving **no tombstone** (a true delete-for-everyone, not
an empty edit). The client always removes its own copy; for a one-to-one message
it sends the `delete` so the peer removes theirs too.

A client applies an `edit` or a `delete` **only** to a message sent by the `from`
party (for a recipient, an incoming message from that peer); it must not
let a peer rewrite or remove messages it did not author. History of prior
revisions is not retained by the protocol (a client may keep its own).

### Replies

Any content message may carry a `replyTo` field naming the `id` of the message it
answers. It is a soft reference: the body is not duplicated. The recipient
resolves `replyTo` against its own transcript for that conversation and, if the
original is present, renders a clickable quote (author + a one-line preview) that
jumps to it; if absent, a static "(original message unavailable)" quote.
`replyTo` is protected only by the
end-to-end seal of the message that carries it - it is not separately signed,
because each recipient receives its own sealed copy.

### Reactions

A `reaction` sets the sender's emoji on a target message (`ref`).
Each user has at most one reaction per message: a new emoji replaces the previous
one, and an **empty** emoji removes it (a client may implement selecting the
current emoji again as removal). Reactions are state rather than history:
recipients store them keyed by `(conversation, target, reactor)` and render them
on the target message, never as a separate entry in the conversation. A reaction is a single
sealed delivery to the peer.

### Clearing a conversation

A user may clear a one-to-one conversation in two ways. **Only for me** wipes the
local transcript and leaves the chat row in place. **For everyone** additionally
sends a `chat.clear` to the peer; the peer's client honours it, wiping its own
transcript and leaving a single "cleared the chat" notice. This is a cooperative
request (the peer's client chooses to honour it), distinct from `delete`, which
removes a single message. Deleting a **contact** is a separate, local-only,
irreversible action: it removes the contact and the whole conversation from the
deleting device (the peer is not told).

### Auto-delete timer

A one-to-one conversation may carry an auto-delete timer. Either side sets it with
`chat.timer`, the other side takes the same value, and both show a notice of who set
it. Each device deletes **its own copy only**: nothing is sent when a message goes.
A message is counted from when it was sent on the sender's devices and from when it
was read on the recipient's, under the timer in force at that moment, so turning the
timer on does not touch what was already sent or read. Text, pictures, voice messages
and files expire; service notices do not. A client checks for expired messages when
a conversation is opened and from time to time across every conversation. Group
conversations carry no timer.

### Avatars and display names

A contact's **display name** and **avatar** are local, recipient-side facts.
Neither is part of the message stream, and a peer is never an authority on the
label or picture a recipient assigns to it.

**Display name.** A name is captured **once, at add time** and kept as a local
label:

- From an **invite**, the descriptor carries an optional `name` (percent-encoded,
  `api/FederatedResolve.md`) the inviter advertises; the adder adopts it.
- From an **alias**, the alias the user typed becomes the name.

Thereafter the name is **static**: it is never re-fetched and never transmitted to
the peer - there is no "my name changed" message. The user may rename a contact
at any time, but that rename is **local only**, mirrored to the user's *own* other
devices via a `device.contact-name` self-message (`DeviceSync.md`) and to no one
else.

**Avatar.** A user's own avatar is chosen locally, cropped **square** and
compressed (PNG/JPEG) to at most **500 KB** before it leaves the device (the
protocol rejects a larger one). Its bytes are a CBOR byte string, as a picture's
and a voice message's are, so an avatar at that limit fits the 512 KiB message
limit signed and sealed. It travels as a silent `avatar` service message -
stored as the sender's picture, never rendered as a chat bubble:

- **On establishing a dialog** - exchanged symmetrically the moment the dialog is
  mutually established: the reply that accepts a contact request carries the
  accepter's avatar, and the original requester sends theirs on receiving that
  acceptance. A bare `contact.request` never carries an avatar and never triggers
  an automatic avatar reply, so nothing of a user's picture reaches a party who
  has not yet been accepted (`Contacts.md`).
- **On change** - re-sent to every established contact, best-effort, with no
  user-visible send error.
- **To the user's own devices** - via a `device.avatar` self-message
  (`DeviceSync.md`).

### Forward compatibility

A client that receives a `type` it does not know **must not** crash or drop the
message. It acknowledges receipt and surfaces a placeholder that names the type
and says this version does not support it, keeping the message body so that a
later client version can render it. `v` gates breaking changes to the envelope shape
itself; unknown `type` within a known `v` is a soft, recoverable condition.

Namespacing is dotted (`call.invite`, not `call-invite`) so related signals
group and a client can pattern-match a family (`call.*`) it partially supports.

### Ordering and timestamps

Every content envelope carries `sentAt` as a **unix millisecond** timestamp,
stamped by the sending client from its own clock at send time. It sits inside the
sealed body, under the authorship signature, and it is the single source of truth
for both the message's displayed time and its position in the conversation.

A recipient orders a conversation by `sentAt`, with one bound: a message from the
other side never goes above a message the recipient had already written in that
conversation when it arrived. The order key of such a message is the later of its
`sentAt` and one millisecond after the recipient's newest own message, while the
time it shows stays its `sentAt`. Apart from that bound there is nothing from a
server to order by: a mailbox is told the delivery class and the size of an
envelope and never its contents, so no server is in a position to say when a
message was written - and none is asked. The local arrival clock stands in only
for an envelope that carries no `sentAt` at all.

Two things disturb the order before the reader sees it, and each is repaired
where it happens:

- **In flight.** The duration of a send is set by I2P, and each message travels
  its own way, so two messages written half a second apart can reach the
  recipient's mailbox seconds apart and in either order. The recipient repairs
  this by sorting on `sentAt`.
- **In the mailbox.** A mailbox returns its waiting items in the order it received
  them (`api/ClientApi.md`), so a burst is fetched in the order it arrived rather
  than in whatever order the server's storage happens to enumerate.

Concretely each message gets a stable **order key**: its `sentAt`, raised to the
bound above when it came from the other side. The conversation is kept sorted by
it, so a message written long ago and delivered only now appears where it was
written among the other side's messages, but below anything the recipient wrote
before it arrived. Such a message can therefore appear above the visible part of
the conversation, and noticing it is not left to the reader: the unread mark is
kept as a high-water over *stored* rows, in the order they were received, so a
late arrival is unread whatever its timestamp says.

The bound exists because two clocks are compared. A reply written a second after
a question, by a clock a second and a half behind the asker's, would otherwise
land above the question it answers. The other costs of ordering by a clock the recipient does not
control remain: a correspondent whose clock is wrong (or who lies about it)
places their own messages wrongly among themselves, and nothing but their own
signature backs the time. This is accepted deliberately - the alternative is to
let the party that is meant to know nothing about a message decide where it goes.

## Delivery status

An outgoing message passes through four states. Each is asserted by a different
party, so a state reports exactly how far the message has travelled and on whose
authority.

| State | Definition | Asserted by |
|---|---|---|
| **preparing** | The sending client is making ready the address the message will be sent from. | The sending client (local knowledge). |
| **in flight** | The sending client is dialling the recipient's serving destination and transferring the envelope. | The sending client (local knowledge). |
| **stored** | The envelope is held in the recipient's mailbox. | The recipient's server, in a confirmation signed over the delivery id and returned in the reply on the same stream (`api/Federation.md`). The client dials from an address that accepts no inbound connections, so the reply is the only path available to a confirmation. |
| **read** | The recipient has read the message. | The recipient's client, in a read `receipt` content message. Optional: a client may be configured not to send receipts. |

The presentation of these states is a matter for each implementation; what is
common is the states and their sources.

The first three states are established by the sender without cooperation from the
recipient's client. **stored** is an authenticated statement by the recipient's
*server*, not by the recipient. Only **read** requires the recipient's client to
participate, and it denotes a read rather than an arrival. A read receipt carries
the `id` of the newest message read; the sender applies it to that message and to
every earlier message it sent that peer **which reached the recipient's server**.
A message that never reached it retains its state: a later message being read is
not evidence about an earlier one that was never delivered.

No server carries a send, so the number of attempts and the interval between them
are determined by the sending client. The protocol fixes the following:

- A refusal of the pass or of the size is final; repeating the delivery does not
  change the outcome. A full mailbox or a rate limit is transient: the same
  delivery is accepted later. A failure to reach the recipient's server is not
  final and may be retried.
- A `delivered` response that carries no verifiable signature over the delivery id
  that was sent is not a delivery. The attempt is treated as unanswered.
- Repeating a delivery is idempotent. The delivery id is derived from the message
  and the recipient mailbox, so a server that has already stored the envelope
  recognises the repetition and stores no second copy.
  This is what permits a client to stop retrying and leave the decision to its
  user, including after a restart, when a delivery that was in progress is
  reported as not performed.

## Files: announced, transferred directly

A file is **never** inlined in the mailbox envelope, whatever its size, and it
is not held on a storage service either - there is none. Pictures and voice
messages are the exception: `image` and `audio` carry their bytes inside the
message, within the per-message limit (*Types*, *Size*). For a file the message
**announces** it and the bytes stay with the sender until the recipient asks for
them, at which point the sender serves that one file over a one-time I2P
destination (full design in `api/FileTransfer.md`):

1. The sender hashes the plaintext and sends `type: "file"` carrying
   `{name, size, sha256, mime}` - all end-to-end encrypted, no bytes.
2. When the recipient wants it, its client sends a silent `file.request`; the
   sender answers `file.offer` with a one-time destination and the key it
   encrypted a temporary ciphertext with (CMS password-based envelope, AES-256 -
   standard primitives only, no custom construction).
3. The recipient connects to that destination, streams the ciphertext with
   `offset`-based resume, verifies the announced digest and decrypts. Each
   attempt dials from a **fresh** destination, so a resumed transfer is
   unlinkable to the one it continues.
4. `file.unavailable` is returned when the sender no longer holds the
   file.

The path is chosen by the content type, not by size: a picture or a voice
message is sent inline and a file is announced. Nothing switches one to the
other: a message over the limit is refused by the client before sending
(*Size*), and the server's `MESSAGE_TOO_LARGE` is a final refusal.

Why announce rather than store: the messaging server never carries a file's
bytes, so a **quota-constrained recipient can receive arbitrarily large
content**, and no third party holds the ciphertext at any point. The constraint
that follows: a file is available only while the sender is online and still holds
it, which `TransferState` reports rather than implying delivery.

## Calls

Call signaling (`api/Calls.md`) is exactly Layer 2 content: `call.invite`,
`call.accept`, `call.decline`, `call.end` are `content`-class deliveries the
server relays as ordinary opaque envelopes. Media itself flows peer-to-peer
over a one-time hidden-network tunnel and never touches a server. A call is not
a server feature; it is a family of content types.
