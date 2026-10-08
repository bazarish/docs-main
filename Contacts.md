# Contact Admission: Delivery Passes

Status: implemented.

## Problem

A server stores incoming content only if the sender is an approved contact of the recipient. Holding that approval as a list of identities on the server would constitute a readable social graph, which the threat model does not permit. The server must therefore be able to *enforce* contact admission without being able to *read* the relationship.

## Scheme

A **delivery pass** is a 256-bit value the recipient's client derives for one correspondent and hands to them over the E2E channel. It does not expire, it is not spent, and every device that correspondent writes from presents the same one.

- The value is derived rather than drawn: `sha256(the account's static delivery secret || that correspondent's fingerprint)`. The secret is drawn once with the account, travels with its backup, and never leaves it. So an account keeps no list of what it issued, every device of it arrives at the same value for the same correspondent, and a restored device can write from its first sync.
- The recipient's client registers the **hash** of the pass with its own server (`POST /v1/messaging/passes`). The sender presents the pass itself, so what is on a node's disk recognises a delivery and cannot produce one.
- A sender attaches the pass to each message, inside the sealed envelope (see `api/Federation.md`) - readable by the recipient's server only.
- The recipient's server hashes the presented pass and looks it up in the mailbox's registered set. Absence of a match is a rejected delivery. Nothing is consumed: the same value admits the next message.
- No signatures involved; plain hashing only (standard primitives).

Properties:

- **No social graph at rest.** The server holds a flat set of opaque hashes per mailbox; nothing maps a hash to a contact.
- **One record per correspondent.** A mailbox holds as many passes as the account has contacts, and that number does not grow with traffic. The one-time scheme this replaced grew by a fresh batch on every refill and revoked the previous one only on removal; a single test account had accumulated sixty-one thousand.
- **Replay protection.** A pass is reusable, so what stops an envelope being offered twice is the server's retained **seen-deliveryId set**: every admitted `deliveryId` is remembered independently of the stored blob's lifecycle, so an already-delivered envelope cannot be re-admitted even after it has been fetched, acked and swept from the mailbox (see `api/Federation.md`). This is what guarded the tokenless path before, and it now guards both.
- **What is deliberately not bounded.** A pass does not limit how much a correspondent may write within it. The ceiling on a mailbox is its storage quota, which is where it always was in practice: the one-time scheme refilled itself automatically, so an active conversation was never really counted either. The cost is that one correspondent can fill a mailbox to the quota and leave nothing for the others; the remedy is blocking them, which is now exact and immediate.

## Pass lifecycle

1. **Issuance** - the recipient's client derives the pass for a contact, registers its hash with its own server (`POST /v1/messaging/passes`, see `api/ClientApi.md`) and delivers the pass inside an E2E message, in the same `bootstrap` block that carries routing. Registration must precede or accompany delivery. Registering one already held is not an error - see *Revocation*.
2. **Presentation** - the sender puts the pass in the envelope (`pass`); the recipient's server hashes it and looks it up. Weight has no price: a line of text and the largest payload the protocol allows are admitted alike, and what bounds the large one is `kMaxMessagePayloadBytes` and the mailbox quota. Processing order on the recipient's server: deliveryId dedup first (a retried message must succeed idempotently, checked against both the pending blob and the retained seen-deliveryId set), then size, then the pass, then the store.
3. **Revocation** - per correspondent, and asked for by the account that issued the pass. Its client sends the hash (`POST /v1/messaging/passes/revoke`); the server drops exactly that record and answers how many were there. Nothing is swept for and nothing else is touched, so there is no walk to queue and no accidental match. Removing or blocking a contact revokes what they hold; deleting the account discards all of them.

   Revocation is the only thing that stops a correspondent writing, so an answer the client did not hear is written down and asked again on the next sync rather than assumed. It is kept per account rather than on the contact, because removing a contact is one of the two things that revokes, and a flag on a record being deleted would go with it.
