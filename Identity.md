# Identity & Cryptography

## Principles

- OpenSSL only; standardized algorithms and containers: keys in PEM and DER, certificates as one composite CBOR frame, CMS for the password envelope alone. No custom cryptographic constructions (codestyle requirement).
- Every long-lived identity is a **hybrid keypair set** (classical + post-quantum, see Algorithms below); human-readable names are optional, revocable mappings on top.

## User identity

- An account **is** a hybrid keypair set, its **root**. The canonical UID is the root fingerprint covering both public keys. The root signs only what speaks for the whole account: device certificates and revocation lists (below), alias certificates and the requests that change an alias (`api/FederatedResolve.md`), and sign-in with the key (`api/SignInWithKey.md`).
- Address / contact forms:
  - `<fingerprint>:<b33>` - the **descriptor**: root fingerprint + the user's own destination, addressed by the blinded b33 of its encrypted LeaseSet2 (written as a `.b32.i2p` host with a 56-character label, where a plain b32 has 52). `fp:b33` is this specification's shorthand; what a client shows, encodes in a QR and accepts is the link `https://bazari.sh/#invite?v=1&fp=...&dest=...&key=...[&name=...]` (`api/FederatedResolve.md`). A fingerprint is an identity, not a route, so it is always paired with a destination.
  - `<alias>` - an exclusive **alias** on the central resolver (paid; the project's monetization, `api/FederatedResolve.md`). Resolving it yields a descriptor, after which the contact card is fetched and verified.
- Users have no binding to a server in anything a contact can see. Registration opens an unbounded channel with the server, and the only thing that expires is the delegation of the user's destination - which the network itself enforces. Nothing signed by the user names the server.
- **Every device has keys of its own**, certified by the root: a signing pair (Ed25519 + ML-DSA-65) and a sealing pair (X25519 + ML-KEM-768). A device signs everything it sends - messages, the contact card, its requests to the server - with its own signing pair, and never with the root. Its **device id** is the first 10 characters of the fingerprint of its signing pair (`kDeviceIdLength`), computed as the root fingerprint is; it names the device on the server, on the self-channel and in the device lists (`DeviceSync.md`).
- A device is **primary** or **secondary**. A primary device holds the root in its account database; a secondary one does not. The role is chosen when the device is added (`DeviceSync.md`) and is written into its certificate.
- The user's **I2P destination is the account's, not a device's**: contacts hold that one address, so every device of the account operates the same master key. A device that arrives without it takes it from the account (its enrollment material, or `device.i2p-master-request` to the account's other devices) and never mints one, because a second address delegated to the server strands every contact holding the first. A delivery pass follows the same rule for the same reason: it is the account's, not a device's, so it travels in the address book and every device presents the same one (`DeviceSync.md`).

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
  Delivery routing and sealing are per-destination and travel
  in the user's contact card (below); a contact fetches that card from the
  destination the **descriptor** names, with no key and no server card
  (`api/FederatedResolve.md`). Cross-server
  alias resolution is served by the separate central resolver
  (`api/FederatedResolve.md`), which signs its own records.
- **Delivery sealing is per-destination.** Each user's own serving destination
  has
  its **own serving sealing key**; the delivery envelope's admission header
  (recipient, pass, deliveryId) is sealed to the serving sealing key of the
  recipient's destination, whose private half the recipient's server holds
  (`InviteAnonymity.md`).
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
  the server that operates them, which is what lets an account move. The signing key is the only key that signs
  live federation traffic (a delivery confirmation, `api/Federation.md`). The server
  **root identity** signs only **outside** federation - the server card above
  (the `https://bazari.sh/#server` configuration link carries no signature, only the
  fingerprint that card is verified against, `api/Facade.md`) and the
  service-node binding - so two users one server hosts share no signer and cannot be
  clustered by a common server fingerprint: no two users share a destination, so
  none share its signing key (`InviteAnonymity.md`).

## Certificates

**Container for all certificates: one composite CBOR frame over a JSON body** (the body format matches the system-wide JSON wire format). Each body carries a format version field.

### Contact card

Issued **by the user** and signed by one of the account's devices, under a
certificate from the root: the cryptographic answer to "where and how do I
deliver to U?". Signed by U's own keys, so it stays trustworthy through any
untrusted intermediary.

