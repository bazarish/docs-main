# Federation API (Server ↔ Server)

Status: draft skeleton.

## Transport

- Delivery is **I2P-only**: the messaging server requires `--i2p` (there is no
  non-I2P transport; the earlier plain-TCP federation was removed). Each server
  runs an I2P **STREAM** session via the SAM v3 bridge. The destination is
  persisted (`<state>/i2p-dest.key`) so the address is stable across restarts.
- Every destination is published as an **encrypted LeaseSet2**, so its shareable,
  routable address is the blinded **b33** (`<b33>.b32.i2p`) — the raw destination
  is not connectable (no plain leaseset is published). The one exception is a
  paid per-user offline-key destination, which can only publish a **standard
  LeaseSet2** (the raw destination is its address) — offline keys cannot sign a
  b33 (see `Identity.md`, `InviteAnonymity.md`).
- The delivery framing operates on an abstract connected stream, so a future
  overlay transport would only need to supply a byte stream; nothing above the
  transport assumes I2P specifics.
- Wire format matches the rest of the system: JSON control messages + raw binary bodies.

### Delivery framing (implemented)

Over the connected stream, one delivery is:

```
request:  {"op":"deliver","sealed":<base64>,"len":<N>}\n   then N raw payload bytes
response: {"delivered":<bool>,"errorCode":...,"errorMessage":...}\n
```

Length-prefixed both ways — no half-close (SAM propagates a SHUT_WR as a full stream teardown). The recipient runs the same delivery pipeline as a local send (dedup → token → size/quota → store), so local and federated deliveries share one code path.

Operational note: a freshly created destination must publish its leaseset before peers can reach it; the first connect after startup may return `CANT_REACH_PEER` ("LeaseSet not found") and should be retried with backoff. Verified end to end against a live i2pd (`i2p-federation-smoke`, and the full cross-server `demo-federation-i2p.sh`).

## Destination routing

Routing is **by destination**, not by server fingerprint. The recipient's
serving destination (its b33) travels end to end in the recipient's contact card
(`../Identity.md`), so the sender's server **dials that destination directly**
over I2P. There is no fingerprint→endpoint resolver and no `peers.json` — a
destination is self-routing. A delivery whose `toDest` is one of the sender's
server's own destinations is stored locally **only when it equals the sender's
own destination** (same destination); a co-located recipient on a *different*
destination is still federated (self-dialled) so delivery latency cannot reveal
co-location (`InviteAnonymity.md`).

## Delivery envelope

Built entirely by the sending **client**; servers never see more than their layer:

```
DeliveryEnvelope:
  toDest: <recipient I2P destination>       # routing only — all the sender's server sees
  sealed:                                   # CMS envelope, encrypted to the serving sealing
                                            # key of that destination (from the contact card)
    class: content | contact                # delivery class (admission selector), not the
                                            # content type, which is E2E — see ../Messages.md
    mailbox: <recipient UID>
    token: <one-time delivery token>        # absent for class "contact"
    messageId: <client-generated random id> # idempotency / dedup
  payload: <E2E blob>                       # opaque to every server-side component
```

- The sender's own server learns only "user U sent something to destination D"
  (plus size and time) — never the recipient identity. For a free user D is a
  shared address-pool destination (k-anonymity); for a paid user it is their
  personal destination.
- The recipient's server learns only "something arrived from destination X into mailbox V with a valid token" — never the sender identity (see `../Contacts.md`).

## Message delivery flow

1. Sender's client → own server: `send` with the envelope. The server accepts it **in memory only** (no outbound persistence, no outbound queue) and returns an attempt id; the client polls for the outcome (see `ClientApi.md`).
2. Sender's server → recipient's server over I2P: `deliver` with the envelope.
3. Recipient's server unseals and processes in strict order:
   1. **messageId dedup** — if the `messageId` is still pending in the mailbox, or is in the retained **seen-messageId set** (every admitted id, kept independently of the blob until the TTL sweep ages it out), respond success *without* consuming a token. This makes client retries idempotent and blocks replays of an already-delivered envelope — including tokenless `contact` requests that have since been fetched, acked and swept;
   2. **token check** — hash the presented token, look it up in the mailbox's registered set (`../Contacts.md`); no match → `DELIVERY_REJECTED`;
   3. **size/quota checks** — explicit typed errors (`QUOTA_EXCEEDED`, `STORAGE_FULL`), revealed only past a valid token;
   4. **store** as pending content; the token is consumed atomically with the store, and the `messageId` is added to the seen set.
4. The outcome propagates back to the sender's client via polling. On failure the message stays **only on the client**, which may retry with the same messageId; the sender's server drops its in-memory copy as soon as the attempt concludes.

`contact-request` deliveries skip the token check, are capped in payload size (default 256 bytes, server-configurable), and are subject to a separate quota and per-origin-server rate limiting.

## Requests (draft)

| Request | Purpose |
|---|---|
| `deliver` | Deliver an envelope (message or contact request) for a local mailbox |
| `resolve` | Sealed alias-resolution query (see below) |
| `card` | Fetch the current server card (TBD whether exposed here) |

## Facade locality

A facade is the client↔own-server last mile and **never** crosses a server
boundary (see `Facade.md`). Servers reach each other only over the federation
transport (I2P), and a client only ever talks to its own server's facade. So no
cross-server operation — delivery, contact request, or alias resolution — may
require a client to connect to, learn, or carry a foreign facade URL. The unit
of cross-server reachability is the recipient's **serving destination** (its
b33), carried in the user-signed **contact card** (`../Identity.md`,
`InviteAnonymity.md`) — distributed inside an invite, never a facade and no
longer a root-signed server card.

This holds in both directions on first contact: a contact request carries the
sender's own **routing** (sealing prekey + destination + serving sealing key) in
its bootstrap, so the recipient replies by federating to the sender's destination
over I2P, having learned only that destination — not a server fingerprint.

## Alias resolution

First contact with `alias@server` (or a main-server alias) needs the target
user's UID and contact card (prekey + destination + serving sealing key) — with
nothing leaked to intermediaries, and **without the client ever touching the
target server's facade**:

1. The client builds a query `{ alias }` sealed to the **target server's
   resolve key** (a server-level key — the one residual server-scoped artifact,
   since the client cannot yet seal to a per-destination key for a user whose
   destination it does not know), plus an ephemeral response key. Its own server
   sees only "user U queries server S", never the alias.
2. The client submits it via its own server (`resolve`, submit+poll like `send`); the own server relays it over I2P without reading it. The client never connects to the target server itself.
3. The target server unseals, asks its service node, and responds with the trust chain — **alias certificate + the user's contact card** (`../Identity.md`) — sealed to the ephemeral key, or `ALIAS_UNKNOWN`.
4. The requesting client verifies the signatures itself; intermediaries are trusted for availability only.

**Implementation status.** The facade-locality invariant is enforced on the
client. Today the facade-free first-contact path is the **invite**
(`bazarish://invite/…`, a single self-verifying contact card, no live lookup),
and same-server `@alias` resolves at the client's own service node. The sealed
federated `resolve` flow above (cross-server `@alias` over I2P) is the next
federation track and is **not yet implemented**.

## Authentication

- Federation accountability is **per-connecting-destination**: an inbound peer
  is rate-limited by the I2P destination it connects from (`FederationRateLimiter`,
  implemented). The server root key **never** signs live federation traffic —
  destinations are self-standing and the root only certifies offline; this keeps
  the connection origin from being a server-identifying signal.

## Open questions

- **Per-server rate limiting and spam policy** between federation peers.
- **Delegated federation keys** vs signing everything with the root key.
