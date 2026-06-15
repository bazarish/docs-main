# Large messages & blob storage

Status: design (2026-06-16).

Any message whose serialized blob exceeds **512 KB** (text, attachments —
content-agnostic) is not delivered through the mailbox. Instead the **client**
encrypts the whole message blob, uploads the ciphertext to a **blob storage**
service, and sends only a small *pointer* through normal delivery. The recipient
pulls the ciphertext out-of-band (directly over I2P when it can, or via its own
server as a proxy), verifies it, decrypts it, and confirms receipt so the blob
can be reclaimed.

This keeps the messaging server (`server-core`) a pure small-message delivery
layer: it never sees a large blob, and a quota-less recipient can receive
arbitrarily large content without it ever touching their mailbox.

```
 sender Client                     blob-storage                 recipient Client
 ──────────────                    ────────────                 ────────────────
 blob > 512 KB                                                  
   │ encrypt (CMS/AES-GCM, per-blob key)
   │ PUT ciphertext + retention  ─────────►  store, assign dest+id
   │ ◄───────────────────────────────────   return i2p URL + delete-token
   │ build pointer {url,id,key,sha256,size,meta}
   │ seal pointer to recipient
   ▼
 server-core delivery (small pointer) ──────────────────────►  mailbox
                                                                   │ read pointer
                                       GET (authless capability)   │ direct SAM, or
                            ◄──────────────────────────────────────┤ proxy via own server
                                       ciphertext (range/resume)    │
                                                                   │ verify sha256, decrypt
                            ◄─── POST confirm {id} (no self-id) ────┤
                            decrement / delete
```

## Threshold and packing (client-side)

- Trigger is a single rule on the **serialized message blob size > 512 KB**,
  regardless of what the message contains. Below the threshold, delivery is
  unchanged (inline through the mailbox).
- Packing happens **entirely on the client**:
  1. Serialize the message; if `> 512 KB`, generate a per-blob symmetric key and
     encrypt the whole blob (CMS `AuthEnvelopedData` / AES-GCM — standard
     primitives only, no custom containers).
  2. `PUT` the ciphertext to a blob storage and receive its download URL.
  3. Replace the message with a **pointer** carrying the metadata needed for a
     free (authless) download, seal the pointer to the recipient, and send it
     through normal delivery.
- `server-core` is untouched: it only ever sees the small sealed pointer, so the
  recipient's mailbox quota is never a factor for large content.

### Pointer contents (sealed E2E to the recipient)

```
{
  "blobUrl":  "<i2p download URL — authless capability>",
  "blobId":   "<high-entropy capability id>",
  "fileKey":  "<per-blob symmetric key>",
  "sha256":   "<ciphertext or plaintext digest for integrity>",
  "size":     <bytes>,
  "meta":     { filename, mime, ... }
}
```

The `fileKey` travels **only** inside the sealed pointer. Blob storage holds
ciphertext only and can never read the content. `blobId` is a bearer capability:
unguessable, so an authless `GET` is acceptable because the payload is already
ciphertext.

## Blob storage (separate service, repo `bazarish/blob-storage`)

A deliberately simple, standalone role with no messaging logic. It can run
bundled with a messaging server (default deployment) or on a separate cheap box,
including a third-party operator. Three faces:

1. **Upload — through the facade, authenticated, quota-gated.** The client `PUT`s
   ciphertext plus a **retention policy**. Storage assigns a `blobId` and an I2P
   download dest, records `owner` (for quota accounting and `DELETE`), and returns
   the download URL plus a **delete-token**. The client keeps the delete-token
   with the message metadata, so the sender can later **manually unsend** the
   message and free its own storage. Per-user quota is controlled by the
   **service node** (the same storage-grant machinery as messaging; the service
   node pushes the effective quota the same way it does to `server-core`).
2. **Download — direct over I2P, authless capability.** `GET` by `blobId` returns
   the ciphertext, with HTTP range support for resume. Anyone holding the link can
   fetch, but it is ciphertext and the key is only in the sealed pointer.
3. **Confirm-receipt — authless, no self-identification.** After a successful,
   hash-validated download the recipient `POST`s `confirm {blobId}` — carrying the
   `blobId` only, never who. This is the signal that the blob was fully and
   correctly received and drives count-based reclamation.

