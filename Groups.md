# Groups & Channels

Host: `bazarish-bot`; member: `client-qt`.

## Principle: a group is a bot that relays

A group is an ordinary identity (`Identity.md`) run as a **group host**: a
`bazarish-bot` account of kind `group`. A member adds it like any contact, by
invite or alias, and the host relays each member's post to every other member,
from itself, under the author's name in the chat. Everything that makes the
conversation a group lives in **Layer 2** (`Messages.md`): to the servers,
facades, federation and storage a group is one contact that sends many
`content` envelopes.

A **channel** is the same host with posting closed to members: only its admins
write, everyone else reads.

## What a member sees, and what they do not

- The number of members, as a snapshot - never the list.
- Other members only by their name in the chat - never a fingerprint.
- Every post signed by the host. **Authorship inside a group is not
  verifiable**: the host could put any text under any name. That is the price
  of a hidden member list, and trust in a group is trust in its host.
- The host knows everything: fingerprint and name of every member, every post,
  who wrote it and when.
- An admin additionally sees the descriptor of a member who asks to join, and
  a member's descriptor when the member asks for contact (see *Contact
  requests*); an ordinary member never does.

The host delivers to all its members from one outbound address, the one
exception to one address per correspondent (`api/Federation.md`). A server
holding several members therefore sees one sender in their mailboxes and
learns that they share a group, though not which.

## The `group` block

Every `content` envelope a group host sends carries one additive block:

```
"group": { "kind":    "group" | "channel",
           "members": <int>,          // member count, a snapshot
           "write":   <bool>,         // this recipient may post
           "admin":   <bool>,         // this recipient is an admin
           "at":      <unix ms> }     // when the host computed this block
```

- It rides on **every** envelope: posts, edits, deletes, reaction counts, the
  host's own prompts and `contact.accept`. So the first message a member ever
  receives from the host already tells their client it is a group.
- All five fields are required. A block missing one is rejected with the
  envelope, as any malformed envelope is.
- `members` is a snapshot the host refreshes **once an hour**, not on every
  join or leave: a live count would tell every member the moment somebody
  joined.
- `write` and `admin` are computed for the recipient. `write` is true while the
  member is choosing a name, true for one who left or was excluded (writing is
  how they join again), true for every member of a `group` and for an admin of
  a `channel`, and false while a request to join awaits an admin.
- The client keeps the newest block per contact (by `at`). A message from that
  contact with no block and a `sentAt` newer than the kept `at` drops the group
  state: the account is no longer a group host.
- The group's title is **not** in the block. A contact's name is the
  recipient's own label, taken from the descriptor at add time
  (`Messages.md`, *Avatars and display names*).

## Relayed posts

A post is an ordinary `text`, `image` or `audio` message from the host with two
additive fields:

```
"author":      <name in the chat>,
"contactable": <bool>      // the author accepts contact requests
```

- **The id is the author's own.** The host relays a post under the `id` its
  author gave it, to everyone. An edit, a delete, a reaction or a `replyTo`
  therefore names the same message on every screen, and the author's other
  devices already hold that id from their own echo (`DeviceSync.md`). The host
  refuses a post whose id it has already relayed.
- `replyTo` is passed on unchanged.
- `sentAt` is the moment the host accepted the post. It is the same in every
  copy and in the history a newcomer receives, so history sorts into the past.
- The author receives no copy: their own outgoing message is the post. The
  host confirms every post it took with a `receipt` to its author, which is
  what marks the post delivered on the author's screen.
- A client honours `author` only on a message that carries a `group` block.
  On any other message it is ignored, or any contact could put a name over its
  own bubble.
- A post the host writes itself, not relayed from a member, carries
  `"fromHost": true` in place of `author` and `contactable`. It goes to every
  member, admins included. A client honours it only on a message with a `group`
  block and draws it with no author line: the group itself is speaking.
- A message from the host with neither `author` nor `fromHost` is the host
  speaking to this member alone - a prompt, a notice, an admin's request card.
  Clients mark it as visible to the recipient only.

A post that would exceed the size limit (`Messages.md`, *Size*) once `author`
and the block are added is refused when it arrives, with a notice to its
author. Files are not relayed: a file message from a member is dropped without a
notice. A picture or a voice note that fits one envelope is relayed.

## Edits and deletes

- The author edits or deletes their post as any message: `edit` / `delete` to
  the host. The host checks that the post is theirs and relays the same
  `edit` / `delete` to everyone else. Only a text post is edited; an edit to a
  picture or a voice note is ignored.
- An admin deletes another member's post with `bot.command` `delete <id>`. The
  host relays the `delete` to everyone except the author. The author's own copy
  stays: it is their outgoing message, and a `delete` applies only to messages
  sent by its `from`.
- Nobody edits another member's post.

## Reactions

- A member reacts as in any chat: a `reaction` to the host. The host keeps one
  reaction per member per post.
