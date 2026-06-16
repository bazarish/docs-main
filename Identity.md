# Identity & Cryptography

Status: draft skeleton.

## Principles

- OpenSSL only; standardized algorithms and containers (PEM, DER, X.509, CMS). No custom cryptographic constructions (codestyle requirement).
- Every long-lived identity is a **hybrid keypair set** (classical + post-quantum, see Algorithms below); human-readable names are optional, revocable mappings on top.

## User identity

- A user **is** a hybrid keypair set. The canonical UID is the identity fingerprint covering both public keys.
- Address forms:
  - `<fingerprint>@<server-fingerprint>` — fully qualified.
  - `<alias>@<server-fingerprint>` — alias registered at that server's service node.
  - `<alias>` — exclusive alias on the **main server**, which is substituted when the `@server` part is absent (paid exclusive usernames — the project's monetization).
- Users have no strict binding to a server; the binding is expressed by a time-limited subscription certificate.
- All clients (devices/app instances) of a user **share the identity keypair set**; a new device receives it during enrollment (see `DeviceSync.md`). Each client registers a unique random **client ID** so the server can track delivery completeness across clients.

## Server identity

- A server's root identity is a hybrid keypair set; its fingerprint is the `@server` part of addresses.
- **Server card** — a document signed by the server's root key, binding the
  server's identity to its transport endpoint(s) (`i2p:<b33>`) and a server-level
  sealing key. It is the server's own identity document (the service node is
  bound to it) and the entry point for sealed cross-server **alias resolution**
  (`api/Federation.md`). It is **not** the peer-facing routing artifact any more
  and is **not** carried in invites: delivery routing and sealing are
  per-destination and travel in the user's contact card (below).
- **Delivery sealing is per-destination.** Each serving destination (the home
  destination, each address-pool destination, a paid per-user destination) has
  its **own serving sealing key**; the delivery envelope's admission header
  (recipient, token, messageId) is sealed to the serving sealing key of the
  recipient's destination, whose private half the recipient's server holds. There
  is no single server-wide delivery sealing key (`InviteAnonymity.md`).

## Certificates

**Container for all certificates: CMS SignedData over a JSON body** (OpenSSL-native, standard; the body format matches the system-wide JSON wire format). Each body carries a format version field.

### Subscription certificate

Issued **by the user**, signed with the identity key; asserts "server S serves user U until T" **and** carries the user's peer-facing routing — it doubles as the **contact card**.

Body (draft): `{ v, user: <fingerprint>, server: <server-fingerprint>, issuedAt, notAfter, sealingKey?, dest?, servingSealingKey? }`.

- `sealingKey` — the user's E2E sealing **prekey** (SPKI DER, base64): a contact encrypts the very first contact request to it before any token exchange. Because it rides inside the user-signed certificate, no intermediary can substitute it, closing the first-message bootstrap gap.
- `dest` — the user's serving **destination** (its b33; the raw destination for a paid offline-key destination): where a contact routes deliveries.
- `servingSealingKey` — the public **serving sealing key** of that destination: a contact seals the delivery envelope's admission header to it. Not a signature and not a trust anchor — the user just vouches for it under their one signature; a wrong value only makes delivery fail (`InviteAnonymity.md`). Its private half is held by the user's server. The client learns `dest`/`servingSealingKey` from its server (`GET /v1/messaging/destination`) and folds them into the certificate.

There is **no server fingerprint dependency or server card** in the peer-facing artifact — routing is by destination.

It has **two consumers**:

1. **The serving server (via the service node)** — the lifecycle anchor: "subscription active" ⇔ "valid certificate on file". Renewal = a fresh certificate; the server publishes its maximum term and rejects longer ones. After expiry the server may purge all pending content for the user (grace period is server config). Registration is grant-gated through the service portal (`api/ServicePortal.md`).
2. **Contacts' clients** — the **contact card**: the same artifact, handed to contacts over E2E (or in an invite), is the cryptographic answer to "where and how do I deliver to U?" (prekey + destination + serving sealing key). Signed by U, so it stays trustworthy through any untrusted intermediary.

Routing-staleness policy for contacts: the last known card remains the routing target even past `notAfter` — expiry signals staleness, not invalidity. A fresh card is pushed to **the server on every renewal**, but to **contacts only when the destination changes** (avoids re-broadcast every few days).

### One serving server, hard-cutover migration

A user holds a subscription with **exactly one server at a time**. Migration S1 → S2:

1. Subscribe to S2, register fresh token hashes there, unsubscribe from S1 (or let the certificate lapse).
2. Push the new contact card (S2 destination + serving sealing key + prekey) + fresh token batches to all contacts over E2E.
3. Contacts switch as the card arrives. Deliveries that race the cutover **fail and stay on the sender's client** (standard send semantics) and are retried after the new card arrives — messages are delayed, not lost.