The owner is known (upload is authenticated, needed for quota and `DELETE`), but
**download and confirm are anonymous**.

### Retention: TTL backstop, plus an optional sender-set download-count

The guiding principle is **store for the minimum time**: as soon as every
intended client has received the blob, delete it; never keep ciphertext longer
than necessary.

By default a blob carries only a **TTL of 7 days** (the backstop). The **sender**
may additionally set an explicit **download-count** to reclaim sooner — the moment
all recipients have it:

- `count = 1` — delete after the first confirmed download. Ephemeral 1:1 delivery
  with a near-zero standing quota footprint.
- `count = N` — a group of N recipients; reclaimed after N confirmations.
- There is **no default count** — only the sender may set it, since only the
  sender knows the intended audience. Without it, reclamation is by TTL alone.
- **TTL is always the backstop**, even for count-based blobs: if the expected
  confirmations never arrive, the blob is swept by time.

Defaults (configurable): `ttl = 7 days`; no default count.

Reclamation counts **confirmations, not `GET`s** — a partial or failed download
never burns a slot. Abuse note: only a party holding the `blobId` (a participant
in the conversation) can confirm; a malicious group member can burn at most one
of N slots, and TTL plus owner `DELETE` (unsend, via the delete-token) remain as
backstops.

Quota is a live high-watermark of currently-stored ciphertext: charged on `PUT`,
released on reclamation (`count` reached, TTL, or `DELETE`). With `count = 1` a
large transfer frees almost immediately after delivery.

### I2P destination rotation

Blob storage does **not** serve from a single stable address. It maintains a pool
of public I2P destinations (default **10**), all forwarding to the same backing
web server, and rotates them continuously so the set of public addresses keeps
drifting — a slow, low-load churn that resists long-term surveillance and
blocklisting and unlinks blobs that happened to share a dest.

State per dest: a count of **active links** (blobs whose download URL is pinned to
that dest).

- **One dest is "frozen" at a time.** A frozen dest stops being handed new links;
  the other 9 take all new links **round-robin without a per-dest limit**.
- A dest becomes the freeze candidate when it **first crosses the link limit**
  (default **10** active links). If several are already over the limit when a
  freeze slot opens, the one with the **highest** active-link count is chosen.
- When the frozen dest's active links drain to **0** (all its blobs reclaimed), it
  is **destroyed** (its I2P keypair discarded) and a **fresh** dest is created in
  its place, keeping the pool at 10. The freeze slot then moves to the next
  candidate and the cycle repeats.
- At low volume no dest may have crossed the limit; then nothing is frozen and all
  10 are in plain round-robin until one crosses it.

**Invariant: a dest is destroyed only at zero active links, so rotation never
invalidates a live download URL.** A blob's URL stays valid for the blob's whole
lifetime; the dest pinned at upload outlives it.

Because constructing a SAM `STREAM` session blocks on tunnel build (tens of
seconds), the replacement dest is **warmed in the background** before the drained
dest is torn down, so the pool stays ready without a noticeable spike. Each pool
dest is one `SamSession` (`common`’s SAM wrapper) running an accept loop that
forwards to the single local web server; create/destroy a dest = construct/destroy
its session.

Defaults (configurable): `poolSize = 10`, `linkLimitPerDest = 10`.

### Encrypted leasesets & the `.b32.i2p` address

Every pooled destination publishes an **encrypted LeaseSet2** (`i2cp.leaseSetType=5`,
no per-client auth), never a standard/cleartext leaseset. A standard leaseset is
stored in the netdb at floodfills in the clear, so a floodfill — or anyone passively
monitoring floodfills — can enumerate the destination and its current tunnel
endpoints. An encrypted LS2 stores the leases encrypted under a key derived from the
blinded address, so floodfills hold opaque data and cannot be queried without the
blinded key. This is a **project-wide invariant**: all Bazarish I2P destinations
(federation, per-user, pool, blob storage) use encrypted leasesets.

The shareable address of an encrypted leaseset is the **blinded b32** (its "b33"),
derived from the destination's signing key — *not* `base32(sha256(destination))`
(the standard-LS form). For an Ed25519 destination (sig type 7, blinded sig type 11
= RedDSA-Ed25519), with signing public key `A` (32 bytes):

