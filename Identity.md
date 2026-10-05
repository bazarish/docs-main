# Identity & Cryptography

Status: draft skeleton.

## Principles

- OpenSSL only; standardized algorithms and containers (PEM, DER, X.509, CMS). No custom cryptographic constructions (codestyle requirement).
- Every long-lived identity is a **hybrid keypair set** (classical + post-quantum, see Algorithms below); human-readable names are optional, revocable mappings on top.

## User identity

- A user **is** a hybrid keypair set. The canonical UID is the identity fingerprint covering both public keys.
- Address / contact forms:
  - `<fingerprint>:<b32>` - the **descriptor**: identity fingerprint + the user's own destination (a `.b32.i2p` host). The primary scannable/QR form; the full link `bazarish://invite?fp=...&dest=...&view=...` also carries the card-read capability (`api/FederatedResolve.md`). A fingerprint is an identity, not a route, so it is always paired with a destination - there is no `@server-fingerprint` form (nothing maps a fingerprint to an endpoint).
  - `<alias>` - an exclusive **alias** on the central resolver (paid; the project's monetization, `api/FederatedResolve.md`). Resolving it yields a descriptor, after which the contact card is fetched and verified.
- Users have no binding to a server in anything a contact can see. Registration opens an unbounded channel with the server, and the only thing that expires is the delegation of the user's destination - which the network itself enforces. Nothing signed by the user names the server.
- All clients (devices/app instances) of a user **share the identity keypair set**; a new device receives it during enrollment (see `DeviceSync.md`). Each client registers a unique random **client ID** so the server can track delivery completeness across clients.
- The user's **I2P destination is the account's, not a device's**: contacts hold that one address, so every device of the account operates the same master key. A device that arrives without it takes it from the account (its enrollment material, or `device.i2p-master-request` to the account's other devices) rather than minting one, because a second address delegated to the server strands every contact holding the first. A delivery pass follows the same rule for the same reason: it is the account's, not a device's, so it travels in the address book and every device presents the same one (`DeviceSync.md`).

## Server identity

- A server's root identity is a hybrid keypair set.
- **Server card** - a document signed by the server's root key, binding the
  server's identity to its transport endpoint(s) (`i2p:<b33>`) and a server-level
  sealing key. It is the server's own identity document (the service node is
  bound to it), and it is also what a **client** fetches in the open before it can
  speak in private: the tunnel is sealed to the sealing key in it, and the card is
  trusted because it is signed by the fingerprint the user already had
  (`api/ClientApi.md`). That sealing key **must be hybrid** - sealing needs the
  ML-KEM half, and a server whose key predates hybrid sealing keys can serve no
  client at all, so it refuses to start rather than fail one connection at a time.
  It is **not** the peer-facing routing artifact and is **not**
  carried in invites: delivery routing and sealing are per-destination and travel
  in the user's contact card (below); a contact discovers the serving sealing key
  needed to fetch that card from the **descriptor** (`key`,
  `api/FederatedResolve.md`), not from a fetched server card. Cross-server
  alias resolution is served by the separate central resolver
  (`api/FederatedResolve.md`), which signs its own records.
- **Delivery sealing is per-destination.** Each user's own serving destination
  has
  its **own serving sealing key**; the delivery envelope's admission header
  (recipient, pass, deliveryId) is sealed to the serving sealing key of the
  recipient's destination, whose private half the recipient's server holds. There
  is no single server-wide delivery sealing key (`InviteAnonymity.md`).
- **Federation signing is per-destination too.** Each serving destination also
  has its **own signing keypair**, stored bound to that destination together with
  its serving sealing key - a `(destination, signing key, serving sealing key)`
  group that is never reused on another destination. It is the only key that signs
  live federation traffic (a delivery confirmation, `api/Federation.md`). The server
  **root identity** (the server card above) signs only **outside** federation -
  the `bazarish://server` configuration link (`api/Facade.md`) and the
  service-node binding - so two users one server hosts on different destinations
  share no signer and cannot be clustered by a common server fingerprint; users
  sharing a destination share its signing key, which is the same k-anonymity set
  that already shares the destination (`InviteAnonymity.md`).

## Certificates

**Container for all certificates: CMS SignedData over a JSON body** (OpenSSL-native, standard; the body format matches the system-wide JSON wire format). Each body carries a format version field.

### Contact card

Issued **by the user** and signed with their identity key: the cryptographic
answer to "where and how do I deliver to U?". Signed by U, so it stays
trustworthy through any untrusted intermediary.

Body: `{ v, t: "contact-card", issuedAt, dest, sealingKey, servingKey }`.

