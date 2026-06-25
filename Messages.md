# Messages & Content Types

Status: design draft (decided direction).

## Principle: two independent layers

A message has **two** type discriminators that must never be conflated:

1. **Delivery class** - visible to the destination server, inside the sealed
   delivery envelope. A **closed** set of exactly two values. It selects the
   server's *admission policy*, nothing else.
2. **Content type** - visible only to the recipient, inside the end-to-end
   encrypted payload. An **open** registry. The server, facades and federation
   relays never see it.

The invariant this buys:

> Adding a new kind of content (a file, a photo, a voice note, a call signal,
> a read receipt) touches **only the client protocol and the content-type
> registry below**. The server, the storage backend and the federation relay
> require **no change** and never learn what kind of content they carried.

A photo is not a server concept. It is a `content` delivery whose encrypted
payload happens to have `type: "photo"`. The server stores and relays the
identical opaque envelope it would for a text line or a call invite.

## Layer 1 - delivery class (server-visible)

Carried as the `class` field of the sealed delivery envelope (see
`api/Federation.md`). Exactly two values, and this set is frozen - content
types never appear here:

| `class` | Token | Server policy |
|---|---|---|
| `contact` | none | Tokenless bootstrap. Size-capped (`--contact-request-limit`), per-origin rate-limited, separate small quota. The only tokenless path into a mailbox. |
| `content` | one-time delivery token (required) | Token consumed atomically with storage (see `Contacts.md`). The path for every established-contact message, whatever its content type. |

The server branches on `class` and nothing else: it validates the mailbox,
deduplicates by `messageId`, enforces the token (for `content`) or the size
cap (for `contact`), and stores the opaque payload. The pending list returns
the `class` back to the recipient - an admission-class hint the server already
knew, never the content type.

Sealed envelope as the server sees it:

```
{ "class": "content" | "contact",
  "mailbox": <recipient fingerprint>,
  "messageId": <dedup id>,
  "token": <base64 one-time token>      // present iff class == "content"
}
```

## Layer 2 - content type (end-to-end only)

