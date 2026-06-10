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
- **Server card** — a document signed by the server's root key, containing:
  - the server's transport endpoints, as a list of typed entries (`i2p:<destination>` first; extensible to future transports);
  - the server's **sealing public key** — used by sending clients to encrypt the sealed part of delivery envelopes (recipient, token, messageId) so that only the destination server can read it (see `api/Federation.md`);
  - a version/issued-at field so newer cards supersede older ones;
  - (TBD) facade bootstrap info given to a user at registration: secret URI list.
- Distribution: handed to the user at registration, exchanged between servers during federation, and embeddable in contact-exchange payloads.

## Certificates

**Container for all certificates: CMS SignedData over a JSON body** (OpenSSL-native, standard; the body format matches the system-wide JSON wire format). Each body carries a format version field.

### Subscription certificate

Issued **by the user**, signed with the identity key; asserts "server S serves user U until T".

Body (draft): `{ v, user: <fingerprint>, server: <server-fingerprint>, issuedAt, notAfter, sealingKey? }`.

The optional `sealingKey` is the user's sealing public key (SPKI DER, base64) — a **prekey**. Because it rides inside the user-signed certificate, a contact can fetch it (`GET /v1/account/contact` or alias resolution) and E2E-encrypt the very first contact request before any token exchange, with no risk of an intermediary substituting the key. This closes the first-message bootstrap gap: every payload, including the contact request, is end-to-end encrypted.

It has **two consumers**:

1. **The serving server (via the service node)** — the lifecycle anchor: "subscription active" ⇔ "valid certificate on file". Renewal = a fresh certificate; the server publishes its maximum term and rejects longer ones. After expiry the server may purge all pending content for the user (grace period is server config).
2. **Contacts' clients** — the **serving statement**: the same artifact, pushed to contacts over E2E, is the cryptographic answer to "where do I deliver to U?". It is signed by U, so it stays trustworthy through any untrusted intermediary.

Routing-staleness policy for contacts: the last known statement remains the routing target even past `notAfter` — expiry signals staleness, not invalidity. A fresh statement is pushed to **the server on every renewal**, but to **contacts only when the serving server changes** (avoids re-broadcast every few days).

### One serving server, hard-cutover migration

A user holds a subscription with **exactly one server at a time**. Migration S1 → S2:

1. Subscribe to S2, register fresh token hashes there, unsubscribe from S1 (or let the certificate lapse).
2. Push the new serving statement + S2's server card + fresh token batches to all contacts over E2E.
3. Contacts switch as the statement arrives. Deliveries that race the cutover **fail and stay on the sender's client** (standard send semantics) and are retried after the new statement arrives — messages are delayed, not lost.

Failure mode accepted: if S1 dies before the user prepared the migration, contacts' retries keep failing until the user pushes a new statement (or, with an alias, contacts re-resolve it).

### Alias certificate

- The user signs the desired nickname: `{ v, alias, user: <fingerprint>, issuedAt, notAfter? }`; the service node registers the mapping alias → UID.
- Validity term is set by the server; default unlimited.

### First-contact trust chain (alias resolution)

Resolving `alias@server` (or a main-server alias) returns three artifacts, verified end-to-end by the requesting client with no trust in intermediaries:

1. **Alias certificate** — signed by the user: "I am alias X";
2. **Subscription certificate** — signed by the user: "server S serves me";
3. **Server card** — signed by S's root key: transport endpoints and sealing key.

The resolve query travels sealed (see `api/Federation.md`), so the requester's own server never learns which alias was looked up.

### Out-of-band invites

The same trust chain can be packaged for sharing over any channel (a link, a
QR code, a paper printout) instead of being fetched from a server. An **invite**
carries two self-verifying artifacts:

1. **Subscription certificate** — signed by the user: serving server + sealing prekey;
2. **Server card** — signed by the server's root key: transport endpoints + sealing key.

The recipient verifies both signatures and binds them to the user's
fingerprint locally, so an invite needs **no trust in any server** — not even
for availability, since nothing is fetched. This is the strongest contact-add
path; its only cost is size (both certificates are hybrid post-quantum signed,
≈ 7–8 KB each), which a single QR code cannot hold — an invite spans a short
sequence of QR symbols (structured append) or rides in a link.

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