```
addr[0]      = flags            // 0 (0x04 would mark per-client auth — not used)
addr[1]      = sigType          // 1 byte  (7)
addr[2]      = blindedSigType   // 1 byte  (11)
addr[3..34]  = A                // 32-byte signing public key
checksum     = crc32(addr[3..34])          // zlib CRC-32, little-endian
addr[0] ^= checksum;  addr[1] ^= checksum>>8;  addr[2] ^= checksum>>16
b33  = base32_rfc4648_lower_nopad(addr[0..34])   // 56 chars
host = b33 + ".b32.i2p"
```

(Per I2P proposals 123/149; matches i2pd's `BlindedPublicKey::ToB33`.) SAM
`NAMING LOOKUP ME` returns the raw destination, not the blinded address, so the
store derives the b33 itself from the session's signing key — reusing the vendored
trimmed **libi2pd** (`IdentityEx` + `Blinding`), the same library the client uses
for key generation. Recipients reach it with SAM `STREAM CONNECT <host>`; the router
performs the blinded lookup and decrypts the leaseset.

**Strict rule — blob storage emits `.b32.i2p` download links only, with no
alternative.** There is no clearnet, LAN, or direct-host fallback for a download URL.
The sole way to fetch a blob is over I2P, so a client (or a careless tool) that
fetches a link directly can never leak its IP. The store has no non-I2P serving
mode.

## Fetch: direct (preferred) or proxied (fallback)

The download is a generic authless I2P `GET` — **federation is not involved**; the
recipient does not need to know "whose" server hosts the blob, only the URL.

- **Direct fetch (a SAM bridge is reachable).** The recipient's client opens its
  own I2P stream and `GET`s the blob itself — its server is not involved at all
  (no bandwidth, no knowledge of the pull). The client uses a **transient SAM
  destination** (a throwaway dest) so the store cannot link downloads to each
  other or to a messaging identity. This is the preferred path.
- **Proxy fetch (no direct SAM access).** For environments that can only reach
  their own facade (mobile, restricted networks), the client asks its **own
  server** to proxy: the server opens the I2P stream, range-`GET`s, and streams the
  ciphertext back. The blob store is a neutral dumb destination, not a foreign
  messaging facade, so this does not breach facade-locality — the client still
  never connects to a foreign facade.

Both paths run the **same** I2P-fetch logic (range `GET` of ciphertext); only the
call site differs (client-side vs server-side). Direct-SAM availability here means
a router with a SAM bridge is reachable — independent of the trimmed `libi2pd`
already vendored in the client (that is for key generation / offline keys, not a
full router).

There is **no recipient-side re-hosting**: the sender serves the file, so the
recipient never copies the ciphertext into its own storage — re-hosting would only
load the recipient's quota for something the sender already provides. This follows
the project's storage principle — **keep the minimum**: a blob lives only until the
intended clients have received it (count) or its TTL elapses, then it is gone.

## Crypto

- Per-blob random symmetric key; ciphertext is CMS `AuthEnvelopedData` (AES-GCM)
  via OpenSSL — standard primitives only.
- Whole-blob `sha256` in the pointer for integrity; resume via HTTP range over the
  ciphertext bytes (integrity verified after full assembly). Chunked AEAD with a
  manifest (incremental verification, random access) is a later optimization, not
  v1.
- `GET`/`confirm` are authless because the payload is ciphertext and `blobId` is an
  unguessable bearer capability; `PUT`/`DELETE` are authenticated (sign-in-with-key
  + storage grant for `PUT`; delete-token for `DELETE`).

## Responsibilities

| Component | Role |
|---|---|
| `client-qt6` | 512 KB detection, encrypt/pack, upload, build+seal pointer; keep delete-token with meta; on receive: fetch (direct/proxy), validate, confirm, decrypt |
| `blob-storage` | upload (authed, quota-gated) / authless capability download / anonymous confirm; retention (TTL or count); dest pool + rotation; rate-limit |
| `service-node` | controls blob-storage per-user quota (storage grant + quota push, extending the existing publisher to a second target) |
| `facade-http` | fronts blob-storage uploads |
| `server-core` | untouched — only ever delivers the small pointer |
