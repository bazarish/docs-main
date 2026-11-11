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

Every `user` account holds one **presence seed**: 32 random bytes, made with the
account and never changed. It travels in the account backup and the pairing
bundle (`DeviceSync.md`) and nowhere else. Everything presence publishes is
derived from it for one UTC day `D`, written `YYYYMMDD`, with HKDF-SHA256 over
the seed, no salt and the info string followed by `|` and `D`:

| Derived | Info | Use |
|---|---|---|
| `dest(D)` signing key | `bazarish presence dest v1` | Ed25519 signing key of the day's presence destination |
| `dest(D)` crypto key | `bazarish presence dest crypto v1` | X25519 key of that destination's identity |
| `reader(D)` | `bazarish presence reader v1` | X25519 lease-set reader key of the account for the day |

The destination's identity pads both key fields with zeros, so every device of
the account derives the same destination byte for byte.

- `dest(D)` is a b33 of an **encrypted LeaseSet2 with per-client X25519
  authorisation** (I2P proposal 123, DH client auth). Its lease set is stored
  under a key derived from the b33 and the date: whoever knows the b33 can fetch
  the outer record and learn that the address is published and until when;
  only a listed reader's secret opens the leases, so only a reader can reach it.
- **Readers** - for day `D`, the `reader(D)` keys that up to 100 contacts sent
  in their schedules, taken in the order of the chat list (pinned chats first,
  then the most recent conversation). The oldest dialogs stop receiving the
  status. A blocked contact, a contact whose card is not a `user` card and a
  contact with no schedule for `D` are not readers. The account's own key is
  not one: its devices do not watch each other.
- **A change of the list** - a contact added, removed or blocked, or the order
  crossing the 100th place - re-creates the destination after the list has been
  still for `readersSettle`; a contact who leaves the list is refused at once.
  The lease set is read at creation only.
- **It is up** on every device of the account that is in focus, each with its
  own tunnels, plus the tail.
- **The day turns over** with `dayOverlap`: from 5 minutes before 00:00 UTC to
  5 minutes after, the addresses of both days are up and both are watched. At
  the end of the overlap each holder of the old day's address tells its
  subscribers "offline" with `moved` and takes it down.
- **The switch** "Show online status" keeps it down when off, and the device then
  watches no one either, since a watch names its sender: with the switch off the
  user sees no one's status. No contact is told of the setting.

### The schedule

A contact learns the account's addresses and reader keys from a **schedule**,
the silent `presence.schedule` message (`Messages.md`):

```
{ days: [ { d: "YYYYMMDD", dest: "<b33>.b32.i2p", reader: "<32-byte X25519 public key, base64>" }, ... ] }
```

- It names `horizon` consecutive days from the day it is sent.
- It goes to a contact when the contact becomes mutual, on both sides, and again
  when the days left in the last schedule sent to that contact fall below
  `refreshBelow`. The account checks this when it opens and when the day
  changes, and spreads the sends over `scheduleSpread`.
- A receiver drops a schedule that comes from anyone but a `user` contact who
  is not blocked, that skips or repeats a day, that names more than `horizon`
  days or none, that starts before yesterday, or whose host is not a b33 or
  whose reader key is not 32 bytes. It replaces the schedule it holds with one
  that starts no earlier, and forgets the days that are past.
- Schedules travel between the account's own devices with the address book
  (`DeviceSync.md`).
- Only public halves leave the account's devices; the seed never does.

A device watches from a **watching address** of its own: a published, minted
destination that carries the account's `reader(D)` as its lease-set reader key,
so that it can read the day's lease sets that list the account. It watches the
`dest(D)` of each contact's schedule; a contact with no schedule for the day is
not watched and is shown offline. It exists while
the device is in focus with the switch on. While at least one account open on the
device has the switch on, one is kept built ahead of time and given the reader key
when it is taken, so a device that gains focus watches at once; with the switch
off on every open account, the device holds no presence or watching destination
at all. Each account takes its own watching address and keeps its own presence
address: two accounts on one device that watch the same contact each watch from
an address of their own.

## Several devices

A device that gains focus raises the presence destination at once. A device
holding it is a **holder**.

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
- `reply` - the watching address the state goes to. The holder does not check
  it: a reader may name any address, and that address then receives the holder's
  sealed state frames for the life of the subscription and nothing more;
- `sub` - the subscription id, in hex, equal to the one in the frame header;
- `key` - a fresh 32-byte AES-256-GCM key for this subscription, base64;
- `sentAt` - Unix seconds;
- `sig` - an Ed25519 signature by the classical half of the watcher's identity
  over the ASCII string `bazarish presence watch v1` followed by the CBOR of the
  watch without `sig`.

The signature is Ed25519 alone, the one exception to the rule that an identity
signs with both halves (`Identity.md`). It is checked once, when the watch
arrives, and a watch older than 120 seconds is dropped, so forging one later
gains nothing. The seal is hybrid, because what it hides - who watches whom - is
worth recording now and reading later. Neither key ever leaves the device, so a gateway that holds the
day's reader key can neither read a watch nor forge one.

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
contact sees typing only while it also sees online. Typing ends at once when
the message is sent, the field is emptied or it loses focus: the holder sends the new state on its next tick.

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
| `dayOverlap` | 10 min, 00:00 UTC in its middle |
| `horizon` | 14 days |
| `refreshBelow` | 7 days |
| `scheduleSpread` | 1 h |

## Through a gateway

A client on an external gateway cannot keep the presence address to itself: the
router that builds it and reads the lease sets of contacts is the gateway's. It
hands over the least that works, all of it for one day:

- **`dest(D)` as a delegation, never its master key.** The gateway gets an
  offline transient of the day's address that ends with a UTC day. Before it
  ends the gateway asks for the next one and the client sends a delegation to
  the end of the following day; with none, the gateway takes the address down a
  minute before the end. An operator can keep the address silent while a
  delegation runs, and cannot raise it after. The address is `D`'s alone:
  contacts stop looking for it when the day's overlap ends.
- **The day's reader list,** the `reader(D)` public keys of up to 100 contacts.
- **`reader(D)`,** the account's reader secret for the day, to read the day's
  lease sets that list the account.

The seed never reaches the gateway, so nothing it holds leads to another day:
after the client leaves, what it kept opens only lease sets and addresses that
are no longer looked for. It can neither read a watch nor forge one, since the
keys for both never leave the device. The wire is in the gateway's
`PROTOCOL.md` (`bazarish-server/gateway`).

## What anyone learns

- **The account's server:** nothing. No mailbox carries presence.
- **An external gateway, for a user who goes through one:** for each day it
  serves, what is above: the account's `dest(D)`, the `dest(D)` of the contacts
  it watches, when they are online and the day's readers. Nothing for a day it
  does not serve.
- **A contact listed as a reader:** the account's status and typing to that
  contact, while it watches.
- **A contact that is not a reader, or one removed or blocked:** the schedule it
  was last sent, up to `horizon` days ahead. It cannot reach the address, but
  until that schedule runs out it can look up the outer lease set of each day's
  address and so tell when the account is online.
- **A watched contact:** that the watcher has the app in focus with that contact
  on screen, since a watch names its sender. A device with the switch off sends
  no watch.
- **Anyone else, including someone holding the account's descriptor or card:**
  nothing. The card names no presence address, one day's address does not lead
  to another's without the seed, and a watch from a fingerprint that is not a
  reader is dropped.