Failure mode accepted: if S1 dies before the user prepared the migration, contacts' retries keep failing until the user pushes a new card (or, with an alias, contacts re-resolve it). A paid per-user destination is portable across servers (it is the user's own offline key), so migration need not change the destination at all.

### Alias certificate

- The user signs the desired nickname: `{ v, alias, user: <fingerprint>, issuedAt, notAfter? }`; the service node registers the mapping alias → UID.
- Validity term is set by the server; default unlimited.

### First-contact trust chain (alias resolution)

Resolving `alias@server` (or a main-server alias) returns two artifacts, verified end-to-end by the requesting client with no trust in intermediaries:

1. **Alias certificate** — signed by the user: "I am alias X";
2. **Contact card** (the user-signed subscription certificate) — prekey + destination + serving sealing key.

The resolve query travels sealed (see `api/Federation.md`), so the requester's own server never learns which alias was looked up.

### Out-of-band invites

The contact card can be packaged for sharing over any channel (a link, a QR
code, a paper printout) instead of being fetched from a server. An **invite**
carries a **single** self-verifying artifact:

- **Contact card** (the user-signed subscription certificate) — prekey +
  destination + serving sealing key, all under the user's one signature.

The recipient verifies that one signature and binds the card to the user's
fingerprint locally, so an invite needs **no trust in any server** — not even
for availability, since nothing is fetched, and no server-identifying fields are
shared (`InviteAnonymity.md`). This is the strongest contact-add path; its only
cost is size (the hybrid post-quantum signature makes the card ≈ 7–8 KB), which a
single QR code cannot hold — an invite spans a short sequence of QR symbols
(structured append) or rides in a link.

Trust ladder for adding a contact, strongest first:

- **By invite** — fully offline, zero server trust, no metadata leak. Largest to share.
- **By fingerprint** — the 52-character UID; certificates are fetched and
  verified against it, so a server cannot forge, only withhold or observe the lookup.
- **By username (alias)** — a resolver maps the human-readable name to a
  fingerprint. Convenient for verbal exchange, but the **alias→fingerprint
  mapping is trusted**: a hostile resolver could return an attacker's
  fingerprint. Everything after the mapping is verified as above.

## Algorithms (decided empirically against OpenSSL 3.5)

- **Identity is a post-quantum hybrid: ECDSA P-256 + ML-DSA-65 (FIPS 204).** Every identity-level statement (subscription certificate, alias certificate, server card) carries **both** signatures and is valid only when both verify — forging requires breaking both schemes. This is mandatory, not optional.
- **Hybrid layering.** OpenSSL CMS cannot carry ML-DSA (or EdDSA) signers, so the hybrid is layered from standard primitives with no custom constructions: the exact body bytes are signed with ML-DSA-65 at the EVP level; the wrapper `{body: base64(bodyBytes), pq: {alg, pub, sig}}` is then CMS-SignedData-signed with the classical key. Verification checks both layers **and both key types** (a classical key smuggled into the pq slot is a downgrade and fails). Replace with CMS composite signatures when OpenSSL supports the LAMPS composite drafts.
- **Identity fingerprint**: lowercase unpadded base32 (RFC 4648) of SHA-256 over the concatenation `classical SPKI DER || ML-DSA SPKI DER` — 52 characters, covering both keys; substituting either key changes the identity. Display shortening is a client UX concern.
- **Sealing keys: ECDH P-256** (CMS KeyAgreeRecipientInfo, RFC 5753) with **AES-256-GCM** content encryption. X25519 recipients are not supported by OpenSSL CMS. A sealing key is distinct from the identity keys by design.
  - **Accepted PQC gap**: sealing is not yet hybrid — OpenSSL CMS has no ML-KEM/KEMRecipientInfo (RFC 9629) support. Sealed envelopes protect routing metadata (recipient, token), so harvest-now-decrypt-later applies to *metadata*, not message content (which is E2E on top). Revisit as soon as OpenSSL ships KEMRI/ML-KEM in CMS; threat-model item.
- **Alias syntax**: `[a-z0-9_.-]`, 1–32 characters, must not start or end with a separator. The length cap guarantees aliases never collide with the 52-character fingerprint form.

Implementation: the `common` repository (`Crypto`, `Cms`, `Certificates` modules; covered by tests, including pq-tamper and downgrade cases). Notable sizes: ML-DSA-65 signature ≈ 3.3 KB, public key ≈ 2 KB — certificates are ~7–8 KB.