- `t` - what this document is. Checked before any of the body is believed, so
  that one signed document cannot be read as another: with optional fields and no
  type, a subscription certificate parsed cleanly as a contact card, and the only
  thing standing in the way was which fields the reader happened to look at.
- `issuedAt` - when it was signed. Two cards of one person are ordered by this
  and by nothing else, and the node that holds one **refuses any that is not
  newer** (`api/ClientApi.md`). Without it the current card is whichever was
  published last, so a second device holding an old copy could republish it and
  put the account's routing quietly back to what it used to be - valid signature,
  right owner, simply stale.
- `sealingKey` - the user's E2E sealing key (SPKI DER, base64): what a contact
  encrypts to. The very first contact request is sealed to it before any pass
  exchange, and so is **every message afterwards** - the card is where a stranger
  learns it, not a key reserved for first contact. Because it rides inside the
  user-signed card, no intermediary can substitute it.
- `dest` - the user's serving **destination** as a `.b32.i2p` host - the blinded
  **b33** of its encrypted LeaseSet2, like every other destination this project
  publishes: where a contact routes deliveries. Never a raw destination or an
  addressbook name (`api/Federation.md`).
- `servingKey` - the public **serving sealing key** of that destination: a contact
  seals the delivery envelope's admission header to it. Not a signature and not a
  trust anchor - the user just vouches for it under their one signature; a wrong
  value only makes delivery fail (`InviteAnonymity.md`). Its private half is held
  by the user's server, which is what lets the server route without being able to
  read: the message itself is sealed to `sealingKey`, whose private half never
  leaves the user's device.

**The owner is not a field.** A card is a statement its signer makes about
themselves, so the subject is taken from the keys that signed it. A name written
beside the signature would be a second place for one fact, and the only thing a
second place can add is a disagreement with the first. A reader who wants a
particular person compares fingerprints - which is where that check belongs,
because only the reader knows whose card they asked for.

What is deliberately **absent**: any server fingerprint or server card (routing
is by destination, and naming the server would let cards be grouped by who hosts
them); the view capability that permits fetching the card (it travels in the
invite - putting it inside would hand it to everyone who already has the card);
and anything about the owner's devices.

**Forward secrecy: there is none, and the card is where that shows.**
`sealingKey` is long-lived and never rotated, so whoever obtains its private half
reads everything that account was ever sent - including what is still sitting in
mailboxes. Compare `servingKey`, which does have a rotation mechanism.

This is a decision rather than an oversight, and the reasoning is worth keeping
because the question comes back. Forward secrecy bounds what a key obtained later
can reopen in ciphertext somebody hoarded. It does not bound what an account's own
database gives up: the transcript is stored decrypted inside it, so a device and
its passphrase yield the history whatever the message keys did. The threat this
protocol is built against is that database, not a recording of the traffic - so
neither key rotation nor a ratchet would answer it, and both cost real
complexity: rotation adds retired keys and the silent loss of mail from a contact
who has been away too long, a ratchet adds state that every device of an account
would have to advance in lockstep while each fetches its mail independently.

Reopen it if something changes the premise - message history stored under a key
of its own, say, which is what would make the ciphertext hoard the weak point
again.

A card is republished to **the server** whenever its contents change (a new
destination, a rotated serving key); contacts learn a new destination from the
`routing` that rides on every message (`Architecture.md`), so a card is not
re-broadcast to them on a schedule.

### One serving server, hard-cutover migration

A user is served by **exactly one server at a time**. Migration S1 -> S2:

1. Register on S2, register the delivery passes there, close the account on S1 (or let it go idle).
2. Push the new contact card (S2 destination + serving key + sealing key) to all contacts over E2E.
3. Contacts switch as the card arrives. Deliveries that race the cutover **fail and stay on the sender's client** (standard send semantics) and are retried after the new card arrives - messages are delayed, not lost.

Accepted failure mode: if S1 becomes permanently unavailable before the user has prepared the migration, deliveries from contacts continue to fail until the user publishes a new card (or, where an alias is used, until contacts resolve it again). A user's destination is portable across servers (it is their own offline key), so migration need not change the destination at all.

### Alias certificate

- The user signs the desired nickname: `{ v, alias, user: <fingerprint>, issuedAt, notAfter? }`; the service node registers the mapping alias -> UID.
- Validity term is set by the server; default unlimited.

### First-contact trust chain (alias resolution)

Resolving an `alias` returns a **descriptor** (`fingerprint`, serving destination,
serving sealing key) signed by the central resolver (a delegation chain to a
hardcoded root fingerprint, `api/FederatedResolve.md`). The client then fetches and
verifies the user-signed **contact card** for that fingerprint. Two trust layers:

