# Bots & Interactive Messages

Status: design draft (decided direction).

## Principle: a bot is an ordinary identity

A bot has no special status anywhere in the infrastructure. It is a normal
identity (`Identity.md`) that registers on a server, accepts contacts and
exchanges the same sealed `content` / `contact` envelopes as a human client
(`Messages.md`, `Contacts.md`). Everything that makes it a *bot* - commands,
inline keyboards, button callbacks - lives entirely in **Layer 2**, the
end-to-end content the server never sees.

This is a direct consequence of the two-layer model:

> The server, facades, federation relays and storage are **unchanged** and
> never learn that a conversation involved a bot. Adding bots touched only the
> client protocol and the content-type registry.

There is deliberately **no** server-side bot registry, bot flag, or bot
discovery in this iteration. A bot is reached exactly like any contact: by its
invite link or by an alias. Discovery, rate-limiting policy and any distinct UI
treatment are open questions left to clients and future iterations - none of them
require a server change to introduce.

A bot runs as a program of its own against `bazarish-bot`, which holds the
account and exposes it through a REST API. A group is a bot that relays its
members' posts to each other; it is specified in `Groups.md`.

## The four content types

All four are `content`-class deliveries (pass-gated, established-contact),
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

If both are present, `data` takes precedence. A keyboard is carried by a message;
it is not a content type of its own and never appears in Layer 1.

### `bot.callback` - a button press

```
{ "v": 1, "type": "bot.callback", "id": <id>, "from": <fp>, "sentAt": <unix ms>,
  "data": "ping",     // the tapped button's payload
  "ref": <keyboard message id>  // which message's keyboard was tapped
}
```

The bot receives the press, acts on `data`, and answers with a new message or
with an `edit` of the keyboard message itself (*Editing in place*). `ref` names
the message whose keyboard was tapped, so the bot knows what the press refers to
and which message an edit replaces.

A button press is not itself shown as a message in the tapping client
(inline-keyboard semantics): the bot's reply or edit is the visible result.
Receipts (`Messages.md`) are not sent for `bot.command` / `bot.callback`; they
are control content, not user-visible lines.

### Editing in place

Rather than reply with a new message, a bot can edit the keyboard message the
button belongs to (an `edit`, `Messages.md`), keyed by the callback's `ref` -
the behaviour in which a message changes in response to a button press. The same
`edit` primitive lets ordinary users revise their own messages, so it is not a
bot-only feature. A bot edits only messages it sent; a client enforces that on
receipt.

### `bot.menu` - the commands a bot offers

```
{ "v": 1, "type": "bot.menu", "id": <id>, "from": <fp>, "sentAt": <unix ms>,
  "menuId": 3303132829,   // chosen at random by the bot
  "items": [ { "title": "Balance", "command": "balance" },
             { "title": "Help", "command": "help" } ]
}
```

A bot tells a contact which commands it takes. A client keeps the newest menu
each contact sent and offers its items beside that conversation; pressing one
sends `/command` to that contact as an ordinary text message. A menu replaces
the one before it as a whole, and an empty `items` takes the menu away. Menus are
ordered by `sentAt`: one sent before the menu a client holds is dropped, because
I2P does not keep order.

The sender checks the limits before sending and the receiver checks them on
arrival; a menu outside them is dropped whole.

| Field | Limit |
|---|---|
| `menuId` | an integer from 0 to 2^32 - 1 |
| `items` | at most 32 |
| `title` | 1 to 32 characters of UTF-8, none of them C0, DEL, C1, a line or paragraph separator or a bidi control (the set `SignInWithKey.md` forbids a consumer) |
| `command` | 1 to 32 of `a-z`, `0-9` and `_`: no slash, no arguments |

A title is shown as plain text: no markup and no links.

**The menu a client holds travels back.** Every `text`, `image`, `audio`,
`file`, `bot.command` and `bot.callback` a client sends to a contact whose menu
it holds carries that menu's id; a client that holds none sends no `menuId`. A
message whose `menuId` is not such an integer is dropped.

```
{ "v": 1, "type": "text", ..., "text": "hello", "menuId": 3303132829 }
```

When a message names no menu or another one than the bot's current menu, the bot
sends `bot.menu` again at once, as a message of its own, without waiting for its
answer. That covers a device added to the account later, a menu lost or
overtaken on the way, and a menu the bot changed: the next message the person
writes brings the current one.

A `bot.menu` is control content: it is not shown as a message and raises no
notification.

### Read receipts

A bot sends a `receipt` for the user-visible content it receives (text, files
and media), so the sender's message advances to the *read* ("green") state - a
bot is a normal recipient in this respect. Receipts are not sent for the control
types (`bot.command`, `bot.callback`, `bot.menu`, `edit`).

## Forward compatibility

`bot.command`, `bot.callback` and `bot.menu` are ordinary registry rows: a client
that does not know them keeps, acks and surfaces them as "unsupported" rather
than dropping them (`Messages.md`). The `keyboard` field degrades to a plain
message, and a `menuId` is ignored by anyone that does not know it.
Nothing here is a breaking change to the envelope, so `v` stays at 1.
