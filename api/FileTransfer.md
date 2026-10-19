# File transfer

Status: implemented (transport + protocol; the sender must be online).

A file is **announced**, not uploaded. The message carries the file's name, size
and digest; the bytes stay on the sender's disk until the recipient asks for
them, at which point the sender creates a one-time I2P destination and serves that
one file over it. There is no storage service: nothing is ever stored on a third
party's disk.

```
 sender                                                     recipient
 ------                                                     ---------
 sendFile: hash + announce {name,size,sha256}  ---------->  file bubble, "Download"
                                                                  |
              <---------------- file.request {fileId} ------------+
 encrypt to a temp ciphertext (fresh key)
 raise a one-time destination (b33)
              -------------- file.offer {host,header,key,sha256,size} ->
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
the smallest value that works: too small a limit pushes ordinary attachments into
a transfer that needs both sides online.

## Service messages

All four are ordinary sealed E2E messages, silent (never a chat bubble):

| Type | Direction | Payload |
|---|---|---|
| `file.request` | recipient -> sender | `fileId` (the announcing message's id), `device` (the requesting client id) |
| `file.offer` | sender -> recipient | `offer` = `{fileId, host, header, key, sha256, size}`, `forDevice` (the client id that asked) |
| `file.unavailable` | sender -> recipient | `fileId` |
| `file.cancel` | either direction | `fileId`, `device` (the client id that is pulling the file) |

**Devices.** A user's devices share one mailbox, so every one of them sees these
messages. A request names the device that made it; the offer names the same
device, and the others ignore it - the one-time destination in it was created for
that device alone. A second device requiring the file issues its own request, and
the sender serves it over a separate destination: several transfers proceed
concurrently,
each with its own address, its own progress and its own cancel. On the sending
side only the device that announced the file (and still holds the bytes) answers
at all; the others stay silent rather than reporting it unavailable.

`file.unavailable` is returned when the sender no longer holds the file -
it was unsent, or the user moved it. Without it the recipient would wait on a
transfer that can never start.

## Stopping

A transfer is stopped from **either** side, and stopping it ends **both** halves:
the side that stops sends `file.cancel`, and the other side ends its half of the
same transfer on receipt. A transfer costs both of them a destination and a set of
tunnels, so a stop that ended only the half on which it was issued would leave
the other half serving no one until its window expired - which is up to half an
hour of tunnels for a file no one is receiving.

`device` names the client that is pulling the file, in both directions, because
that is what identifies the half: one contact may be pulling the same file onto
two devices at once, each over its own one-time address. A stop that names one of
them leaves the other transfer running, and a `file.cancel` naming a device that
is not the recipient's own changes nothing there.

A stop is reported as an ended transfer with its reason, not as a failure of the
transport, and the side that issued it and the side that was notified are
distinguished: neither is left with a progress bar that continues to move.

## The one-time destination

The sender serves from a destination created for this transfer and destroyed when
it ends, and the fetcher dials from a fresh one-time destination **on every
attempt**. Two transfers between the same two people are therefore not linkable
by address, and neither is a resumed attempt to the attempt it continues.

The offer's `host` is a b33 (encrypted LeaseSet), and it is shared only inside the
sealed offer.

## Wire protocol

Deliberately minimal - no HTTP:

```
fetcher -> server:  offset   (8 bytes, big-endian)
server  -> fetcher: total    (8 bytes, big-endian), then every byte from offset on
```

A dropped stream is not a special case: the fetcher reconnects and sends the next
offset. The driver stops after a bounded number of attempts that transfer no
bytes, so a sender that stops responding does not block the transfer
indefinitely.

## Encryption and integrity

The sender encrypts the file to a temp ciphertext with a fresh per-transfer key
(CMS password-based envelope, RFC 3211 PWRI, AES-256-CBC - standard primitives
only). Encrypting in advance, rather than on the fly, is what makes arbitrary byte
ranges servable and gives the digest before the first byte is sent.

The envelope's encrypted content is detached (RFC 5652 lets `encryptedContent`
be absent): what is served is the bare ciphertext, and the envelope itself - a
few hundred bytes of recipient info and parameters - travels base64 in the offer
as `header`. A CMS decoder reads an envelope that carries its content whole into
memory, so a recipient refuses one that does.

The header, the key and the **ciphertext** digest travel in the sealed offer. The
recipient assembles the ciphertext, verifies that digest, and only then decrypts:
a tampered or truncated transfer never yields cleartext.

The whole path is streamed. Neither side ever holds the file whole in memory, so
a file substantially larger than available memory transfers without difficulty.

## Availability constraint

A file is available only while the sender is online **and** the file remains at
the path from which it was announced. This is the consequence of operating no
storage service, and a client must report it rather than imply delivery:

- `eRequested` - asked; waiting for the sender to come online
- `eRunning` - bytes being transferred (progress is over the ciphertext stream)
- `eDone`
- `eFailed` - with the reason, including "the sender no longer has this file"

## Responsibilities

| Component | Role |
|---|---|
| `client-qt` | everything: announce, request, serve (one-time dest + temp ciphertext), fetch with resume, verify, decrypt, and report progress |
| `server-core` | untouched - it carries only the small announcement and the four service messages |
| `facade-http` | untouched - nothing streams through the facade any more |
