# Presence & Typing

Client: `client-qt`; engine: `common` (`client/Presence`).

## What is shown

- **Online** - the app window is open and in focus, on a desktop and on a phone
  alike, with a one-minute tail after focus is lost so that the status does not
  flicker. A running core with no focused window is not online, and neither is
  an account that is switched off.
- **Typing** - the user is composing in the conversation with that contact.

Neither is stored anywhere: both are signals that live for seconds. No server
learns anything from either, because presence never travels through a mailbox:
it goes between devices over I2P, as call media does.

Only `user` accounts have presence. A `bot` or `chat` card carries no presence
fields (`Identity.md`), and a client shows none for it.

## The presence address

Every `user` account holds one **presence destination** and one **presence
key**, an X25519 pair. Both are made with the account, never change, travel in
the account backup and the pairing bundle (`DeviceSync.md`), and are published
in the contact card as `presenceDest` and `presenceKey`.

- The destination is a b33 of an **encrypted LeaseSet2 with per-client X25519
  authorisation** (I2P proposal 123, DH client auth). Without a listed reader's
  secret its lease set cannot be read, so the address cannot be reached or even
  found.
- **Readers** - at most 100: the account's own presence key, so that its other
  devices reach it, and the presence keys of up to 99 contacts, taken in the order
  of the chat list (pinned chats first, then the most recent conversation). The
  oldest dialogs stop receiving the status. A blocked contact and a contact whose
  card is not a `user` card are not readers.
- **A change of the list** - a contact added, removed or blocked, or the order
  crossing the 99th place - re-creates the destination after the list has been
  still for `readersSettle`. The lease set is read at creation only, so there is
  no other way to change it.
- **It is up** while a device of the account is in focus, plus the tail.
- **The switch** "Show online status" keeps it down when off. The card does not
  change, so the card does not reveal the setting. The switch is not reciprocal:
  hiding one's own status still shows others'.

A device watches from a **watching address** of its own: a published, minted
destination that carries the account's presence secret as its lease-set reader
key, so that it can read the lease sets that list the account. It exists while
the device is in focus.

## Several devices: holder and siblings

Exactly one device holds the presence destination at a time. Two routers
publishing it at once would overwrite each other's lease set, and a contact would
reach one of them at random.

1. A device that gains focus watches its **own** presence address.
2. An answer means another device is the **holder**, and this one becomes a
   **sibling**. A sibling renews every `siblingKeepalive` with whether it is in
   focus and whom it is typing to, and the holder speaks for it.
3. No answer within the probe patience - `probeDeadline` plus a random share of
   up to half of it, so that two devices that start together rarely give up
   together - and the device raises the destination and becomes the holder.
4. The holder answers "online" while it or any sibling is in focus, and keeps
   the destination up for the tail after the last of them, so there is no
   handover and no gap.
5. A sibling that hears nothing from the holder for `holderSilence` raises the
   destination itself; its newer lease set replaces the old one.
6. A holder in focus watches its own address every `selfCheckEvery`. When the
   answer comes from another device, two of them raised it at once: this one
   takes its destination down and continues as that device's sibling.
7. A holder whose tail runs out tells every subscriber "offline" before it takes
   the destination down.

## Watching

A device in focus watches the contacts on its screen: the open conversation and
the rows of the chat list in view, and no others. For each it sends a **watch**,
renews it every `renewEvery`, and the holder answers each watch and renew, and
every change of state, with a **state**. A watch that has had no state for
`subscriptionLife` is started again, and the contact is shown offline meanwhile.
A contact that leaves the screen is no longer watched and is shown offline.

### Frames

Every datagram is a raw I2P datagram, from a watching address to a presence
address and back:

```
type (1 byte) || subscription id (8 bytes) || body
```

| Type | Name | Direction | Body |
|---|---|---|---|
| 1 | `watch` | watcher -> holder | `hybrid::seal` to the account's `sealingKey` of the CBOR below |
| 2 | `renew` | watcher -> holder | `nonce (12) || AES-256-GCM(CBOR)` |
| 3 | `state` | holder -> watcher | `nonce (12) || AES-256-GCM(CBOR)` |

