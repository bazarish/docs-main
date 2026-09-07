# Forward secrecy: what it would take here

Status: **research, not implemented**. Nothing in this document describes what
Bazarish does today. It exists to be argued with before any of it is built.

## Where we stand

An account has one sealing key: hybrid by construction (EC P-256 + ML-KEM-768),
generated once, never rotated. It is published in the contact card, handed to
correspondents again in the `bootstrap` of a first reply, and **everything** is
sealed to it - the first contact request read from the card and every message
afterwards through the correspondent's stored copy of the same value. Incoming
mail is opened with the same private half.

The consequence, stated without softening: **whoever obtains that private half
reads everything the account was ever sent.** Not only future traffic - the
mail still sitting in mailboxes, and any ciphertext an observer kept. There is
no ratchet, no rotation, no window. The key lives in the account's encrypted
database and is copied to every device of the account at import.

That is a real property, not a gap in the implementation: nothing in the protocol
was ever designed to bound it. Two things follow.

- The word "prekey", which used to appear in the code and the docs for this key,
  promised something the design does not deliver. It is gone now; the docs say
  plainly that the key is long-lived and that there is no forward secrecy.
- The serving key - the one whose private half the **server** holds, and which
  protects only routing metadata - *does* have a rotation mechanism
  (`prepareServingKey` / `commitServingKey`, two-step, with a re-signed card).
  The key that protects message content has none. The cheaper secret is better
  looked after than the dearer one.

## What forward secrecy would actually buy

Worth being concrete, because "PFS" is often invoked without saying against
whom.

| Scenario | Today | With FS |
|---|---|---|
| Device seized, passphrase compelled | All past conversations readable | Only what is still within the ratchet's window |
| Mailbox seized at the server | Ciphertext is opaque, but becomes readable if the key is ever obtained later | Stays opaque: the keys that opened it are gone |
| Key exfiltrated by malware, quietly | Everything, past and future, until the key changes - which is never | Past is safe; future is safe again after the next ratchet step (post-compromise security), if the design has that too |
| Traffic recorded on the network | Same as the mailbox row | Same |

The middle two rows are the ones that matter for this project's stated threat
model, where the network is hostile and a server's disk may be read.

## The state of the art, as of this writing

- **Signal** has run PQXDH for the initial handshake since 2023, and since
  October 2025 the ratchet itself is post-quantum: **SPQR** (Sparse Post Quantum
  Ratchet) runs alongside the classic Double Ratchet, and the pair is called the
  **Triple Ratchet**. The interesting engineering detail for us is *why* it is
  "sparse": ML-KEM public keys and ciphertexts do not fit in a message header, so
  the material is cut into chunks and spread across several messages, with keys
  from both ratchets combined through a KDF so an attacker must break both
  layers. Downgrade to the plain Double Ratchet for older peers is
  cryptographically authenticated rather than silent.
- **MLS (RFC 9420)** is an IETF standard for asynchronous group key establishment
  with forward secrecy and post-compromise security, defined for groups "from two
  to thousands" - the two-party instantiation is sometimes called 2MLS.
  Post-quantum ciphersuites are an IETF draft (`draft-ietf-mls-pq-ciphersuites`),
  and implementations exist; for a C++ codebase the relevant one is mlspp.

## The constraints that decide this, and they are ours, not Signal's

**1. One key, many devices - and the device count is private.**

This is the hard one. Every device of an account shares the *same* identity and
the *same* sealing key (an import copies `sealingPem` verbatim). A message is
placed in the account's mailbox **once**, and every registered device fetches it
independently; a device may be offline for days and fetch it later.

A ratchet is a piece of state that must advance in lockstep with the sender. If
one device advances it, the others cannot open what follows. Signal solves this
by treating every device as its own session and having the sender encrypt N
copies - which is exactly what tells the sender how many devices the recipient
has. This project deliberately treats that number as private: the mailbox is
per account, and `GET /v1/messaging/clients` answers only about one's own.

So the first question is not "which ratchet" but: **are we willing to let a
correspondent learn how many devices we have?** Every answer below depends on it.

There is a middle path worth studying: an account-level key that own devices
share and re-key among themselves over the existing device-sync channel
(`DeviceSync.md` already carries contact books and token grants between devices).
That keeps the device count private and gives forward secrecy against an outside
attacker, at the cost that a compromised device can still read the account's
traffic until the account re-keys - which is arguably the honest security level
for a design where all devices share one identity anyway.

