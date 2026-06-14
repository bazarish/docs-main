# Messages & Content Types

Status: design draft (decided direction).

## Principle: two independent layers

A message has **two** type discriminators that must never be conflated:

1. **Delivery class** — visible to the destination server, inside the sealed
   delivery envelope. A **closed** set of exactly two values. It selects the
   server's *admission policy*, nothing else.
2. **Content type** — visible only to the recipient, inside the end-to-end
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

## Layer 1 — delivery class (server-visible)

Carried as the `class` field of the sealed delivery envelope (see
`api/Federation.md`). Exactly two values, and this set is frozen — content
types never appear here:

| `class` | Token | Server policy |
|---|---|---|
| `contact` | none | Tokenless bootstrap. Size-capped (`--contact-request-limit`), per-origin rate-limited, separate small quota. The only tokenless path into a mailbox. |
| `content` | one-time delivery token (required) | Token consumed atomically with storage (see `Contacts.md`). The path for every established-contact message, whatever its content type. |

The server branches on `class` and nothing else: it validates the mailbox,
deduplicates by `messageId`, enforces the token (for `content`) or the size
cap (for `contact`), and stores the opaque payload. The pending list returns
the `class` back to the recipient — an admission-class hint the server already
knew, never the content type.

Sealed envelope as the server sees it:

```
{ "class": "content" | "contact",
  "mailbox": <recipient fingerprint>,
  "messageId": <dedup id>,
  "token": <base64 one-time token>      // present iff class == "content"
}
```

## Layer 2 — content type (end-to-end only)

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
| `text` | content | `text` — a plain text line. |
| `contact.request` | contact | Bootstrap: `fromSealing`, `fromServer`, `fromServerCard`, `replyTokens`, `text`. The first message; see `Contacts.md`. |
| `token-refill` | content | `replyTokens` — a fresh stash when the peer's drops low (`Contacts.md`). |
| `file` | content | A content-store reference: `ref`, `key`, `size`, `mime`, `name`. See *Media* below. |
| `photo` | content | Same shape as `file`, plus optional `width`, `height`. |
| `audio` / `voice` | content | Same shape as `file`, plus optional `duration`. |
| `call.invite` | content | Contact type (audio/video), one-time tunnel destination + credentials (`api/Calls.md`). |
| `call.accept` / `call.decline` / `call.end` | content | Call control (`api/Calls.md`). |
| `receipt` | content | `ref` (the acknowledged message's `id`). The recipient's client sends it to confirm receipt — the "green" delivery state below. Optional/toggleable. |
| `bot.command` | content | A command invocation aimed at a bot: `command` (name, no leading slash), `args` (raw argument string). See *Bots & interactive messages*. |
| `bot.callback` | content | An inline-keyboard button press: `data` (the button's payload), `ref` (the `id` of the keyboard message). See *Bots & interactive messages*. |

The registry is **open**: new rows are added here and implemented in clients.
No server, facade, federation or storage change accompanies a new row.

### Bots & interactive messages

A bot is not a server concept either: it is an **ordinary identity** that
subscribes to a server like any client, accepts contacts and exchanges the same
`content`/`contact` envelopes. "Botness" lives entirely in Layer 2 — three
client-only pieces, all forward-compatible:

**Inline keyboard.** Any content message (typically `text`) may carry an
optional `keyboard` field: rows of buttons rendered under the message. A client
that does not understand it still renders the message body, so the field is
additive.

```
"keyboard": [
  [ { "text": "🏓 Ping", "data": "ping" }, { "text": "🕑 Time", "data": "time" } ],
  [ { "text": "❔ Help", "command": "help" } ]
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

### Forward compatibility

A client that receives a `type` it does not know **must not** crash or drop the
message. It acknowledges receipt and surfaces a neutral placeholder
("unsupported message — update your app"), preserving the envelope so a later
client version can render it. `v` gates breaking changes to the envelope shape
itself; unknown `type` within a known `v` is a soft, recoverable condition.

Namespacing is dotted (`call.invite`, not `call-invite`) so related signals
group and a client can pattern-match a family (`call.*`) it partially supports.

## Delivery status

An outgoing message advances through three observable states, each from a
distinct source — so the tick a user sees reflects exactly how far the message
actually travelled:

| Tick | State | Source |
|---|---|---|
| **grey** ✓ | Accepted by the **sender's own server** (it took the sealed envelope into its outbound buffer). | The sender's client, when its `submit` is accepted (local knowledge). |
| **yellow** ✓✓ | Accepted by the **recipient's server** (handed off from the sender's server buffer into the recipient's mailbox). | The sender's server, reported as the send-attempt result. |
| **green** ✓✓ | Received by the **recipient's client**, which explicitly returned a `receipt`. | The recipient's client (a `receipt` content message). Toggleable — a client may decline to send receipts. |

The first two are transport facts the sender learns without any cooperation
from the recipient; only the green state requires the recipient's client to
opt in. The receipt carries the original message's `id`, so the sender matches
it back to the sent message. A failed delivery surfaces as a distinct failed
state, not a tick.

## Media: content-store references

Bulk content (files, photos, audio, voice) is **never** inlined in the mailbox
envelope. It rides through a generic, content-addressed **content store**
(`api/ClientApi.md`):

1. The sender encrypts the bytes with a fresh random content key
   (CMS password-based envelope, AES-256; the key is the password — standard
   primitives only, no custom construction).
2. The sender uploads the ciphertext to the content store, which returns a
   content id = `sha256(ciphertext)`. The store is **type-blind**: it holds
   opaque bytes addressed by hash, with its own quota and TTL, and dedups
   identical ciphertext.
3. The sender's message (`type: "file"` etc.) carries `{ ref: <content id>,
   key: <base64 content key>, size, mime, name }` — all end-to-end encrypted.
4. The recipient fetches `ref` from the content store and decrypts it with
   `key`. The store never sees the key and cannot read the content.

Why a reference and not an inline blob: it keeps the messaging path light,
lets a large transfer resume independently of the conversation, dedups shared
media, and supports group fan-out (one upload, many references) without
re-encrypting per recipient. Crucially the content store is **one generic
mechanism**: it is added once and every present and future media type reuses it
with zero further server work — which is the whole point of the two-layer model.

The content store sees: ciphertext bytes, their size, an opaque content id.
It never sees: who the content is for, what type it is, or its plaintext.

## Calls

Call signaling (`api/Calls.md`) is exactly Layer 2 content: `call.invite`,
`call.accept`, `call.decline`, `call.end` are `content`-class deliveries the
server relays as ordinary opaque envelopes. Media itself flows peer-to-peer
over a one-time hidden-network tunnel and never touches a server. A call is not
a server feature; it is a family of content types.
