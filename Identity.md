# Identity & Cryptography

Status: draft skeleton.

## Principles

- OpenSSL only; standardized algorithms and containers (PEM, DER, X.509, CMS). No custom cryptographic constructions (codestyle requirement).
- Every long-lived identity is a **hybrid keypair set** (classical + post-quantum, see Algorithms below); human-readable names are optional, revocable mappings on top.

## User identity

- A user **is** a hybrid keypair set. The canonical UID is the identity fingerprint covering both public keys.
- Address / contact forms:
  - `<fingerprint>:<b33>` - the **descriptor**: identity fingerprint + the user's serving destination (a `.b32.i2p` host). The primary scannable/QR form; the full link `bazarish://invite?fp=...&srv=...&srv_key=...` also carries the server's serving sealing key (`api/FederatedResolve.md`). A fingerprint is an identity, not a route, so it is always paired with a destination - there is no `@server-fingerprint` form (nothing maps a fingerprint to an endpoint).
  - `<alias>` - an exclusive **username** on the central resolver (paid; the project's monetization, `api/AliasResolver.md`). Resolving it yields a descriptor, after which the contact card is fetched and verified.
- Users have no strict binding to a server; the binding is expressed by a time-limited subscription certificate.
- All clients (devices/app instances) of a user **share the identity keypair set**; a new device receives it during enrollment (see `DeviceSync.md`). Each client registers a unique random **client ID** so the server can track delivery completeness across clients.

## Server identity

- A server's root identity is a hybrid keypair set.
- **Server card** - a document signed by the server's root key, binding the
  server's identity to its transport endpoint(s) (`i2p:<b33>`) and a server-level
  sealing key. It is the server's own identity document (the service node is
  bound to it). It is **not** the peer-facing routing artifact and is **not**
  carried in invites: delivery routing and sealing are per-destination and travel
  in the user's contact card (below); a contact discovers the serving sealing key
  needed to fetch that card from the **descriptor** (`srv_key`,
  `api/FederatedResolve.md`), not from a fetched server card. Cross-server
  username resolution is served by the separate central resolver
  (`api/AliasResolver.md`), which signs its own records.
- **Delivery sealing is per-destination.** Each serving destination (the home
  destination, each address-pool destination, a paid per-user destination) has
  its **own serving sealing key**; the delivery envelope's admission header
  (recipient, token, messageId) is sealed to the serving sealing key of the
  recipient's destination, whose private half the recipient's server holds. There
  is no single server-wide delivery sealing key (`InviteAnonymity.md`).
- **Federation signing is per-destination too.** Each serving destination also
  has its **own signing keypair**, stored bound to that destination together with
  its serving sealing key - a `(destination, signing key, serving sealing key)`
  group that is never reused on another destination. It is the only key that signs
  live federation traffic (the `delivered-ack`, `api/Federation.md`). The server
  **root identity** (the server card above) signs only **outside** federation -
  the `bazarish://server` configuration link (`api/Facade.md`) and the
  service-node binding - so two users one server hosts on different destinations
  share no signer and cannot be clustered by a common server fingerprint; users
  sharing a destination share its signing key, which is the same k-anonymity set
  that already shares the destination (`InviteAnonymity.md`).

## Certificates

**Container for all certificates: CMS SignedData over a JSON body** (OpenSSL-native, standard; the body format matches the system-wide JSON wire format). Each body carries a format version field.

### Subscription certificate

Issued **by the user**, signed with the identity key; asserts "server S serves user U until T" **and** carries the user's peer-facing routing - it doubles as the **contact card**.

Body (draft): `{ v, user: <fingerprint>, server: <server-fingerprint>, issuedAt, notAfter, sealingKey?, dest?, servingSealingKey? }`.

- `sealingKey` - the user's E2E sealing **prekey** (SPKI DER, base64): a contact encrypts the very first contact request to it before any token exchange. Because it rides inside the user-signed certificate, no intermediary can substitute it, closing the first-message bootstrap gap.
- `dest` - the user's serving **destination** as a `.b32.i2p` host (its b33; its standard b32 for a paid offline-key destination, which cannot publish a b33): where a contact routes deliveries. Never a raw destination or an addressbook name (`api/Federation.md`).
- `servingSealingKey` - the public **serving sealing key** of that destination: a contact seals the delivery envelope's admission header to it. Not a signature and not a trust anchor - the user just vouches for it under their one signature; a wrong value only makes delivery fail (`InviteAnonymity.md`). Its private half is held by the user's server. The client learns `dest`/`servingSealingKey` from its server (`GET /v1/messaging/destination`) and folds them into the certificate.

There is **no server fingerprint dependency or server card** in the peer-facing artifact - routing is by destination.

It has **two consumers**:

1. **The serving server (via the service node)** - the lifecycle anchor: "subscription active" <=> "valid certificate on file". Renewal = a fresh certificate; the server publishes its maximum term and rejects longer ones. After expiry the server may purge all pending content for the user (grace period is server config). Registration is grant-gated through the service portal (`api/ServicePortal.md`).
2. **Contacts' clients** - the **contact card**: the same artifact, handed to contacts over E2E (or in an invite), is the cryptographic answer to "where and how do I deliver to U?" (prekey + destination + serving sealing key). Signed by U, so it stays trustworthy through any untrusted intermediary.

