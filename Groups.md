# Group Chats

Status: **v1 design** - the directional decisions below are now realized by the concrete protocol in *v1 design*. Group traffic is pure Layer-2 content (`Messages.md`); no server, facade, federation or storage change.

## Decided direction

- **Client-side fan-out.** A group is a purely client-side construct: the roster exists only on members' devices, E2E. Sending to a group = N-1 ordinary pairwise deliveries through standard sealed envelopes. Servers never learn that groups exist - group traffic is indistinguishable from direct traffic; no new server code, no new metadata.
  - The group-mailbox alternative (group as a pseudo-user, members as its "devices") was considered and rejected: the hosting server sees group size and activity, and member removal is broken under a shared identity key (an excluded member can read forever; only full group re-creation helps).
- **Group token pools.** Each member issues one pool of one-time delivery tokens to the group (registered as usual, distributed via E2E group-control messages) instead of O(N2) pairwise stashes. Concurrent senders may collide on a token - same optimistic retry as multi-device (`DeviceSync.md`). Member removal: everyone revokes the pool tokens they issued to the group and broadcasts fresh pools.
- **Multi-admin governance.** The roster and the admin set are signed by any current admin's identity key; the creator is the first admin and can appoint others. The group survives the creator's account loss (identity keys are unrecoverable, see `DeviceSync.md`).
- **Target size <= ~100 members.** Fan-out over I2P is realistic at this scale (parallel sends, retries). Large broadcast channels are a separate future concept and must not be solved by stretching groups.
- **No batch-send API.** A batch endpoint would turn the statistical timing correlation at the sender's server into an explicit, reliable list of linked destination servers. Clients send individually with their own pacing/jitter policy.

## v1 design

A group is identified by a random **`groupId`** (16 bytes, hex) that exists only
in end-to-end payloads. Every group construct is a Layer-2 content type (added to
the `Messages.md` registry); the server relays the same opaque envelopes it does
for direct messages.

### Signed roster

The roster is the single source of truth for membership, routing and the admin
set. It is a JSON body signed by a **current admin's identity key** (hybrid PQ,
`cms::signJsonHybrid` - the signature is self-contained and yields the signer's
identity fingerprint on verification):

```
{ "v": 1, "groupId": <hex>, "name": <string>, "epoch": <int>,
  "members": [
    { "fp": <fingerprint>, "sealing": <b64 SPKI>, "dest": <i2p destination>,
      "servingKey": <b64 SPKI>, "admin": <bool> }, ... ],
  "admins": [ <fingerprint>, ... ] }
```

Each member entry carries everything needed to deliver to that member (their user
sealing key, their serving destination, and that destination's serving sealing
key), so a member learns how to reach **every** other member from the roster
alone - no per-member contact lookup.

**Admin authority is anchored, never self-asserted.** A recipient verifies the
embedded signature and then decides whether to trust the body as follows:

