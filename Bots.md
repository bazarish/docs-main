# Bots & Interactive Messages

Status: design draft (decided direction).

## Principle: a bot is an ordinary identity

A bot has no special status anywhere in the infrastructure. It is a normal
identity (`Identity.md`) that subscribes to a server, accepts contacts and
exchanges the same sealed `content` / `contact` envelopes as a human client
(`Messages.md`, `Contacts.md`). Everything that makes it a *bot* - commands,
inline keyboards, button callbacks - lives entirely in **Layer 2**, the
end-to-end content the server never sees.

This is a direct consequence of the two-layer model:

> The server, facades, federation relays and storage are **unchanged** and
> never learn that a conversation involved a bot. Adding bots touched only the
> client protocol and the content-type registry.

There is deliberately **no** server-side bot registry, bot flag, or bot
discovery in this iteration. A bot is reached exactly like any contact: by
fingerprint, by invite link, or by a username (alias) it registered with a
service node. Discovery, rate-limiting policy and any distinct UI treatment are
open questions left to clients and future iterations - none of them require a
server change to introduce.

## The three content types

All three are `content`-class deliveries (token-gated, established-contact),
sealed end-to-end like any message.

### `bot.command` - a command invocation

```
{ "v": 1, "type": "bot.command", "id": <id>, "from": <fp>, "sentAt": <unix ms>,
  "command": "help",        // command name, without a leading slash
  "args": "extra arguments" // raw argument string (may be empty)
}
```

A `bot.command` is the structured form of a `/name args` line. A client SHOULD
dispatch a plain text line beginning with `/` and a `bot.command` through the
same handler, so a bot author writes one handler per command regardless of how
the user expressed it.

### Inline keyboards - interactive messages

Any content message (typically `text`) MAY carry an optional `keyboard`: rows of
buttons rendered under the message. The field is purely additive - a client that
does not understand keyboards still renders the message body.

```
{ "v": 1, "type": "text", "id": <id>, "from": <fp>, "sentAt": <unix ms>,
  "text": "Pick one:",
  "keyboard": [
    [ { "text": "Ping", "data": "ping" }, { "text": "Time", "data": "time" } ],
    [ { "text": "Help", "command": "help" } ]
  ]
}
```

`keyboard` is an array of rows; each row is an array of buttons. A button has a
label (`text`) and exactly one action field:

| Action field | Tap behaviour |
|---|---|
| `data` | The client sends a `bot.callback` to the message's sender, carrying this `data` and `ref` = the keyboard message's `id`. |
| `command` | The client sends a `bot.command` to the sender with this command name. |

If both are present, `data` wins. A keyboard rides on a message; it is not a
content type of its own and never appears in Layer 1.

### `bot.callback` - a button press

```
{ "v": 1, "type": "bot.callback", "id": <id>, "from": <fp>, "sentAt": <unix ms>,
  "data": "ping",     // the tapped button's payload
  "ref": <keyboard message id>  // which message's keyboard was tapped
}
```

The bot receives the press, acts on `data`, and may reply - a new message, a
fresh keyboard, anything. `ref` lets the bot correlate the press to the message
it sent (this protocol has no message editing, so a bot replies rather than
mutating the original).

A button press is not itself shown as a message in the tapping client
(inline-keyboard semantics): the bot's reply is the visible result. Receipts
(`Messages.md`) are not sent for `bot.command` / `bot.callback`; they are
control content, not user-visible lines.

### Editing in place

Rather than reply with a new message, a bot can edit the keyboard message the
button belongs to (an `edit`, `Messages.md`), keyed by the callback's `ref` -
the familiar "the message changes when you tap a button" behaviour. The same
`edit` primitive lets ordinary users revise their own messages, so it is not a
bot-only feature. A bot edits only messages it sent; a client enforces that on
receipt.

### Read receipts

The reference runtime sends a delivery receipt for the user-visible content it
receives (text and media), so the sender's tick advances to the delivered
("green") state - a bot is a normal recipient in this respect. Receipts are not
sent for the control types (`bot.command`, `bot.callback`, `edit`).

## Forward compatibility

`bot.command` and `bot.callback` are ordinary registry rows: a client that does
not know them keeps, acks and surfaces them as "unsupported" rather than
dropping them (`Messages.md`). The `keyboard` field degrades to a plain message.
Nothing here is a breaking change to the envelope, so `v` stays at 1.

## Reference runtime

The client repository ships a minimal bot framework over the same `Session`
logic the CLI and GUI use, plus a demo bot (`bazarish-bot`):

- handlers register per command, for plain text, and for callbacks;
- a new contact is auto-greeted (it falls back to the `start` command);
- the bot polls its mailbox, decrypts with `sync`, and dispatches each update;
- replies are sent with the normal `content` path, spending delivery tokens
  exactly as a human client does.

Because the bot is just a subscribed identity, the same framework runs against a
single server or across a federation with no change.