Routing-staleness policy for contacts: the last known card remains the routing target even past `notAfter` - expiry signals staleness, not invalidity. A fresh card is pushed to **the server on every renewal**, but to **contacts only when the destination changes** (avoids re-broadcast every few days).

### One serving server, hard-cutover migration

A user holds a subscription with **exactly one server at a time**. Migration S1 -> S2:

1. Subscribe to S2, register fresh token hashes there, unsubscribe from S1 (or let the certificate lapse).
2. Push the new contact card (S2 destination + serving sealing key + prekey) + fresh token batches to all contacts over E2E.
3. Contacts switch as the card arrives. Deliveries that race the cutover **fail and stay on the sender's client** (standard send semantics) and are retried after the new card arrives - messages are delayed, not lost.

Failure mode accepted: if S1 dies before the user prepared the migration, contacts' retries keep failing until the user pushes a new card (or, with an alias, contacts re-resolve it). A paid per-user destination is portable across servers (it is the user's own offline key), so migration need not change the destination at all.

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
2. everything after - the contact card (prekey + destination + serving sealing
   key) - is **user-signed and verified against the fingerprint**, so no
   intermediary can forge it.

The lookup travels over I2P (a direct fetch over a transient I2P destination, or
sealed via the client's own server), so the requester's own server never learns which name was looked up
(`api/FederatedResolve.md`).

### Out-of-band invites (the descriptor)

A contact is shared out of band as a small **descriptor** - `<fingerprint>:<b33>`,
or the link `bazarish://invite?fp=...&srv=...&srv_key=...`: identity + serving
destination + the server's serving sealing key. It fits a single QR. The recipient
fetches the user-signed **contact card** for that fingerprint from the destination
and verifies it against the fingerprint, so the binding is cryptographic and
intermediaries are trusted for availability only. This replaces the earlier
full-card invite (the whole ~= 7-8 KB user-signed card inline, spanning several QR
symbols): smaller and always-fresh, at the cost of a live fetch at add-time
(`api/FederatedResolve.md`, `api/AliasResolver.md`).

Trust ladder for adding a contact, strongest first:

- **By descriptor / fingerprint** (`fp:b33`, a QR or link) - the contact card is
  fetched and verified against the fingerprint, so a server cannot forge, only
  withhold; on a direct fetch over a transient I2P destination it cannot even observe the lookup.
- **By username (alias)** - the central resolver maps the human-readable name to a
  fingerprint. Convenient, but the **alias->fingerprint mapping is trusted**: a
  hostile resolver could return an attacker's fingerprint, so the UI surfaces the
  resolved fingerprint for out-of-band verification. Everything after the mapping
  is verified as above.

## Algorithms (decided empirically against OpenSSL 3.5)

- **Identity is a post-quantum hybrid: ECDSA P-256 + ML-DSA-65 (FIPS 204).** Every identity-level statement (subscription certificate, alias certificate, server card) carries **both** signatures and is valid only when both verify - forging requires breaking both schemes. This is mandatory, not optional.
- **Hybrid layering.** OpenSSL CMS cannot carry ML-DSA (or EdDSA) signers, so the hybrid is layered from standard primitives with no custom constructions: the exact body bytes are signed with ML-DSA-65 at the EVP level; the wrapper `{body: base64(bodyBytes), pq: {alg, pub, sig}}` is then CMS-SignedData-signed with the classical key. Verification checks both layers **and both key types** (a classical key smuggled into the pq slot is a downgrade and fails). Replace with CMS composite signatures when OpenSSL supports the LAMPS composite drafts.
- **Identity fingerprint**: lowercase unpadded base32 (RFC 4648) of SHA-256 over the concatenation `classical SPKI DER || ML-DSA SPKI DER` - 52 characters, covering both keys; substituting either key changes the identity. Display shortening is a client UX concern.
- **Sealing keys: ECDH P-256** (CMS KeyAgreeRecipientInfo, RFC 5753) with **AES-256-GCM** content encryption. X25519 recipients are not supported by OpenSSL CMS. A sealing key is distinct from the identity keys by design.
  - **Accepted PQC gap**: sealing is not yet hybrid - OpenSSL CMS has no ML-KEM/KEMRecipientInfo (RFC 9629) support. Sealed envelopes protect routing metadata (recipient, token), so harvest-now-decrypt-later applies to *metadata*, not message content (which is E2E on top). Revisit as soon as OpenSSL ships KEMRI/ML-KEM in CMS; threat-model item.
- **Alias syntax**: `a-z` and `0-9` only, case-insensitive (normalized to lowercase), 1-32 characters (`api/AliasResolver.md`). The length cap keeps an alias distinct from the 52-character fingerprint form.

Implementation: the `common` repository (`Crypto`, `Cms`, `Certificates` modules; covered by tests, including pq-tamper and downgrade cases). Notable sizes: ML-DSA-65 signature ~= 3.3 KB, public key ~= 2 KB - certificates are ~7-8 KB.