4. **Lifting a block** - the correspondent still holds the pass, so the account simply registers the same hash again. Nothing is sent to them, nothing waits for a message to ride on, and they can answer at once.
5. **No replacement** - a correspondent's pass is the same value for as long as the account lives. There is no way to hand them a different one: the derivation takes the account's secret and their fingerprint, so removing and re-adding them arrives back at the value they already hold. The only remedy for a pass in the wrong hands - a correspondent's device stolen, say - is to revoke it, which means parting with that correspondent rather than re-admitting them on new terms. This is a deliberate trade: a replacement mechanism costs a stored epoch, a way to deliver the new value, and a state where an account holds two passes for one correspondent, and none of that is worth carrying for a case that ends in blocking anyway.

Which pass an account holds for each correspondent is private client state, carried between the user's devices in the address book (`DeviceSync.md`). It is not spent, so every device holds the same one and there is nothing for two devices to race for.

## Contact request bootstrap

A is not B's contact and B is not A's - the first exchange must bootstrap both directions:

1. A's client derives its pass *for B* and registers its hash on A's own server.
2. A sends a **contact request** to B: a delivery admitted by nothing, flagged as a request, payload capped (17590 bytes, the measured worst case; a protocol constant, not an operator setting). The cap is on the delivered, sealed bytes and must comfortably exceed a real contact request: the E2E payload carries A's identity, the introduction message, A's self-chosen **display name** (`dn`, so B can seed a named friend - see "Display names" below), **the pass B will write back with**, plus A's routing **bootstrap** - `{ sealing key, dest, servingKey }` (see `../Identity.md`) - without which B would not know where to send the acceptance, so a sealed request weighs 17558 bytes at its worst, the authorship block that
introduces the sender's keys (`Messages.md`) being 5471 of them. The cap is that
number plus what the variable-length parts can add, and no more: room above the
worst case is room for a stranger to fill a mailbox with. (A cap below that silently rejects every contact request with `CONTACT_REQUEST_TOO_LARGE`. No server fingerprint or server card: routing is by destination.)
3. B accepts, and B's client sends a **`contact.accept`** (`Messages.md`), presenting the pass A handed over. As B's first reply it carries B's own pass and the same routing bootstrap in the E2E payload. A's client applies that bootstrap without user action and reports the acceptance. Both directions are then established. Acceptance is an explicit action; declining is not signalled.
   Until B accepts, **nothing whatsoever travels from B to A**, on any device of B's and whatever B's client would otherwise find useful to do. The pass the request offers is B's to hold from the moment it arrives - holding it is local - and it is presented no earlier than the acceptance itself. This is the same rule the avatar follows below.

   Acceptance is what B's pass **arriving** is, not what a send leaving is: B's client marks the contact accepted only once A's server confirms it stored the `contact.accept`, because until it is in A's mailbox A cannot answer, and a contact marked accepted on the strength of a send that never landed is a dialog that exists on one side. An acceptance that fails leaves the request exactly as it was. A second acceptance while the first is still in the air does nothing.

   Acceptance is the account's, not the device's: the device that agrees tells the others over `device.contact-accepted` (`DeviceSync.md`), so none of them goes on offering an Agree that has been given.

4. B declines by taking no action: nothing is transmitted. The hash A pre-registered stays until A removes the pending contact, which revokes it.

Contact requests are the only path into a mailbox that presents nothing; they are bounded by rate and by depth rather than by a quota of their own - a mailbox has one quota, and every class is stored against it. Two bounds, not one: **3 per minute** per serving destination (`kContactRequestsPerMinute`) says how fast they may arrive, and **30 waiting at once** (`kMaxPendingContactRequests`, counted over what the mailbox is still holding) says how many may pile up. Past either, the answer is the same transient refusal a rate-limited sender gets - come back later - which says nothing about who is served at that destination.

## Display names and avatars

A contact's **display name** is a local label, captured once at add time - from the invite descriptor's optional `name` or from the alias used - and never refreshed from the peer afterwards. A later rename is local, mirrored only to the user's own devices (`DeviceSync.md`).