The encrypted payload (sealed to the recipient's user sealing key) is a
versioned envelope:

```
{ "v": 1,
  "type": <content-type string>,
  "id":   <client message id>,
  "from": <sender fingerprint>,
  "sentAt": <unix seconds>,
  "replyTo": <id>,            // optional
  ... type-specific fields ...
}
```

### Content-type registry (v1)

| `type` | Delivery class | Purpose / payload |
|---|---|---|
| `text` | content | `text` - a plain text line. |
| `contact.request` | contact | Carries a `bootstrap` block (`sealing` prekey, `dest`, `servingKey`, `replyTokens`) plus `text`. The first message; see `Contacts.md`. |
| `token-refill` | content | `replyTokens` - a fresh stash when the peer's drops low (`Contacts.md`). |
| `file` | content | A sealed blob-storage pointer `ptr` (base64) - the encrypted blob lives in blob storage, fetched over I2P. See *Media* below and `api/LargeFiles.md`. |
| `photo` | content | Same shape as `file`, plus optional `width`, `height`. |
| `audio` / `voice` | content | Same shape as `file`, plus optional `duration`. |
| `call.invite` | content | Contact type (audio/video), one-time tunnel destination + credentials (`api/Calls.md`). |
| `call.accept` / `call.decline` / `call.end` | content | Call control (`api/Calls.md`). |
| `receipt` | content | `ref` (the read message's `id`). A **read** receipt - the recipient's client sends it only on a genuine read (the chat is open, the app window focused, and the message scrolled into the visible area), confirming the "green" delivery state below. `ref` is the newest read incoming id, so it acknowledges that message and every earlier one (a read high-water). Optional/toggleable. |
| `bot.command` | content | A command invocation aimed at a bot: `command` (name, no leading slash), `args` (raw argument string). See *Bots & interactive messages*. |
| `bot.callback` | content | An inline-keyboard button press: `data` (the button's payload), `ref` (the `id` of the keyboard message). See *Bots & interactive messages*. |
| `edit` | content | An in-place edit of a message the sender previously sent: `ref` (the target message's `id`), `text` (new text), `keyboard` (new inline keyboard, `[]` to clear). The recipient replaces its copy. Applied only to a message `from` actually sent. See *Editing messages*. |
| `group.invite` | content | Adds a recipient to a group: `groupId`, `name`, `roster` (b64 signed roster). See `Groups.md`. |
| `group.tokens` | contact | A member's one-time token pool for a group: `groupId`, `tokens`. Tokenless distribution. See `Groups.md`. |
| `group.roster` | contact | A signed roster update (membership/admin/epoch). Sent over the tokenless contact class so a critical update never fails on a drained pool. See `Groups.md`. |
| `group.leave` | contact | The sender is leaving a group: `groupId`. See `Groups.md`. |
| `call.invite` | content | Places an audio call: `callId`, `media` (`audio`), `codec` (`opus`), `dest` (the caller's media b32), `key` (base64 per-call media key). See `api/Calls.md`. |
| `call.accept` | content | Accepts a call: `callId`, `dest` (the callee's media b32). See `api/Calls.md`. |
| `call.decline` | content | Rejects a call: `callId`, `reason`. See `api/Calls.md`. |
| `call.end` | content | Hangs up / cancels a call: `callId`. See `api/Calls.md`. |

A content message of any group type **and** any ordinary content type (`text`,
`file`, ...) may additionally carry a `"group": { "id": <groupId> }` field - it
files the message under that group conversation instead of the one-to-one thread.
The group is invisible to the server: group messages are ordinary `content`
deliveries fanned out to each member client-side (`Groups.md`).

The registry is **open**: new rows are added here and implemented in clients.
No server, facade, federation or storage change accompanies a new row.

### Bots & interactive messages

A bot is not a server concept either: it is an **ordinary identity** that
subscribes to a server like any client, accepts contacts and exchanges the same
`content`/`contact` envelopes. "Botness" lives entirely in Layer 2 - three
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

Because all three are `content`-class deliveries, the server, facades,
federation and storage are unchanged and never learn that a conversation
involved a bot. The full model and the reference bot runtime are in `Bots.md`.

### Editing messages

A sender can revise a message it already sent with an `edit` (above): the
recipient finds the message whose `id` equals `ref` and replaces its `text` and
`keyboard` in place, marking it edited. It is the same primitive for two cases:

- **Bots** edit their own keyboard message on a callback - the Telegram-style
  "the message updates when you tap a button" - by sending an `edit` whose `ref`
  is the callback's `ref`.
- **Users** edit their own line: the client updates its local copy and sends an
  `edit` so the peer updates theirs.

A client applies an `edit` **only** to a message the `from` party actually sent
(for a recipient, an incoming message from that peer); it must not let a peer
rewrite messages it did not author. There is no separate "delete"; an edit to
empty text is the closest equivalent. History of prior revisions is not
retained by the protocol (a client may keep its own).

### Forward compatibility

A client that receives a `type` it does not know **must not** crash or drop the
message. It acknowledges receipt and surfaces a neutral placeholder
("unsupported message - update your app"), preserving the envelope so a later
client version can render it. `v` gates breaking changes to the envelope shape
itself; unknown `type` within a known `v` is a soft, recoverable condition.

Namespacing is dotted (`call.invite`, not `call-invite`) so related signals
group and a client can pattern-match a family (`call.*`) it partially supports.

## Delivery status

An outgoing message advances through three observable states, each from a
distinct source - so the tick a user sees reflects exactly how far the message
actually travelled:

| Tick | State | Source |
|---|---|---|
| **grey** (single) | Accepted by the **sender's own server** (it took the sealed envelope into its outbound buffer). | The sender's client, when its `submit` is accepted (local knowledge). Instant on the LAN facade - the submit never blocks on federation. |
| **yellow** (double) | Stored by the **recipient's server** (handed into the recipient's mailbox). | The recipient's server, via a **signed `delivered-ack`** it federates back to the sender's server (`api/Federation.md`); the sender's client learns it by polling the send attempt. |
| **green** (double) | **Read** by the recipient - the chat is open, focused and the message scrolled into view. | The recipient's client (a read `receipt` content message). Toggleable - a client may decline to send receipts. |

Grey and yellow are transport facts the sender learns without the recipient's
client cooperating (yellow is the recipient *server's* authenticated confirmation,
not the recipient *user*); only green requires the recipient's client to opt in
and reflects an actual read, not mere arrival. The read receipt carries the newest
read message's `id`, so the sender greens that message and every earlier one it
sent that peer. A failed delivery surfaces as a distinct failed state, not a tick;
a delivery the sender's server cannot confirm stays grey (never a false failure).

## Media: blob-storage pointers

Bulk content (files, photos, audio, voice) - and any message blob over **512 KB**
- is **never** inlined in the mailbox envelope. It rides through the **blob store**
(rotating encrypted-LeaseSet destinations served only over I2P). The whole path is
**streamed end-to-end** - sender, facade, store and recipient each spool to disk, so
no node holds the blob in memory and a file can exceed any one node's RAM (full
design in `api/LargeFiles.md`):

1. The sender encrypts the bytes with a fresh random key (CMS password-based
   envelope, AES-256; the key is the password - standard primitives only, no
   custom construction).
2. The sender uploads the ciphertext to the blob store, which returns a
   `.b32.i2p` capability URL, a `blobId` and a `deleteToken`. The store holds
   opaque ciphertext only, with its own quota and TTL.
3. The sender's message (`type: "file"` etc.) carries `{ ptr: base64(sealed
   BlobPointer), size, mime, name }` - all end-to-end encrypted. The pointer
   holds the download URL, the decryption key and the integrity digest.
4. The recipient fetches the ciphertext over I2P (a transient I2P destination
   straight to the blob's b33, or its own server's proxy fetch), verifies the
   digest and decrypts. The store never sees the key and cannot read the content.

Why a pointer and not an inline blob: the messaging server never carries the
bulk bytes, so a **quota-constrained recipient can receive arbitrarily large
content**; the transfer is independent of the conversation; and download is
I2P-only (`.b32.i2p`), so a direct fetch can never leak the fetcher's IP.

The blob store sees: ciphertext bytes and their size. It never sees: who the
content is for, what type it is, the decryption key, or its plaintext.

## Calls

Call signaling (`api/Calls.md`) is exactly Layer 2 content: `call.invite`,
`call.accept`, `call.decline`, `call.end` are `content`-class deliveries the
server relays as ordinary opaque envelopes. Media itself flows peer-to-peer
over a one-time hidden-network tunnel and never touches a server. A call is not
a server feature; it is a family of content types.
