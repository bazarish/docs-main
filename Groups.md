# Groups & Channels

Host: `bazarish-bot`; member: `client-qt`.

## Principle: a group is a bot that relays

A group is an ordinary identity (`Identity.md`) run as a **group host**: a
`bazarish-bot` account of kind `group`, whose contact card says
`kind: "chat"`. A member adds it like any contact, by
invite or alias, and the host relays each member's post to every other member,
from itself, under the author's name in the chat. Everything that makes the
conversation a group lives in **Layer 2** (`Messages.md`): to the servers,
facades, federation and storage a group is one contact that sends many
`content` envelopes.

A **channel** is the same host with posting closed to members: only its admins
write, everyone else reads.

## What a member sees, and what they do not

- The number of members, as a snapshot. In a `group` also the names of the
  members (*Member list*); in a `channel` never the list.
- Other members only by their name in the chat - never a fingerprint.
- Every post signed by the host. **Authorship inside a group is not
  verifiable**: the host could put any text under any name. That is the price
  of hiding who the members are, and trust in a group is trust in its host.
- The host knows everything: fingerprint and name of every member, every post,
  who wrote it and when.
- An admin additionally sees the descriptor and the request text of a member
  who asks to join, and a member's descriptor when the member asks for contact
  (see *Contact requests*); an ordinary member never does.

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
- `members` is a snapshot the host refreshes **once an hour**, so the count
  does not tell members the moment somebody joined.
- `write` and `admin` are computed for the recipient. `write` is true while the
  member is choosing a name, true for one who left or was excluded (writing is
  how they join again), true for every member of a `group` and for an admin of
  a `channel`, and false while a request to join awaits an admin.
- The client keeps the newest block per contact (by `at`). A message from that
  contact with no block and a `sentAt` newer than the kept `at` drops the group
  state: the account is no longer a group host.
- The group's name on the recipient's screen is the recipient's own label for
  the contact, taken from the descriptor at add time (`Messages.md`, *Avatars
  and display names*).

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
  host answers every post a member sends it with a `receipt` to its author,
  a refused post included, so the author's screen shows the post delivered; a
  refusal is told by a notice of its own.
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
2. The member's **name in the chat** is the display name their contact
   request carries (`dn`, `Contacts.md`). The host checks it (*Names*) and
   says nothing when it is accepted.
3. Only when the request carries no name, or the name is taken or refused,
   does the host ask for one. The member answers with a plain text message,
   checked the same way and asked for again until one is accepted. Only a post
   is an answer: a receipt or any other service message from the member gets
   no reply.
4. In an **open** group the member is in. In an
   **approval** group every admin receives the applicant's name, descriptor and
   the text of their contact request with two inline buttons; the first admin to decide settles it, and the host
   edits the card for every admin to say who decided. Requests no admin has
   decided are also listed in the host's own interface.
5. A member who is in receives the last N posts (N chosen by the owner, 0
   allowed) and from then on every new one.

Everything waiting for a member goes out together: posts, edits, deletes and
reaction counts are packed into one envelope in the order they happened, oldest
first, as many as fit its size limit (`Messages.md`, *Encoding*). A change
therefore never arrives before the post it changes. A post goes out with its
current text, so an edit made before it is not sent again.

A contact request repeated under the same id by someone already on the way in
or in - naming, waiting for an admin, or a member - is answered with the accept
alone: the host does not ask for a name again, admit again or send the
history again. A request under a new id is a contact added anew, and joining
starts over.

## Names

- 1 to 32 code points, without control or format characters, and without a
  Bazarish link in either spelling (`Architecture.md`).