Body: `{ v, t: "contact-card", issuedAt, kind, dest, epochs, servingKey, devices, revocation }`,
the same for every kind. A card is refused unless it carries every one of these
fields, each non-empty, no other field, and `v` 1.

- `t` - what this document is. Checked before any of the body is believed, so
  that one signed document cannot be read as another.
- `issuedAt` - when it was signed. Two cards of one person are ordered by this
  and by nothing else, and the node that holds one **refuses any that is not
  newer** (`api/ClientApi.md`), so a device holding an old copy cannot republish
  it and revert the account's routing.
- `kind` - `user`, `bot` or `chat` (a group or channel host, `Groups.md`): what
  kind of account signed the card. It is fixed when the account is made, every
  card carries it, and a card without it is refused. A client draws a contact by
  it; it is the owner's own statement, so it decides rendering and never trust.
- `epochs` - the account's public **epoch keys** for the current epoch and the
  one after it, `[{n, from, g, by, key}]` (*Epoch keys*, below): what a contact
  encrypts to. The very first contact request is sealed to the current one, and so
  is **every message afterwards**. Each key must have been made by an active
  primary device of the card's own device list. Because they are carried inside
  the signed card, no intermediary can substitute them.
- `dest` - the user's serving **destination** as a `.b32.i2p` host - the blinded
  **b33** of its encrypted LeaseSet2, like every other destination this project
  publishes: where a contact routes deliveries. Never a raw destination or an
  addressbook name (`api/Federation.md`).
- `servingKey` - the public **serving sealing key** of that destination: a contact
  seals the delivery envelope's admission header to it. The user vouches for it
  under their one signature; a wrong value only makes delivery fail (`InviteAnonymity.md`). The account issues it
  and hands its private half to the server that operates the destination, which
  is what lets the server route without being able to read: the message itself is sealed to an epoch key, whose private half never
  leaves the account's devices.
- `devices` - the certificate of every active device of the account, each once
  (*Device certificate*, below).
- `revocation` - the account's newest **revocation list** (below). A card whose
  list is older than the one the node holds is refused (`api/ClientApi.md`).

**The owner is the root, the signer is a device.** A card is a statement its
owner makes about themselves, so the subject is taken from the root that
certified the devices in `devices`, and the signature must verify under one of
those devices that the list does not revoke. A reader who wants a particular
person compares fingerprints, because only the reader knows whose card they
asked for.

A card names **no server** (no server fingerprint or server card), so cards
cannot be grouped by who hosts them, and no presence address: a contact learns
that from a schedule (`Presence.md`). It does show how many devices the account
has and which of them are primary.

**Forward secrecy is by epoch.** Mail is sealed to an epoch key that lasts
30 days, and every device erases the private half of an epoch key 30 days
after its epoch ends; a server deletes every mailbox item 30 days after it
arrived (`api/Storage.md`). So whoever obtains the keys held on a device opens
what was sealed to the epochs that device still holds - at most the current one,
the next one and those that ended within the last 30 days - and nothing older.
`servingKey` is issued anew whenever the account raises a new destination, and
only then. The transcript is stored decrypted inside the account's own database,
so a device and its passphrase yield the history whatever the message keys did;
that database is protected by its passphrase alone (`ThreatModel.md`).

A card is republished to **the server** whenever its contents change. When the
routing it carries changes - a new destination, or a serving key the account
issued anew - contacts are told once, by a `contact.routing` message carrying
the new destination and serving key (`Messages.md`).

**Raising a new destination is one action**, and only a primary device takes
it. A client offers a single
destructive control - *New key* - and it mints a fresh offline key together with
a fresh serving sealing key and signing key, delegates the address with them
(`api/ClientApi.md`), publishes the card, hands the new master and keys to the
account's own other devices (`device.i2p-master`, `DeviceSync.md`), and only then
starts the fan-out. The keys follow the destination: the serving key is issued
anew only with it, and the old destination stops answering inside the same call, which is the intended effect
when the reason for moving is a flood. The invite links already distributed stop
working, and the aliases are repointed automatically.

### Device certificate

Issued **by the root** for one device:
`{ v, t: "device-cert", root, device, signing: {c, pq}, sealing, role, issuedAt }`.

- `root` - the root fingerprint; the certificate must be signed by the root it
  names.