1. **alias -> fingerprint** is asserted by the resolver - the inherent trust of
   names; the UI surfaces the resolved fingerprint for out-of-band verification.
2. everything after - the contact card (sealing key + destination + serving
   key) - is **user-signed and verified against the fingerprint**, so no
   intermediary can forge it.

The lookup travels over I2P (a direct fetch over a transient I2P destination, or
sealed via the client's own server), so the requester's own server never learns which name was looked up
(`api/FederatedResolve.md`).

### Out-of-band invites (the descriptor)

A contact is shared out of band as a small **descriptor** - `<fingerprint>:<b33>`,
or the link `bazarish://invite?fp=...&dest=...&view=...[&name=...]`: identity +
serving destination + the server's serving sealing key, plus an optional advertised
display name the recipient adopts at add time (static thereafter; see `Messages.md`).
It fits a single QR. The recipient
fetches the user-signed **contact card** for that fingerprint from the destination
and verifies it against the fingerprint, so the binding is cryptographic and
intermediaries are trusted for availability only. This replaces the earlier
full-card invite (the whole ~= 7-8 KB user-signed card inline, spanning several QR
symbols): smaller and always-fresh, at the cost of a live fetch at add-time
(`api/FederatedResolve.md`).

Trust ladder for adding a contact, strongest first:

- **By descriptor / fingerprint** (`fp:b33`, a QR or link) - the contact card is
  fetched and verified against the fingerprint, so a server cannot forge, only
  withhold; on a direct fetch over a transient I2P destination it cannot even observe the lookup.
- **By alias** - the central resolver maps the human-readable name to a
  fingerprint. Convenient, but the **alias->fingerprint mapping is trusted**: a
  hostile resolver could return an attacker's fingerprint, so the UI surfaces the
  resolved fingerprint for out-of-band verification. Everything after the mapping
  is verified as above.

## Algorithms (decided empirically against OpenSSL 3.5)

- **Identity is a post-quantum hybrid: ECDSA P-256 + ML-DSA-65 (FIPS 204).** Every identity-level statement (contact card, alias certificate, server card) carries **both** signatures and is valid only when both verify - forging requires breaking both schemes. This is mandatory, not optional.
- **Hybrid layering.** OpenSSL CMS cannot carry ML-DSA (or EdDSA) signers, so the hybrid is layered from standard primitives with no custom constructions: the exact body bytes are signed with ML-DSA-65 at the EVP level; the wrapper `{body: base64(bodyBytes), pq: {alg, pub, sig}}` is then CMS-SignedData-signed with the classical key. Verification checks both layers **and both key types** (a classical key smuggled into the pq slot is a downgrade and fails). Replace with CMS composite signatures when OpenSSL supports the LAMPS composite drafts.
- **Identity fingerprint**: lowercase unpadded base32 (RFC 4648) of SHA-256 over the concatenation `classical SPKI DER || ML-DSA SPKI DER` - 52 characters, covering both keys; substituting either key changes the identity. Display shortening is a client UX concern.
- **Sealing keys: ECDH P-256** (CMS KeyAgreeRecipientInfo, RFC 5753) with **AES-256-GCM** content encryption. X25519 recipients are not supported by OpenSSL CMS. A sealing key is distinct from the identity keys by design.
  - **Sealing is hybrid too: ECDH P-256 + ML-KEM-768 (FIPS 203).** A sealed blob is AES-256-GCM under a key HKDF'd from an ML-KEM encapsulation, wrapped in the classical CMS envelope; opening it needs **both** private halves, so breaking one scheme alone reveals nothing. This covers everything encrypted in the project - delivery-envelope headers, message bodies end to end, federation queries - because they all go through the same `cms::seal`.
  - The layering exists only because OpenSSL CMS has no KEMRecipientInfo (RFC 9629) yet - verified against 3.5, whose `cms.h` knows only TRANS/AGREE/KEK/PASS/OTHER. Every primitive is OpenSSL's; the wrapper is versioned so the KEM recipient moves into the CMS itself when support lands, without changing the container or breaking what is already written.
- **Alias syntax**: `a-z` and `0-9` only, case-insensitive (normalized to lowercase), 1-16 characters (`api/FederatedResolve.md`). The length cap keeps an alias distinct from the 52-character fingerprint form.

Implementation: the `common` repository (`Crypto`, `Cms`, `Certificates` modules; covered by tests, including pq-tamper and downgrade cases). Notable sizes: ML-DSA-65 signature ~= 3.3 KB, public key ~= 2 KB - certificates are ~7-8 KB.