The **watch** is
`{ t: "presence.watch", from, device, reply, sub, key, sentAt, sig }`, sealed with
the hybrid seal to the watched account's sealing key, so only that account learns
who is watching, now or after a quantum computer exists:

- `from` - the watcher's fingerprint; `device` - the watcher's client id;
- `reply` - the watching address the state goes to;
- `sub` - the subscription id, in hex, equal to the one in the frame header;
- `key` - a fresh 32-byte AES-256-GCM key for this subscription, base64;
- `sentAt` - Unix seconds;
- `sig` - an Ed25519 signature by the classical half of the watcher's identity
  over the ASCII string `bazarish presence watch v1` followed by the CBOR of the
  watch without `sig`.

The signature is Ed25519 alone, the one exception to the rule that an identity
signs with both halves (`Identity.md`). It is checked once, when the watch
arrives, and a watch older than 120 seconds is dropped, so forging one later
gains nothing: a post-quantum signature would guard against a forger who breaks
Ed25519 within those seconds, and costs 3309 bytes on every watch. The seal stays
hybrid, because what it hides - who watches whom - is worth recording now and
reading later. Neither key ever leaves the device, so a gateway that holds the
presence secret can neither read a watch nor forge one.

The holder drops a watch whose type or `sub` does not match, whose `from` is
neither the account itself nor one of its readers, whose `sig` does not verify
against the Ed25519 identity key of that `from`, or whose `sentAt` is more than
120 seconds away from its own clock.

**Renew** and **state** are sealed under the subscription key. The nonce is
`direction || 0 0 0 || sequence`, with direction 1 towards the holder and 2 from
it, and a 64-bit big-endian sequence that only grows: a frame whose sequence is
not above the last one opened is dropped, as is one whose direction is wrong.

- A contact's renew is `{}`. A sibling's is `{ focused, typing }`, `typing` being
  the fingerprints of the contacts it is typing to.
- A state is `{ online, typing }`, `typing` meaning typing to the watcher. To the
  account's own devices it also carries `device`, the holder's client id, which
  is how a holder finds a second holder (step 6). A contact is never told it.

Measured sizes: a watch is 1447 bytes, most of it the hybrid seal; a renew is 109
and a state 64.

### Typing

Composing in a conversation marks typing to that contact for `typingPulse`. A
sibling forwards it to the holder at most every `typingResend`. A contact sees
typing only while it also sees online.

### Timings

| Name | Value |
|---|---|
| `watchRetry` | 3 s - a watch not yet answered is resent, for `probeDeadline`; then every `renewEvery` |
| `renewEvery` | 15 s |
| `subscriptionLife` | 30 s |
| `siblingKeepalive` | 5 s |
| `holderSilence` | 30 s |
| `probeDeadline` | 20 s, plus a random share of up to 10 s |
| `onlineTail` | 60 s |
| `typingPulse` | 5 s |
| `typingResend` | 2 s |
| `readersSettle` | 10 s |
| `selfCheckEvery` | 60 s |

## Through a gateway

A client on an external gateway hands it the presence destination's keys, the
reader list and, for a watching address, the account's presence secret, because
the router that builds and reads those lease sets is the gateway's. The
operator then holds a lasting ability to see when the account's contacts who list
it are online, and to keep the account's presence address silent. It cannot
read a watch or forge a state. The wire and the full cost are in the gateway's
`PROTOCOL.md` (`bazarish-server/gateway`).

## What anyone learns

- **The account's server:** nothing. No mailbox carries presence.
- **An external gateway, for a user who goes through one:** what is above.
- **A contact listed as a reader:** the account's status and typing to that
  contact, while it watches.
- **A watched contact:** that the watcher has the app in focus with that contact
  on screen, since a watch names its sender. This holds with the watcher's own
  switch off: the switch hides the watcher's status, not its watching.
- **Anybody else:** nothing. Without a reader's secret the presence address
  cannot be found, and a watch from a fingerprint that is not a reader is dropped.