- Two names that **look alike** are the same name: names are compared by their
  confusable skeleton (Unicode UTS #39) after case folding and NFKC, so `a` and
  Cyrillic `a` clash, and a name that mixes scripts is refused.
- A name holds no white space other than the space U+0020.
- Two names whose mention forms (*Mentions*) look alike are the same name, so
  `Anna Maria` and `Anna_Maria` cannot both be in one chat: the comparison
  above is made on the name with every space written as `_`.
- A name stays with its member: a member who leaves keeps it, and the same
  fingerprint coming back gets it back.
- A member changes it with `bot.command` `rename`. The next text they send is
  the new name, checked as above and not relayed; a refused one is answered and
  the host waits for another. The old name is freed.

## Member list

In a `group` every member who is in holds the names of all members. A
`channel` has no list.

```
{ "type":  "group.members",
  "op":    "full" | "added" | "removed" | "none",
  "names": [ <name>, ... ],   // full, added, removed
  "total": <int>,             // full, added, removed: the member count after this letter
  "part":  <int>,             // full only: 1 for the first part
  "parts": <int>,             // full only: how many parts the list has
  "you":   <name> }           // optional: which of `names` is the recipient
```

- It comes from the host only, with the `group` block, and is never shown as a
  bubble.
- `full` is the whole list, split into parts of at most 1000 names, sent part 1
  first and in order. The client takes the list when the last part has come and
  the names it collected number `total`.
- `added` and `removed` are one change each, sent as it happens, in order with
  the posts. `total` is the member count after the change.
- A rename is a `removed` with the old name followed by an `added` with the new
  one. The client keeps names, not people, so to it a rename is exactly the old
  name leaving and the new one coming; a second operation would carry nothing
  more.
- `you` names the recipient: in the part of a full list that holds the
  recipient's own name, and in the `added` that brings the recipient's new name
  after a rename. The client keeps it as its own name in the chat; mentions of
  the recipient are matched against it (*Mentions*).
- The host sends the full list to a member it admits, and to a member who asks
  for it with `group.members.ask` (a message with no body). It answers only a
  member who is in; an ask from someone choosing a name, waiting for an admin,
  or gone gets nothing.
- A client asks when it has a `group` block from the host and no list, once per
  run of the client for that chat; and again whenever a full list arrives
  broken (a part out of order, a count other than `total`) or a change leaves
  it a count other than `total`.
- In a `channel` the host answers the ask with `{"op": "none"}` and sends no
  changes, and the client offers no member list. When the owner changes the
  kind, every member receives the full list or `none`.
- `total` is exact: unlike the hourly `members` of the block, it tells the
  members of a `group` of a join or a leave as it happens.
- A name in `names` or `you` is at most 128 bytes (32 code points, *Names*), and
  one letter carries at most 1000 names. A letter outside these bounds is
  malformed.

## Mentions

A mention is `@` followed by a member's **mention form** in the text of a post:
the name as the member list spells it, with every space (U+0020) written as
`_`. Member `Anna` is mentioned as `@Anna`, member `Anna Maria` as
`@Anna_Maria`. There is no field for it: the host relays the text as it is and
checks nothing. An `@` that names no member is plain text.

The reader's client finds mentions against its member list and its own name:

1. A mention starts at an `@` that is the first character of the text or
   follows white space or one of `(` `[` `{` `"` `'` U+00AB U+201C U+2018.
2. The run is every character after the `@` up to the next white space or the
   end of the text.
3. The candidates are the run, then the run without its last character, again
   and again while that character is one of `,` `.` `!` `?` `:` `;` `)` `]`
   `}` `"` `'` U+00BB U+201D U+2019 U+2026. The first candidate that is a
   member's mention form is the mention; when none is, the `@` mentions nobody.

White space is the Unicode `White_Space` set: U+0009-U+000D, U+0020, U+0085,
U+00A0, U+1680, U+2000-U+200A, U+2028, U+2029, U+202F, U+205F, U+3000.

- A post **mentions the recipient** when the recipient's own name (`you`) is a
  mention in it, or when its `replyTo` names a post of the recipient's own.
- An edit finds the mentions of the new text.
- The client marks such a post, counts the unread ones per chat and leads to
  the first of them. It wakes a notification for such a post even when the
  chat's notifications are off; the client's own notification settings still
  apply.
- The composer offers names from the local list, the recipient's own left out,
  and writes the chosen one in its mention form.

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
  not passed on. A member who asks an author who refuses contact requests is
  told so, and the author learns nothing of it.
- A member turns this on and off with `bot.command` `contactable on` /
  `contactable off`. The owner can switch it off for the whole group.

## Leaving

A member leaves by deleting the host from their contacts. The host learns it
only as `DELIVERY_REJECTED` on its next delivery (`Errors.md`): the first such
refusal makes the member one who left, and the host stops sending. A server does
not say whether a pass is unknown or a mailbox is someone else's, so the refusal
is the only signal.

A member whose server does not answer goes on an offline list, with the time it
first did not; what is relayed to them waits in the host's outbox
(`Messages.md`). The next delivery that arrives takes them off it. A member still
on the list 30 days later is a dead account: they leave, what waits for them is
withdrawn, the host stops sending, and the member count drops.

A member who left and writes again joins again (*Joining*), under the same name.

## Settings

A member who sends the `bot.command` `settings` gets a message of their own that
names their name in the chat and carries two inline buttons: the `contactable`
switch and **Change name** (`rename`). Pressing the switch redraws that message.
The host relays neither the command nor the answer. A client offers `settings` in
the group chat's header as **Account settings**, the way it offers a bot's menu
(`Bots.md`).

## Descriptors inside text

Any text message, in any chat, may contain a descriptor link
(`https://bazari.sh/#invite?...`, `api/FederatedResolve.md`). Clients render it as a
plate with the name it carries and a **Write** action that opens adding a
contact with that descriptor filled in; the rest of the text stays in the same
bubble. This is how a request to join reaches an admin and a contact request
reaches an author. The name in a descriptor is the sender's claim and is not
verified, and a descriptor goes stale when its owner raises a new key.

## Commands

| `bot.command` | From | Meaning |
|---|---|---|
| `settings` | member | a message with the member's name, the `contactable` switch and Change name |
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
