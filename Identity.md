# Identity & Cryptography

## Principles

- OpenSSL only; standardized algorithms and containers: keys in PEM and DER, certificates as one composite CBOR frame, CMS for the password envelope alone. No custom cryptographic constructions (codestyle requirement).
- Every long-lived identity is a **hybrid keypair set** (classical + post-quantum, see Algorithms below); human-readable names are optional, revocable mappings on top.

## User identity

- A user **is** a hybrid keypair set. The canonical UID is the identity fingerprint covering both public keys.
- Address / contact forms:
  - `<fingerprint>:<b33>` - the **descriptor**: identity fingerprint + the user's own destination, addressed by the blinded b33 of its encrypted LeaseSet2 (written as a `.b32.i2p` host with a 56-character label, where a plain b32 has 52). `fp:b33` is this specification's shorthand; what a client shows, encodes in a QR and accepts is the link `bazarish://invite?v=1&fp=...&dest=...[&name=...]` (`api/FederatedResolve.md`). A fingerprint is an identity, not a route, so it is always paired with a destination - there is no `@server-fingerprint` form (nothing maps a fingerprint to an endpoint).
  - `<alias>` - an exclusive **alias** on the central resolver (paid; the project's monetization, `api/FederatedResolve.md`). Resolving it yields a descriptor, after which the contact card is fetched and verified.
- Users have no binding to a server in anything a contact can see. Registration opens an unbounded channel with the server, and the only thing that expires is the delegation of the user's destination - which the network itself enforces. Nothing signed by the user names the server.
- All clients (devices/app instances) of a user **share the identity keypair set**; a new device receives it during enrollment (see `DeviceSync.md`). Each client registers a unique random **client ID** so the server can track delivery completeness across clients.
- The user's **I2P destination is the account's, not a device's**: contacts hold that one address, so every device of the account operates the same master key. A device that arrives without it takes it from the account (its enrollment material, or `device.i2p-master-request` to the account's other devices) rather than minting one, because a second address delegated to the server strands every contact holding the first. A delivery pass follows the same rule for the same reason: it is the account's, not a device's, so it travels in the address book and every device presents the same one (`DeviceSync.md`).

## Server identity

- A server's root identity is a hybrid keypair set.
- **Server card** - a document signed by the server's root key, binding the
  server's identity to a server-level sealing key
  (`{ v, t: "server-card", server, sealingKey, issuedAt }`; it names no transport
  endpoint). It is the server's own identity document (the service node is
  bound to it), and it is also what a **client** fetches in the open before it can
  speak in private: the messaging tunnel is sealed to the sealing key in it, and the card is
  trusted because it is signed by the fingerprint the user already had
  (`api/ClientApi.md`). That sealing key is hybrid (X25519 + ML-KEM-768), as every
  sealing key in the project is.
  It is **not** the peer-facing routing artifact and is **not**
  carried in invites: delivery routing and sealing are per-destination and travel
  in the user's contact card (below); a contact fetches that card from the
  destination the **descriptor** names, with no key and no server card
  (`api/FederatedResolve.md`). Cross-server
  alias resolution is served by the separate central resolver
  (`api/FederatedResolve.md`), which signs its own records.
- **Delivery sealing is per-destination.** Each user's own serving destination
  has
  its **own serving sealing key**; the delivery envelope's admission header
  (recipient, pass, deliveryId) is sealed to the serving sealing key of the
  recipient's destination, whose private half the recipient's server holds. There
  is no single server-wide delivery sealing key (`InviteAnonymity.md`).
- **Federation signing is per-destination too.** Each serving destination also
  has its **own signing keypair**, bound to that destination together with its
  serving sealing key - a `(destination, signing key, serving sealing key)`
  group that is never reused on another destination.
