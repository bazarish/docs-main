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
- **Readers** - the presence keys of up to 100 contacts, taken in the order of
  the chat list (pinned chats first, then the most recent conversation). The
  oldest dialogs stop receiving the status. A blocked contact and a contact whose
  card is not a `user` card are not readers. The account's own key is not one:
  its devices do not watch each other.
- **A change of the list** - a contact added, removed or blocked, or the order
  crossing the 100th place - re-creates the destination after the list has been
  still for `readersSettle`. The lease set is read at creation only, so there is
  no other way to change it.
- **It is up** on every device of the account that is in focus, each with its
  own tunnels, plus the tail.
- **The switch** "Show online status" keeps it down when off, and the device then
  watches no one either, since a watch names its sender: with the switch off the
  user sees no one's status. The card does not change, so the card does not
  reveal the setting.

A device watches from a **watching address** of its own: a published, minted
destination that carries the account's presence secret as its lease-set reader
key, so that it can read the lease sets that list the account. It exists while
the device is in focus with the switch on. One is kept built ahead of time and
given the secret when it is taken, so a device that gains focus watches at once.

## Several devices

A device that gains focus raises the presence destination at once; nothing is
asked of the account's other devices first. A device holding it is a **holder**.

1. A holder answers for itself alone: "online" while it is in focus and for the
   tail after, and "typing" to the contacts it is typing to.
2. Two devices in focus at once both hold the destination. The newer lease set
   wins at the floodfills, and a contact that found one of them keeps it: it sees
   that device's typing, not the other's.
3. A holder whose tail runs out tells every subscriber "offline" with `moved`
   and takes the destination down. The same goes for a holder that re-creates the
   destination for a new reader list, and for the switch turned off, without
   `moved`.

## Watching

A device in focus with the switch on watches the contacts on its screen: the
open conversation and the rows of the chat list in view, and no others. For each
it sends a **watch**, renews it every `renewEvery`, and the holder answers each
watch and renew, and every change of state, with a **state**. A watch not yet
answered is resent every `watchRetry`. A contact that leaves the screen is no
longer watched and is shown offline.

A destination that has found a lease set keeps it until the lease set expires,
up to `leaseSetLife` after its holder is gone, and does not look again, so a
contact whose holder changed is reached only from a new watching address. The
device moves all of its watches to a new one when:

- a state says `moved`;
- an answered watch has had no state for `subscriptionLife`;
- a watch to a contact heard from within `leaseSetLife` has gone unanswered for
  `subscriptionLife`;
- a watch from a contact arrived, which says the contact's address is up, and
  the device's own watch to that contact is still unanswered `watchRetry` after
  it. The own watch is resent at once when such a watch arrives;
- its own presence address has just come up.

A contact shown online before the move stays online, without typing, until it
answers or for `subscriptionLife`.

Watches go out as soon as the device is in focus, without waiting for its own
presence address: a contact on the same gateway is met inside it (the gateway's
`PROTOCOL.md`) and answers at once. A contact woken by such an early watch may
look the address up before it is published and find the one before; the move
that follows the address coming up sends every watch again from a new watching
address, which wakes that contact once more.

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
`{ t: "presence.watch", from, reply, sub, key, sentAt, sig }`, sealed with
the hybrid seal to the watched account's sealing key, so only that account learns
who is watching, now or after a quantum computer exists:

- `from` - the watcher's fingerprint;
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
not one of its readers, whose `sig` does not verify
against the Ed25519 identity key of that `from`, or whose `sentAt` is more than
120 seconds away from its own clock.

**Renew** and **state** are sealed under the subscription key. The nonce is
`direction || 0 0 0 || sequence`, with direction 1 towards the holder and 2 from
it, and a 64-bit big-endian sequence that only grows: a frame whose sequence is
not above the last one opened is dropped, as is one whose direction is wrong.

- A renew is `{}`.
- A state is `{ online, typing }`, `typing` meaning typing to the watcher, and
  `moved: true` in the last one before the holder takes the destination down.

Measured sizes: a watch is 1447 bytes, most of it the hybrid seal; a renew is 109
and a state 64.

### Typing

Composing in a conversation marks typing to that contact for `typingPulse`. A
contact sees typing only while it also sees online.

### Timings

| Name | Value |
|---|---|
| `watchRetry` | 5 s |
| `renewEvery` | 15 s |
| `subscriptionLife` | 30 s |
| `onlineTail` | 60 s |
| `typingPulse` | 5 s |
| `readersSettle` | 10 s |
| `leaseSetLife` | 660 s - i2pd's tunnel lifetime, which bounds a lease set's |

## Through a gateway

A client on an external gateway cannot keep the presence address to itself: the
router that builds it and reads the lease sets of contacts is the gateway's. It
hands over the least that works:

- **The presence address as a delegation, never its master key.** The gateway
  gets an offline transient that ends with the UTC day. Before it ends the
  gateway asks for the next one and the client sends a delegation to the end of
  the following day; with none, the gateway takes the address down a minute
  before the end. The client keeps no clock for it. An operator can keep the
  address silent while a delegation runs, and cannot raise it after.
- **The reader list,** which names the presence keys of up to 100 contacts.
- **The account's presence secret,** to read the lease sets that list the
  account. A secret that decrypts cannot be delegated, so this is the lasting
  part: the operator can see when those contacts are online for as long as the
  secret stands, and a new secret means a new card.

It can neither read a watch nor forge one, since the keys for both never leave
the device. The wire is in the gateway's `PROTOCOL.md` (`bazarish-server/gateway`).

## What anyone learns

- **The account's server:** nothing. No mailbox carries presence.
- **An external gateway, for a user who goes through one:** what is above.
- **A contact listed as a reader:** the account's status and typing to that
  contact, while it watches.
- **A watched contact:** that the watcher has the app in focus with that contact
  on screen, since a watch names its sender. A device with the switch off sends
  no watch.
- **Anybody else:** nothing. Without a reader's secret the presence address
  cannot be found, and a watch from a fingerprint that is not a reader is dropped.