- `device` - the device id, which must be the one its `signing` keys give.
- `signing` - the device's Ed25519 and ML-DSA-65 public keys (SPKI DER, base64).
- `sealing` - the SHA-256 of the device's public sealing key, base64: the
  certificate names the key without giving it, because a contact never seals to a
  device. The account's own devices exchange the keys themselves (`DeviceSync.md`)
  and check each against this digest.
- `role` - `primary` or `secondary`.

Whatever a device signs is checked against its certificate: a message
(`Messages.md`, `auth`), the contact card, a request to the server
(`api/SignInWithKey.md`, section 1).

### Revocation list

Issued **by the root**: `{ v, t: "revocation-list", root, number, revoked, issuedAt }`,
`revoked` the ids of every device the account has revoked, ascending and each once,
and `number` counting from 0. Only a primary device holds the root, so only a
primary device revokes, and it never revokes itself. Whoever holds a list
refuses one with a lower `number`, and a device on the list is refused wherever a
signature of its would be checked: by contacts, by the server and by the
account's other devices. A revoked device is not readmitted; a device that is to
come back is added again as a new device, with new keys.

Revoking a device is one action of a primary device: it signs the next list,
makes new keys for every epoch from the current one to the last it holds, under
the new list's number (*Epoch keys*), hands them only to the devices that stay,
publishes the card, and sends every contact the new device list
(`account.devices`) and the new keys (`account.epochs`, `Messages.md`).

### Epoch keys

Time is cut into **epochs** of 30 days (`kEpochDays`), counted from epoch 0 at
2026-01-01T00:00:00Z. An epoch key is a hybrid sealing pair made at random, never
derived. Its public form `{n, from, g, by, key}` names the epoch `n`, its start
`from`, its **generation** `g` - the number of the revocation list it was made
under - and `by`, the id of the device that made it. One epoch may have several
keys; a sender uses the one of the highest `g`, and of two with the same `g` the
one whose `by` is smaller. The private halves of all of them stay on the
account's devices until 30 days after the epoch ends, so mail sealed to a key
that lost that choice still opens.

- **Only a primary device makes epoch keys.** A card naming a key whose maker is
  not an active primary device of its own list is refused, an `account.epochs`
  letter is taken only when a primary device signed it, and a key whose maker is
  not an active primary device is dropped by a reader that holds the account's
  device list. A secondary device holds the private halves and reads with them;
  were it to make keys, it could hand contacts a key whose private half the
  account's other devices never got.
- A primary device keeps a key for the current epoch and for the next. It makes a
  missing key after a random pause of up to an hour; a key that arrives from
  another primary device in that pause is taken instead.
- The private halves travel to the account's other devices sealed to each
  device's own key (`device.epochs`, `DeviceSync.md`); the public halves travel in
  the card and to every contact in `account.epochs`, which a device sends when
  what it last sent leaves less than one epoch ahead.
- A sender seals each attempt to the key of the epoch current at that attempt
  (`Messages.md`). A sender that holds no key for the current epoch keeps the
  message and fetches the recipient's card, once a day, for the keys.

### Moving to another server

A user is served by **exactly one server at a time**. The address and its keys
belong to the account, not to the server, so a move changes nothing a contact
holds. Moving S1 -> S2:

1. Register on S2.
2. Register there the hash of every pass the account has issued: each is derived
   from the account's delivery secret and the contact's fingerprint
   (`Contacts.md`), so the client computes them again.
3. Delegate the same address to S2 with the same serving and signing keys, and
   publish the same card there.
4. The client stops renewing the delegation on S1, which runs out by the end
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

- The account's root signs the desired nickname: `{ v, t: "alias-certificate", alias, user: <fingerprint>, issuedAt }`; the central resolver registers the mapping alias -> UID (`api/FederatedResolve.md`).
- The certificate carries no term: an alias is rented, and the resolver keeps its term beside the claim.

### First-contact trust chain (alias resolution)

Resolving an `alias` returns a **descriptor** (`fingerprint`, serving destination)
signed by the central resolver (a delegation chain to a
hardcoded root fingerprint, `api/FederatedResolve.md`). The client then fetches and
verifies the user-signed **contact card** for that fingerprint. Two trust layers:

1. **alias -> fingerprint** is asserted by the resolver - the inherent trust of
   names; the UI surfaces the resolved fingerprint for out-of-band verification.
2. everything after - the contact card (epoch keys + devices + destination +
   serving key) - is **signed by a device the root certified and verified against
   the fingerprint**, so no intermediary can forge it.