- **The account issues all three.** The destination's master key, its serving
  sealing key and its signing key are made by the client together, kept in the
  account and its backup, and handed to the server with each delegation
  (`api/ClientApi.md`). A server operates them and generates no key of its own
  for an address; its own keys sign and seal only what concerns the server itself
  (its card and the facade tunnel). So an address and its keys do not depend on
  the server that operates them, which is what lets an account move. It is the only key that signs
  live federation traffic (a delivery confirmation, `api/Federation.md`). The server
  **root identity** signs only **outside** federation - the server card above
  (the `bazarish://server` configuration link carries no signature, only the
  fingerprint that card is verified against, `api/Facade.md`) and the
  service-node binding - so two users one server hosts share no signer and cannot be
  clustered by a common server fingerprint: no two users share a destination, so
  none share its signing key (`InviteAnonymity.md`).

## Certificates

**Container for all certificates: one composite CBOR frame over a JSON body** (the body format matches the system-wide JSON wire format). Each body carries a format version field.

### Contact card

Issued **by the user** and signed with their identity key: the cryptographic
answer to "where and how do I deliver to U?". Signed by U, so it stays
trustworthy through any untrusted intermediary.

Body: `{ v, t: "contact-card", issuedAt, kind, dest, sealingKey, servingKey,
presenceKey, presenceDest }`.

- `t` - what this document is. Checked before any of the body is believed, so
  that one signed document cannot be read as another: with optional fields and no
  type, a subscription certificate parsed cleanly as a contact card, and the only
  thing preventing that was which fields the reader happened to examine.
- `issuedAt` - when it was signed. Two cards of one person are ordered by this
  and by nothing else, and the node that holds one **refuses any that is not
  newer** (`api/ClientApi.md`). Without it the current card is whichever was
  published last, so a second device holding an old copy could republish it and
  silently revert the account's routing - valid signature, correct owner, but
  stale.
- `kind` - `user`, `bot` or `chat` (a group or channel host, `Groups.md`): what
  kind of account signed the card. It is fixed when the account is made, every
  card carries it, and a card without it is refused. A client draws a contact by
  it; it is the owner's own statement, so it decides rendering and never trust.
- `sealingKey` - the user's E2E sealing key (SPKI DER, base64): what a contact
  encrypts to. The very first contact request is sealed to it before any pass
  exchange, and so is **every message afterwards** - the card is where a stranger
  learns it, not a key reserved for first contact. Because it is carried inside
  the user-signed card, no intermediary can substitute it.
- `dest` - the user's serving **destination** as a `.b32.i2p` host - the blinded
  **b33** of its encrypted LeaseSet2, like every other destination this project
  publishes: where a contact routes deliveries. Never a raw destination or an
  addressbook name (`api/Federation.md`).
- `servingKey` - the public **serving sealing key** of that destination: a contact
  seals the delivery envelope's admission header to it. Not a signature and not a
  trust anchor - the user only vouches for it under their one signature; a wrong
  value only makes delivery fail (`InviteAnonymity.md`). The account issues it
  and hands its private half to the server that operates the destination, which
  is what lets the server route without being able to read: the message itself is sealed to `sealingKey`, whose private half never
  leaves the user's device.
- `presenceKey` - the account's X25519 presence key (SPKI DER, base64, a bare
  X25519 key and not a sealing pair), and `presenceDest` - the b33 of its
  presence destination (`Presence.md`). A `user` card carries both and a card of
  another kind carries neither; any other combination is refused. Both are made
  with the account and never change.

**The owner is not a field.** A card is a statement its signer makes about
themselves, so the subject is taken from the keys that signed it. A name written
beside the signature would be a second place for one fact, and the only thing a
second place can add is a disagreement with the first. A reader who wants a
particular person compares fingerprints - which is where that check belongs,
because only the reader knows whose card they asked for.

What is deliberately **absent**: any server fingerprint or server card (routing
is by destination, and naming the server would let cards be grouped by who hosts
them); and anything about the owner's devices - the presence destination is one
per account, whichever device holds it.

**Forward secrecy: there is none, and the card is where that shows.**
`sealingKey` is long-lived and never rotated, so whoever obtains its private half
reads everything that account was ever sent - including what is still stored in
mailboxes. `servingKey` is issued anew whenever the account raises a new
destination, and only then.

