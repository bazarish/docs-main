# File transfer

Status: implemented (transport + protocol; the sender must be online).

A file is **announced**, not uploaded. The message carries the file's name, size
and digest; the bytes stay on the sender's disk until the recipient asks for
them, at which point the sender raises a one-time I2P destination and serves that
one file over it. There is no storage service: nothing is ever parked on a third
party's disk.

```
 sender                                                     recipient
 ------                                                     ---------
 sendFile: hash + announce {name,size,sha256}  ---------->  file bubble, "Download"
                                                                  |
              <---------------- file.request {fileId} ------------+
 encrypt to a temp ciphertext (fresh key)
 raise a one-time destination (b33)
              ----------------- file.offer {host,key,sha256,size} ->
                                                            connect, stream, resume
              <================ direct I2P stream ==================
 tear the destination down, drop the temp    ------------>  verify digest, decrypt
```

## What the message carries

```
"file": { "name": ..., "size": <plaintext bytes>, "sha256": <plaintext digest>, "mime": ... }
```

The digest is over the **plaintext**, so the recipient can check that what it
finally holds is what was announced, independently of how many transfer attempts
it took, and independently of the per-transfer encryption below.

## Threshold: no constant

There is no client-side size threshold. A client sends normally and falls back to
a direct transfer when the server answers with its typed per-message limit error.
The threshold is therefore the server's own advertised limit, and it adapts per
server with nothing to keep in sync.

Operators should size that limit against the mailbox quota rather than against
the smallest thing that works: too small a limit pushes ordinary attachments into
a transfer that needs both sides online.

## Service messages

All three are ordinary sealed E2E messages, silent (never a chat bubble):

| Type | Direction | Payload |
|---|---|---|
| `file.request` | recipient -> sender | `fileId` (the announcing message's id) |
| `file.offer` | sender -> recipient | `offer` = `{fileId, host, key, sha256, size}` |
| `file.unavailable` | sender -> recipient | `fileId` |

`file.unavailable` is the honest answer when the sender no longer has the file -
it was unsent, or the user moved it. Without it the recipient would wait on a
transfer that can never start.

## The one-time destination

The sender serves from a destination created for this transfer and destroyed when
it ends, and the fetcher dials from a fresh one-time destination **on every
attempt**. Two transfers between the same two people are therefore not linkable
by address, and neither is a resumed attempt to the attempt it continues.

The offer's `host` is a b33 (encrypted LeaseSet), and it is shared only inside the
sealed offer.

## Wire protocol

Deliberately tiny - no HTTP:

```
fetcher -> server:  offset   (8 bytes, big-endian)
server  -> fetcher: total    (8 bytes, big-endian), then every byte from offset on
```

A dropped stream is not a special case: the fetcher reconnects and sends the next
offset. The driver gives up after a bounded number of attempts that move no
bytes, so a sender that goes quiet does not hang the transfer forever.

## Encryption and integrity

The sender encrypts the file to a temp ciphertext with a fresh per-transfer key
(CMS password-based envelope, RFC 3211 PWRI, AES-256-CBC - standard primitives
only). Encrypting up front, rather than on the fly, is what makes arbitrary byte
ranges servable and gives the digest before the first byte moves.

The key and the **ciphertext** digest travel in the sealed offer. The recipient
assembles the ciphertext, verifies that digest, and only then decrypts
file-to-file: a tampered or truncated transfer never yields cleartext.

The whole path is streamed. Neither side ever holds the file whole in memory, so
a file far larger than available RAM transfers fine.

## The cost, stated plainly

A file is available only while the sender is online **and** the file is still at
the path it was announced from. This is the deliberate price of having no storage
service, and the client must show it rather than implying delivery:

- `eRequested` - asked; waiting for the sender to come up
- `eRunning` - bytes moving (progress is over the ciphertext stream)
- `eDone`
- `eFailed` - with the reason, including "the sender no longer has this file"

## Responsibilities

| Component | Role |
|---|---|
| `client-qt6` | everything: announce, request, serve (one-time dest + temp ciphertext), fetch with resume, verify, decrypt, and report progress |
| `server-core` | untouched - it only ever carries the small announcement and the three service messages |
| `facade-http` | untouched - nothing streams through the facade any more |