- For a group it **already holds**, the signer must be an admin the recipient
  **already recognizes** (an admin in the recipient's current roster). The
  incoming roster's own `admins` array is **not** consulted for this decision -
  consulting it would be circular: any member could sign a roster that lists
  themselves as admin and take the group over. This is the critical rule that
  makes "only an admin may change the name/photo/membership" a **cryptographic**
  guarantee, not a UI restriction. A roster signed by a non-admin (even one that
  names itself admin) is rejected.
- Only the **first** roster a member ever sees for a group - delivered as a
  `group.invite` (trust on first use, via the inviting contact) - may establish
  the initial admin set from its own `admins`. A `group.roster` for a group the
  recipient does not hold is rejected (it cannot bootstrap an admin set).

Later rosters must also carry a `epoch` **>=** the one held (a lower epoch is
stale and ignored), so a captured older roster cannot roll membership back.

### Token pools, distributed tokenless

Each member issues **one** pool of one-time delivery tokens for the group
(registered with its own server like any token batch) and hands the raw tokens to
every other member. Distribution does **not** need a relay: it rides the
**tokenless `contact` delivery class** (`Messages.md`), the standing path into any
mailbox. So on joining, a member sends each other member a `group.tokens` over the
`contact` class - no chicken-and-egg, no batch API. To send into the group, a
member spends one token from the **recipient's** pool, per recipient.

### Content types

| `type` | Class | Purpose / payload |
|---|---|---|
| `group.invite` | content | Bootstrap to a new member: `groupId`, `name`, `roster` (b64 signed roster). Sent by an admin over an established contact. |
| `group.tokens` | contact | A member's pool for the group: `groupId`, `tokens` (b64 one-time tokens). Tokenless distribution to each member. |
| `group.roster` | contact | A roster update (membership/admin/epoch change), `roster` (b64 signed). Broadcast to all members over the tokenless contact class, so a critical update is not lost to a drained pool. |
| `group.leave` | contact | The sender is leaving: `groupId`, plus a `gsig` over `{type, groupId, from}`. The recipient removes **only the signer** (the signed `from` must equal the verified signer), so a member can remove themselves but cannot forge another member's removal. |
| `group.avatar` | content | **Admin only.** Sets the group photo: `avatar` = `{ mime, data }` (square PNG/JPEG, <= 500 KB). A signed group message: recipients store it as the group's picture **and** surface a "set the group photo" bubble from the setter. Dropped if the signer is not a current admin. |
| `group.rename` | content | **Admin only.** A rename notice: `text` = the new name. A signed group message that surfaces a "changed the group name" bubble. The **authoritative** name is the one in the roster (`group.roster`, epoch-protected); this is the highlighted notice only. Dropped if the signer is not a current admin. |
| `reaction` | content | A member's reaction to a group message: `ref` (target id), `text` (emoji; empty removes it), plus a `gsig`. Fanned out to every member, so everyone sees who reacted. One per member per message; a new emoji overwrites the old. See `Messages.md`. |
| `receipt` | content | A group **read** receipt: `ref` (the read message's id), plus a `gsig`. Sent to the read message's **author only** (never fanned out), so the author records the verified viewer and the message turns green on the first read. Opt-in (the local read-receipts setting). See `Messages.md`. |
| `group` field | - | Any content message (`text`, `file`, ...) may carry `"group": { "id": <groupId> }`. It rides the existing type; the recipient files it under that group. Not a type of its own. |

A group content message may also carry a `dn` field (the sender's own display
name) and a `replyTo` field (`Messages.md`). `dn` is inside the signed body
(below), so it cannot be spoofed onto another member.

### Lifecycle

- **Create.** The creator (first admin) builds the roster from existing contacts,
  signs it, sends each member a `group.invite`, and broadcasts its own
  `group.tokens` to every member.
- **Join bootstrap.** On `group.invite`, a member verifies the roster, stores the
  group and every member's routing, issues its own pool, and sends a
  `group.tokens` to every other member. After all members have done so the mesh of
  pools is complete and anyone can send to anyone.
- **Send.** For each member != self, seal the inner payload (with the `group`
  field) to that member and deliver `content`-class with one of *their* pool
  tokens - N-1 individual sends, the client's own pacing/jitter (no batch API).
- **Add (admin action).** Adding members is an admin privilege: an admin signs a
  new roster (with the new members, `epoch+1`), sends each **new** member a
  `group.invite` (their join bootstrap), broadcasts the updated `group.roster` to
  the **existing** members, and broadcasts its pool to the new members. Each side
  shows a service notice so the change is never silent: the admin sees an
  optimistic "You added X" line, a new member sees "You were added to <group>"
  (from the invite), and an existing member sees "X joined the group" - derived by
  the recipient from diffing the applied roster against the one it held (the
  members present now but not before), since the wire roster carries no per-member
  display name. The notice names a joiner by the recipient's local contact name if
  any, else a short fingerprint (the joiner's own name fills in once they first
  send, per *Sender authenticity*). The diff is idempotent: a redelivered or
  equal-epoch roster re-applies to the same membership and adds nobody, so no
  duplicate notice. A roster that only removed members or changed admins adds
  nobody and stays quiet.
- **Remove (admin action).** Removing *another* member is an admin privilege: an
  admin signs a new roster (without the member, `epoch+1`) and broadcasts a
  `group.roster`. Recipients apply it only because the signer is an admin they
  already recognize (see *Signed roster*); a non-admin's removal attempt is
  rejected. Every member then revokes the pool tokens it issued (deletes the
  hashes at its server) and broadcasts a **fresh** `group.tokens` to the remaining
  members, so the removed member's stale tokens stop working at the servers and it
  can no longer deliver into the group. (Removing *oneself* is **Leave**, below,
  and needs no admin.)
- **Rename / set photo.** An **admin** renames the group or sets its photo. A
  rename bumps the epoch and broadcasts a fresh `group.roster` (the authoritative
  name) plus a `group.rename` notice; a photo broadcasts a `group.avatar`. Both
  are signed group messages; recipients drop one whose signer is not a current
  admin. The photo is **not** in the roster, so a member added later does not
  receive an existing photo until it is next set.
- **Promote / demote admin.** An admin grants or revokes another member's admin
  flag: it bumps the epoch and broadcasts a fresh `group.roster`. Recipients apply
  it because the signer is an admin they already recognize; a newly-promoted member
  becomes a recognized admin for subsequent updates. This is the multi-admin
  governance that lets a group survive the creator's account loss.
- **Leave (self-service).** **Any** member can leave on their own, with no admin
  involvement: it sends a **signed** `group.leave`, revokes the pool it issued (so
  no member can deliver to it in this group again), erases its local group state
  and transcript, and **tombstones** the `groupId`. Each recipient verifies the
  signature and removes exactly the signer (a member cannot forge another's
  leave). A tombstoned id is never re-created from a late item (a stray
  roster/token/content message a member sent before the revocation reached them is
  dropped on sync), so the deleted chat cannot reappear. A genuine re-`group.invite`
  lifts the tombstone. Admins later fold the departure into the next roster; until
  then, current members already stop addressing the departed member.
- **Refill.** As in direct messaging (`Contacts.md`), a member whose issued pool
  is running low broadcasts a fresh `group.tokens`.

### Sender authenticity

A shared pool means the delivery token no longer identifies the sender (unlike a
one-to-one stash), so each group **content** message carries a per-message hybrid
signature, `gsig`. The sender signs `{type, id, from, groupId, sentAt, text, ref}`
with their identity key (`cms::signJsonHybrid` - the same self-contained hybrid-PQ
signature the roster uses, which yields the signer's identity fingerprint). The
signed body also includes the sender's own display name `dn`, so a member cannot
put a different self-name on another member's message. The signed `ref` binds the
target of an `edit`, `reaction` or `receipt`, so a member can neither forge another
member's reaction/receipt nor re-point a signed one at a different message;
`group.rename` likewise signs its new name. On receipt the recipient verifies it and takes
the signature's **identity fingerprint as the authoritative sender**, requiring
that it equals the message's `from`, matches the signed identifying/content
fields, and - for a group it already knows - is a current member. For the
admin-only `group.avatar` and `group.rename`, the recipient additionally requires
the verified signer to be a current **admin**. Anything unsigned, malformed,
field-mismatched, or from a non-member is **dropped, never surfaced**, so a member
cannot forge another member's `from`. (A message that arrives before its roster
defers only the membership check; the signature's from-authenticity is still
enforced.) `gsig` rides inside the E2E-sealed content, so it is invisible to
servers. The roster (membership/governance) keeps its own admin signature; this
is the message-level counterpart.

### Member display names

A group message carries the sender's own display name in `dn` (signed, above). A
reader shows a member by the **local contact name** they keep for them, if any;
otherwise by that `dn` (the member's self-chosen name, rendered in a distinct
colour as unverified) with the short fingerprint beneath it; otherwise by the
short fingerprint alone. `dn` is learned and cached per member as their messages
arrive; the fingerprint, shown for any non-contact, stays the ground truth.

### Adding a member as a contact

A member can add another member as a one-to-one contact straight from the roster,
without an invite link or alias. The routing comes from the (admin-signed)
roster, but the contact **card is fetched and verified end-to-end against the
fingerprint** (`Contacts.md`) - so a hostile admin who forged a member's routing
can only make the add **fail**, never substitute a key. The request is a direct,
E2E-sealed `contact.request` to that member **only** - it is never a group
message, so no other member sees it, and the two parties exchange nothing else
until the member agrees (the standard request -> `contact.accept` handshake in
`Contacts.md`). The group roster is unchanged; the new contact and its chat are
private to the two parties.

### Known limits (v1)

- **Ordering** follows the same `sentAt` (unix ms) rule as one-to-one chats
  (`Messages.md`): recent arrivals are sorted by `sentAt`, late ones are appended.
  With multiple senders it stays best-effort - each `sentAt` is its own sender's
  clock - so there is no global order.
- **Attachment fan-out** costs xN uplink (one blob-storage upload, N pointers -
  the sealed blob pointer rides each member's sealed message). Mitigations deferred.
- **Scale** <= ~100 members (see *Decided direction*). Larger broadcast is a
  separate future concept.

## Still open

- Roster conflict resolution beyond `epoch` (concurrent admin updates at the same
  epoch - last-writer-by-signer-fingerprint tiebreak is a candidate).
- Group calls.