This is a decision rather than an oversight, and the reasoning is worth keeping
because the question recurs. Forward secrecy bounds what a key obtained later
can reopen in ciphertext somebody hoarded. It does not bound what an account's own
database reveals: the transcript is stored decrypted inside it, so a device and
its passphrase yield the history whatever the message keys did. The threat this
protocol is built against is that database, not a recording of the traffic - so
neither key rotation nor a ratchet would answer it, and both add substantial
complexity: rotation adds retired keys and the silent loss of mail from a contact
who has been away too long, a ratchet adds state that every device of an account
would have to advance in lockstep while each fetches its mail independently.

Reopen it if something changes the premise - for example, message history
stored under a key of its own, which is what would make the ciphertext hoard the
weak point again.

A card is republished to **the server** whenever its contents change. When the
routing it carries changes - a new destination, or a serving key the account
issued anew - contacts are told once, by a `contact.routing` message carrying
the new destination and serving key (`Messages.md`); nothing is carried in
ordinary messages, and a card is not re-broadcast on a schedule.

**Raising a new destination is one action, not three.** A client offers a single
destructive control - *New key* - and it mints a fresh offline key together with
a fresh serving sealing key and signing key, delegates the address with them
(`api/ClientApi.md`), publishes the card, hands the new master and keys to the
account's own other devices (`device.i2p-master`, `DeviceSync.md`), and only then
starts the fan-out. There is no separate way to issue the serving key anew and no
grace period on the old address: the keys follow the destination, and the old
destination stops answering inside the same call, which is the intended effect
when the reason for moving is a flood. The invite links already distributed stop
working, and the aliases are repointed automatically.

### Moving to another server

A user is served by **exactly one server at a time**. The address and its keys
belong to the account, not to the server, so a move changes nothing a contact
holds. Moving S1 -> S2:

1. Register on S2.
2. Register there the hash of every pass the account has issued: each is derived
   from the account's delivery secret and the contact's fingerprint
   (`Contacts.md`), so the client computes them again rather than keeping a list.
3. Delegate the same address to S2 with the same serving and signing keys, and
   publish the same card there.
4. The client no longer renews the delegation on S1, which runs out by the end
   of its term, and S1 stops publishing the address. What reached S1 meanwhile
   stays there: the client does not fetch it or close the account on S1 by
   itself.

No `contact.routing` is sent and nothing on the wire tells anyone that the server
changed: contacts deliver to the same address with the same key and pass.
Deliveries that reach S1's lease set while senders still hold it fail and stay on
the sender's client (standard send semantics), to be retried - messages are
delayed, not lost. S1 need not be reachable for any of this: a move works the
same when S1 is gone.

### Alias certificate

- The user signs the desired nickname: `{ v, t: "alias-certificate", alias, user: <fingerprint>, issuedAt }`; the central resolver registers the mapping alias -> UID (`api/FederatedResolve.md`).
- The certificate carries no term: an alias is rented, and the resolver keeps its term beside the claim.

### First-contact trust chain (alias resolution)

Resolving an `alias` returns a **descriptor** (`fingerprint`, serving destination)
signed by the central resolver (a delegation chain to a
hardcoded root fingerprint, `api/FederatedResolve.md`). The client then fetches and
verifies the user-signed **contact card** for that fingerprint. Two trust layers:

1. **alias -> fingerprint** is asserted by the resolver - the inherent trust of
   names; the UI surfaces the resolved fingerprint for out-of-band verification.
2. everything after - the contact card (sealing key + destination + serving
   key) - is **user-signed and verified against the fingerprint**, so no
   intermediary can forge it.

The lookup travels over I2P, as a direct fetch from a transient I2P destination
of the client's own, so the requester's own server never learns which name was
looked up (`api/FederatedResolve.md`).

### Out-of-band invites (the descriptor)

A contact is shared out of band as a small **descriptor** - the link
`bazarish://invite?v=1&fp=...&dest=...[&name=...]` (`fp:b33`): identity +
serving destination, plus an optional advertised
display name the recipient adopts at add time (static thereafter; see `Messages.md`).
It fits a single QR. The recipient
fetches the user-signed **contact card** for that fingerprint from the destination
and verifies it against the fingerprint, so the binding is cryptographic and
intermediaries are trusted for availability only. The descriptor is small and the
card always fresh, at the cost of a live fetch at add-time
(`api/FederatedResolve.md`).