The lookup travels over I2P, as a direct fetch from a transient I2P destination
of the client's own, so the requester's own server never learns which name was
looked up (`api/FederatedResolve.md`).

### Out-of-band invites (the descriptor)

A contact is shared out of band as a small **descriptor** - the link
`https://bazari.sh/#invite?v=1&fp=...&dest=...&key=...[&name=...]` (`fp:b33`): identity +
serving destination, plus an optional advertised
display name the recipient adopts at add time (static thereafter; see `Messages.md`).
It fits a single QR. The recipient
fetches the user's **contact card** for that fingerprint from the destination
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

- **Identity is a post-quantum hybrid: Ed25519 + ML-DSA-65 (FIPS 204).** The root and every device key are such a pair. Every identity-level statement (contact card, device certificate, revocation list, alias certificate, server card) carries **both** signatures and is valid only when both verify - forging requires breaking both schemes. This is mandatory. The one exception is a presence watch, a request checked once on arrival and dropped after 120 seconds, which carries the Ed25519 signature of the watcher's device alone under its own domain string (`Presence.md`).
- **One composite frame.** A statement is one CBOR frame built from standard primitives: `{v, body, c: {alg, pub, sig}, q: {alg, pub, sig}}`, with `body` the exact signed bytes. Both halves sign the **same** message: the domain-separation string, then SHA-256 over `Ed25519 SPKI DER || ML-DSA SPKI DER`, then the body bytes. Naming both keys in the signed message is what stops either half being copied into a frame that names a different key. Verification checks both signatures **and both key types** (a classical key in the pq slot is a downgrade and fails).
- **Identity fingerprint**: lowercase unpadded base32 (RFC 4648) of SHA-256 over the concatenation `Ed25519 SPKI DER || ML-DSA SPKI DER` - 52 characters, covering both keys; substituting either key changes the identity. A device id is the first 10 characters of the same fingerprint of the device's signing pair. Display shortening is a client UX concern.
- **Sealing keys: X25519 + ML-KEM-768 (FIPS 203), one composite KEM.** A sealing key is distinct from the identity keys by design, and it is a pair: its public form is the CBOR `{t, c, q}` carrying both public halves (1268 bytes, measured).
  - A sealed blob is **one** AES-256-GCM layer under a key derived with HKDF-SHA256 from both shared secrets at once - `ikm = ss_ML-KEM || ss_X25519`, salt = the frame's transcript (KEM ciphertext, ephemeral X25519 public key, recipient public pair, SHA-256 of `kid`), info = the seal's domain-separation string - carried as `{v, kid, kem, eph, n, ct}`. Opening needs **both** private halves, so breaking one scheme alone reveals nothing, and no public field of the frame can be moved without changing the derived key. This covers everything encrypted in the project - delivery-envelope headers, message bodies end to end, federation queries - because they all go through the same `hybrid::seal`.
  - `kid` names the recipient key, at most 19 bytes: for an epoch key `0x01`, `n` and `g` as 4-byte big-endian numbers, and the maker's device id (19 bytes); for a device's own key `0x02` and the device id (11 bytes). A reader picks the private key by it, and a frame naming a key the reader no longer holds does not open. Elsewhere it is empty. Overhead over the plaintext is at most 1200 bytes (`kHybridSealOverheadBytes`, measured with the longest `kid` on a body of 64 KiB or more; a few bytes less on a smaller one).
  - **CMS is used for the password envelope only** (RFC 3211 PWRI, AES-256-CBC): the account backup and the file-transfer payload are keyed by a password, involve no asymmetric key, and stream, so the standard container is used there.
- **Alias syntax**: `a-z` and `0-9` only, case-insensitive (normalized to lowercase), 1-16 characters (`api/FederatedResolve.md`). The length cap keeps an alias distinct from the 52-character fingerprint form.

Implementation: the `common` repository (`Crypto`, `Hybrid`, `Certificates`, `Devices` and `Epochs` modules; `Cms` is the password envelope alone). Tests cover tamper, downgrade and half-splicing cases. Notable sizes, measured: ML-DSA-65 signature 3309 bytes and public key 1974 bytes against Ed25519's 64 and 44, so an alias certificate is 5593 bytes. A contact card carries two epoch keys and a certificate for every active device, so its size grows with the number of devices.
