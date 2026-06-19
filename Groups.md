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
alone - no per-member contact lookup. A recipient **verifies** the embedded signature, checks the signer's
fingerprint is in `admins`, and only then trusts the body. The trust anchor for a
fresh member is the inviter (a known contact) who signed/forwarded the first
roster; later rosters are accepted when signed by any listed admin and carrying a
**higher `epoch`**.

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
| `group.leave` | contact | The sender is leaving: `groupId`. |
| `group` field | - | Any content message (`text`, `file`, ...) may carry `"group": { "id": <groupId> }`. It rides the existing type; the recipient files it under that group. Not a type of its own. |

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
- **Remove.** An admin signs a new roster (without the member, `epoch+1`) and
  broadcasts a `group.roster`; every member revokes the pool tokens it issued
  (deletes the hashes at its server) and broadcasts a **fresh** `group.tokens` to
  the remaining members. The removed member's stale tokens stop working at the
  servers, so it can no longer deliver into the group.
- **Leave.** A member sends `group.leave` and drops its state; admins fold the
  departure into the next roster.
- **Refill.** As in direct messaging (`Contacts.md`), a member whose issued pool
  is running low broadcasts a fresh `group.tokens`.

### Sender authenticity

A shared pool means the delivery token no longer identifies the sender (unlike a
one-to-one stash), so each group **content** message carries a per-message hybrid
signature, `gsig`. The sender signs `{type, id, from, groupId, sentAt, text}` with
their identity key (`cms::signJsonHybrid` - the same self-contained hybrid-PQ
signature the roster uses, which yields the signer's identity fingerprint). On
receipt the recipient verifies it and takes the signature's **identity
fingerprint as the authoritative sender**, requiring that it equals the message's
`from`, matches the signed identifying/content fields, and - for a group it
already knows - is a current member. Anything unsigned, malformed,
field-mismatched, or from a non-member is **dropped, never surfaced**, so a member
cannot forge another member's `from`. (A message that arrives before its roster
defers only the membership check; the signature's from-authenticity is still
enforced.) `gsig` rides inside the E2E-sealed content, so it is invisible to
servers. The roster (membership/governance) keeps its own admin signature; this
is the message-level counterpart.

### Known limits (v1)

- **Ordering** is best-effort by `sentAt` plus local arrival; there is no global
  order.
- **Attachment fan-out** costs xN uplink (one blob-storage upload, N pointers -
  the sealed blob pointer rides each member's sealed message). Mitigations deferred.
- **Scale** <= ~100 members (see *Decided direction*). Larger broadcast is a
  separate future concept.

## Still open

- Roster conflict resolution beyond `epoch` (concurrent admin updates at the same
  epoch - last-writer-by-signer-fingerprint tiebreak is a candidate).
- Group calls.