Trust ladder for adding a contact, strongest first:

- **By descriptor / fingerprint** (`fp:b33`, a QR or link) - the contact card is
  fetched and verified against the fingerprint, so a server cannot forge, only
  withhold; the fetch is direct, over a transient I2P destination, so the requester's own server cannot even observe the lookup.
- **By alias** - the central resolver maps the human-readable name to a
  fingerprint. Convenient, but the **alias->fingerprint mapping is trusted**: a
  hostile resolver could return an attacker's fingerprint, so the UI surfaces the
  resolved fingerprint for out-of-band verification. Everything after the mapping
  is verified as above.

## Algorithms (decided empirically against OpenSSL 3.5)

- **Identity is a post-quantum hybrid: Ed25519 + ML-DSA-65 (FIPS 204).** Every identity-level statement (contact card, alias certificate, server card) carries **both** signatures and is valid only when both verify - forging requires breaking both schemes. This is mandatory, not optional. The one exception is a presence watch, a request checked once on arrival and dropped after 120 seconds, which carries the Ed25519 signature alone under its own domain string (`Presence.md`).
- **One composite frame, no CMS.** OpenSSL CMS carries neither signer type: measured against 3.5.7, `CMS_add1_signer` refuses Ed25519 with `no default digest`, and an X25519 recipient is refused with `operation not supported for this keytype`. So a statement is one CBOR frame built from standard primitives: `{v, body, c: {alg, pub, sig}, q: {alg, pub, sig}}`, with `body` the exact signed bytes. Both halves sign the **same** message: the domain-separation string, then SHA-256 over `Ed25519 SPKI DER || ML-DSA SPKI DER`, then the body bytes. Naming both keys in the signed message is what stops either half being copied into a frame that names a different key. Verification checks both signatures **and both key types** (a classical key in the pq slot is a downgrade and fails).
- **Identity fingerprint**: lowercase unpadded base32 (RFC 4648) of SHA-256 over the concatenation `Ed25519 SPKI DER || ML-DSA SPKI DER` - 52 characters, covering both keys; substituting either key changes the identity. Display shortening is a client UX concern.
- **Sealing keys: X25519 + ML-KEM-768 (FIPS 203), one composite KEM.** A sealing key is distinct from the identity keys by design, and it is a pair: its public form is the CBOR `{t, c, q}` carrying both public halves (1268 bytes, measured).
  - A sealed blob is **one** AES-256-GCM layer under a key derived with HKDF-SHA256 from both shared secrets at once - `ikm = ss_ML-KEM || ss_X25519`, salt = the frame's transcript (KEM ciphertext, ephemeral X25519 public key, recipient public pair), info = the seal's domain-separation string - carried as `{v, kem, eph, n, ct}`. Opening needs **both** private halves, so breaking one scheme alone reveals nothing, and no public field of the frame can be moved without changing the derived key. This covers everything encrypted in the project - delivery-envelope headers, message bodies end to end, federation queries - because they all go through the same `hybrid::seal`. Overhead over the plaintext is 1173 bytes, measured.
  - **CMS stays for the password envelope only** (RFC 3211 PWRI, AES-256-CBC): the account backup and the file-transfer payload are keyed by a password, involve no asymmetric key, and stream, so the standard container remains in use there.
- **Alias syntax**: `a-z` and `0-9` only, case-insensitive (normalized to lowercase), 1-16 characters (`api/FederatedResolve.md`). The length cap keeps an alias distinct from the 52-character fingerprint form.

Implementation: the `common` repository (`Crypto`, `Hybrid`, `Certificates` modules; `Cms` is the password envelope alone). Tests cover tamper, downgrade and half-splicing cases. Notable sizes, measured: ML-DSA-65 signature 3309 bytes and public key 1974 bytes against Ed25519's 64 and 44, so an alias certificate is 5593 bytes and a `user` contact card, which carries two sealing pairs and the presence fields, is 9172 (a `bot` card, without them, is 9012).