**2. Store-and-forward, with long gaps.**

Mail waits up to the retention window (14 days by default) and arrives out of
order. A ratchet has to keep skipped-message keys for that long, which means
they are on disk, which means they are part of what a seized device gives up -
the very thing forward secrecy was meant to remove. Whatever is chosen, the
retention of skipped keys is a parameter with a security meaning and must be
stated, not left to a library default.

**3. The first message has no room.**

The contact request is tokenless and capped: the measured worst case is 11668
bytes against a cap of 11700. There is no headroom. A hybrid handshake that puts
ML-KEM material in the first message does not fit - which is precisely the
problem SPQR solves by spreading key material over several messages. Any design
here must say what the *first* message carries and what waits for the second.

**4. The project's own rule.**

`CLAUDE.md`: "creating one's own implementations of cryptographic algorithms or
primitives is not permitted". A ratchet is a protocol built from standard
primitives rather than a primitive itself, so it is not literally forbidden - but
hand-writing a Double Ratchet is the sort of thing that rule exists to prevent.
This argues strongly for adopting a standardised protocol with an existing
implementation over designing a scheme of our own, however small it looks.

## The ladder

Ordered by cost. Each row says what it actually delivers, not what it sounds
like.

1. **Change nothing; keep saying so.** Cost: none, already done. Delivers: no
   secrecy, but no false impression either. The reference point for the rest.
2. **Rotate the account sealing key on a schedule.** Cost: small - the mechanism
   already exists for the serving key, and the card is already versioned by
   `issuedAt`. Delivers: a *window*, not forward secrecy: everything within the
   current period is still opened by one key, and old keys must be kept as long
   as old mail may arrive. Honest description: "a seized device gives up the last
   N days", which is a real improvement over "everything".
3. **Symmetric ratchet per conversation.** A KDF chain advanced per message, no
   new key exchange. Cost: moderate; the state is per correspondent and must be
   synchronised across own devices (constraint 1 in full force). Delivers:
   forward secrecy against later compromise; **no** recovery after a compromise -
   an attacker who takes the chain key follows it forever.
4. **Double ratchet, hybrid (Signal-shaped).** Cost: high - it is the multi-device
   problem, the skipped-key problem and the first-message size problem all at
   once, and either a Rust dependency (libsignal) or our own implementation,
   which rule 4 discourages. Delivers: forward secrecy and post-compromise
   security, the strongest of the options.
5. **2MLS with a post-quantum ciphersuite.** Cost: high, but a different kind:
   a standard, a C++ implementation to evaluate (mlspp), and PQ ciphersuites
   still at draft. Delivers: forward secrecy and post-compromise security, and -
   the reason it is on this list despite the weight - **multi-device falls out of
   the model rather than being bolted on**: each device is a leaf of the group,
   which is what MLS is for. It also fits the project's rule better than anything
   else here, because it is an RFC with implementations rather than a scheme we
   would be writing.

## Recommendation

Decide constraint 1 first - whether the number of a user's devices may become
visible to their correspondents. It is a product question, not a cryptographic
one, and every option below the first depends on the answer.

If the answer is **no** (the count stays private), then options 4 and 5 in their
standard shapes are both off unless devices share ratchet state, and the honest
shortlist is (2) now and (3) or an MLS group whose leaves are *own devices*
later.

If the answer is **yes**, 2MLS deserves a serious evaluation before anything is
written: it is the only option on this list that is a standard with a C++
implementation, it gives post-compromise security as well as forward secrecy,
and multi-device is native rather than grafted.

Either way, (2) is worth doing regardless and independently: it is small, it uses
mechanisms that already exist, and it turns "everything, forever" into "the last
N days" while the larger question is settled.

## Sources

- [Signal Protocol and Post-Quantum Ratchets](https://signal.org/blog/spqr/)
- [Triple Threat: Signal's Ratchet Goes Post-Quantum (Quarkslab)](https://blog.quarkslab.com/triple-threat-signals-ratchet-goes-post-quantum.html)
- [RFC 9420 - The Messaging Layer Security (MLS) Protocol](https://datatracker.ietf.org/doc/html/rfc9420)
- [ML-KEM and Hybrid Cipher Suites for MLS (IETF draft)](https://messaginglayersecurity.rocks/mls-pq-ciphersuites/draft-ietf-mls-pq-ciphersuites.html)