The label is symmetric across the first exchange. The requester A names B from the descriptor/alias it added by. So B is not left with a bare fingerprint, A's **contact request carries A's own display name** (`dn`); B adopts it as the initial local label for A, exactly once, **only when B holds no name yet** (it never overwrites a name B already set). This is a one-time seed, not a live name channel: a later rename on either side is never pushed to the other.

A user's **avatar** (square, compressed client-side to <= 500 KB) is exchanged only **after the dialog is mutually established**: the accepting reply carries the accepter's avatar, and the requester sends theirs on receiving that acceptance. A bare contact request neither carries an avatar nor triggers an automatic avatar reply, so nothing of a user's picture leaks to a party who has not yet been accepted. Both ride the silent `avatar` (and self-channel `device.*`) content types in `Messages.md`.

## First-contact discovery and co-location

Before A can send the contact request above it must learn B's routing (`{ sealing key, dest, servingKey }`). One discovery path exists:

- **Invite / alias** - A holds a self-verifying descriptor (fingerprint + serving destination + serving key) and fetches B's user-signed card **over I2P federation**, dialing B's serving destination directly. The card is verified against the fingerprint, so no server is trusted (see `api/FederatedResolve.md`).


**Co-location rule (mandatory).** Two clients of the same server always have *different* I2P destinations - every account owns one - so first-contact discovery goes over I2P and there is no by-fingerprint lookup through a facade at all: one existed in an earlier draft, and a lookup that told a shared server "client A is asking for client B" would have disclosed that A and B are **co-located on it**.

The same rule binds the answering end, which is the harder half. A card fetch arrives on a destination, and that destination answers for **its own user only**. Answering for any fingerprint the node happens to serve turned every published address into an oracle: dial A's destination, ask for B, and a card back proves the two are served by the same server. Nothing exotic is needed to try it - a fingerprint and a destination, and every contact holds both.

For the same reason the contact request **delivery** is routed by destination over federation (I2P) in every case, including one in which the same server operates both destinations. No local short-circuit exists; the sending client performs the delivery, so no server is in a position to take one. See `api/Federation.md`.

## Residual disclosure to the server

- **Set size** is the number of contacts, which is the degree of the contact graph rather than an estimate of it. Mitigation: clients may pad the set with unused hashes. The one-time scheme hid this behind a stash size but leaked the same grouping anyway, since two of its tokens for one correspondent XORed to a value ending in four zero bytes.
- **Registration timing**: a pass registered immediately after an outgoing contact request indicates that it belongs to that pending contact. Mitigation: randomised delay and pre-registration padding. Recorded in the threat model; not a release blocker.
- **Traffic volume per mailbox** is inherent to any store-and-forward server.

## Anti-enumeration

A destination belongs to one user, and only that user's mail is stored in it. A
contact request naming any other mailbox receives the same response as a
delivered one and is discarded; refusing it would disclose which user the
destination serves. Such a request does not enter the seen-set, so it cannot be
used to suppress the genuine delivery of the same id. A pass-bearing delivery for another mailbox is
refused like any other bad admission.

Unknown pass, unknown mailbox, and a recipient not served here must be externally indistinguishable (single `DELIVERY_REJECTED` error). Two refusals the one-time scheme had are simply gone: nothing can be "already spent", and no envelope can present the wrong number of them. Detailed typed errors (`STORAGE_FULL` and `MESSAGE_TOO_LARGE`, which the sender must see explicitly) are returned only after a valid pass has been presented - so only actual contacts can learn mailbox state. The order that makes this true is in `api/Federation.md`, and it is load-bearing rather than incidental.

## Open questions

- **Defaults** - `kMaxPassesPerMailbox` and `kMaxPassesPerRequest` bound what one account may register; whether either belongs in server config rather than the protocol is open.
- **Fairness** - nothing keeps one correspondent from filling a mailbox to the quota. Whether a per-pass rate limit is worth its state is open, and deliberately not built yet.