- Members see **counts only**, never who reacted. The host sends them as they
  change; changes that come while the previous delivery to a member is on its
  way reach that member together:

```
{ "type": "group.reactions",
  "reactions": [ { "ref": <post id>, "counts": { <emoji>: <n>, ... } }, ... ] }
```

- The counts **leave out the recipient's own reaction**; the client adds the
  one it holds. An empty `counts` clears a post's reactions.

## Joining

1. The member adds the host as a contact; the host accepts.
2. The host asks for a **name in the chat**. The member answers with a plain
   text message. Only a post is an answer: a receipt or any other service
   message from the member gets no reply.
3. The host checks the name (*Names*). A taken or refused name is asked for
   again.
4. In an **open** group the member is in. In an **approval** group every admin
   receives the applicant's name and descriptor with two inline buttons; the
   first admin to decide settles it, and the host edits the card for every
   admin to say who decided. Requests no admin has decided are also listed in
   the host's own interface.
5. A member who is in receives the last N posts (N chosen by the owner, 0
   allowed) and from then on every new one.

A contact request repeated under the same id by someone already on the way in
or in - naming, waiting for an admin, or a member - is answered with the accept
alone: the host does not ask for the name again, admit again or send the
history again. A request under a new id is a contact added anew, and joining
starts over.

## Names

- 1 to 32 code points, without control or format characters, and without a
  `bazarish://` link.
- Two names that **look alike** are the same name: names are compared by their
  confusable skeleton (Unicode UTS #39) after case folding and NFKC, so `a` and
  Cyrillic `a` clash, and a name that mixes scripts is refused.
- A name stays with its member: a member who leaves keeps it, and the same
  fingerprint coming back gets it back.
- A member changes it with `bot.command` `rename`. The next text they send is
  the new name, checked as above and not relayed; a refused one is answered and
  the host waits for another. The old name is freed.

## Roles

There is one role, **admin**, granted in the host's own interface. The owner is
the host itself.

- An admin decides requests to join (*Joining*).
- An admin excludes a member from one of their posts: `bot.command`
  `exclude <id>`. The member is out and may come back by writing again.
- An admin blocks a member the same way: `bot.command` `block <id>`. The
  member is banned by fingerprint and cannot rejoin. An admin cannot exclude or
  block another admin.
- An admin deletes any post (*Edits and deletes*).
- In a channel, admins write.

A member's client learns of its own role from `admin` in the block. The host
sends `group.state` - a message with no body, only the block - when `write` or
`admin` changes and it has nothing else to send.

## Contact requests

- A post marked `contactable` offers "request contact". The member sends
  `bot.command` `contact <id>`, and the host passes the requester's descriptor
  to the author, in a message only the author sees. The author decides; the
  author's own descriptor goes nowhere. A member asks a given author once; a repeated ask is
  not passed on.
- A member turns this on and off with `bot.command` `contactable on` /
  `contactable off`. The owner can switch it off for the whole group.

## Leaving

A member leaves by deleting the host from their contacts. The host learns it
only as `DELIVERY_REJECTED` on its next delivery (`Errors.md`): the first such
refusal makes the member one who left, and the host stops sending. A server does
not say whether a pass is unknown or a mailbox is someone else's, so the refusal
is the only signal. A member who left and writes again joins again
(*Joining*), under the same name.

## Settings

The welcome message carries one inline button, **Settings**, and says that
`/settings` brings it up again, written as an offer to send it (`!!/settings!!`).
The button turns that message's buttons into the `contactable` switch and
**Change name** (`rename`). A member who sends `settings` - as a `bot.command`,
or as the plain text `/settings` - gets a message of their own with the same
button; the host does not relay it.

## Descriptors inside text

Any text message, in any chat, may contain a descriptor link
(`bazarish://invite?...`, `api/FederatedResolve.md`). Clients render it as a
plate with the name it carries and a **Write** action that opens adding a
contact with that descriptor filled in; the rest of the text stays in the same
bubble. This is how a request to join reaches an admin and a contact request
reaches an author. The name in a descriptor is the sender's claim and is not
verified, and a descriptor goes stale when its owner raises a new key.

## Commands

| `bot.command` | From | Meaning |
|---|---|---|
| `settings` | member | a message with the Settings button |
| `rename` | member | take the next text as the new name |
| `contactable` `on` / `off` | member | accept or refuse contact requests |
| `contact <id>` | member | ask the author of post `<id>` for contact |
| `exclude <id>` | admin | exclude the author of post `<id>` |
| `block <id>` | admin | exclude and ban the author of post `<id>` |
| `delete <id>` | admin | delete post `<id>` for everyone |

Decisions on a request to join are `bot.callback` presses on the card's
buttons.

## History and re-joining

The history a newcomer receives is relayed with the posts' own ids. A member
who comes back within the time a server remembers a delivery id gets nothing
new for posts already delivered there; only a member who cleared the chat
notices.
