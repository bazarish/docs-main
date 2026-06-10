# Federation API (Server ↔ Server)

Status: draft skeleton.

## Transport

- First iteration: I2P. Each server runs an I2P **STREAM** session via the SAM v3 bridge and publishes its destination in the server card (see `../Identity.md`). The destination is persisted (`<state>/i2p-dest.key`) so the address is stable across restarts.
- The architecture must allow additional hidden/overlay transports later; nothing in the protocol may assume I2P specifics outside the transport layer. The delivery framing operates on an abstract stream, so a new transport only supplies a connected byte stream.
- Wire format matches the rest of the system: JSON control messages + raw binary bodies.

### Delivery framing (implemented)

Over the connected stream, one delivery is:

```
request:  {"op":"deliver","sealed":<base64>,"len":<N>}\n   then N raw payload bytes
response: {"delivered":<bool>,"errorCode":...,"errorMessage":...}\n
```

Length-prefixed both ways — no half-close (SAM propagates a SHUT_WR as a full stream teardown). The recipient runs the same delivery pipeline as a local send (dedup → token → size/quota → store), so local and federated deliveries share one code path.

Operational note: a freshly created destination must publish its leaseset before peers can reach it; the first connect after startup may return `CANT_REACH_PEER` ("LeaseSet not found") and should be retried with backoff. Verified end to end against a live i2pd (`i2p-federation-smoke`).

## Server discovery

1. The sender's server takes the `@server` fingerprint from the recipient address (the main server is implied when the part is absent).
2. It resolves the fingerprint to transport endpoints via the recipient's **server card** (obtained from contact-exchange payloads, previous deliveries, or — TBD — a card-fetch request).
3. It verifies the card signature against the fingerprint before connecting.

## Delivery envelope

Built entirely by the sending **client**; servers never see more than their layer:

```
DeliveryEnvelope:
  toServer: <server fingerprint>            # routing only — all the sender's server sees
  sealed:                                   # CMS envelope, encrypted to the recipient
                                            # server's sealing key (from its server card)
    class: content | contact                # delivery class (admission selector), not the
                                            # content type, which is E2E — see ../Messages.md
    mailbox: <recipient UID>
    token: <one-time delivery token>        # absent for class "contact"
    messageId: <client-generated random id> # idempotency / dedup
  payload: <E2E blob>                       # opaque to every server-side component
```

- The sender's own server learns only "user U sent something to server S" (plus size and time) — never the recipient identity.
- The recipient's server learns only "something arrived from server X into mailbox V with a valid token" — never the sender identity (see `../Contacts.md`).

## Message delivery flow

1. Sender's client → own server: `send` with the envelope. The server accepts it **in memory only** (no outbound persistence, no outbound queue) and returns an attempt id; the client polls for the outcome (see `ClientApi.md`).
2. Sender's server → recipient's server over I2P: `deliver` with the envelope.
3. Recipient's server unseals and processes in strict order:
   1. **messageId dedup** — if already seen within the retention window, respond success *without* consuming a token (makes client retries idempotent);
   2. **token check** — hash the presented token, look it up in the mailbox's registered set (`../Contacts.md`); no match → `DELIVERY_REJECTED`;
   3. **size/quota checks** — explicit typed errors (`QUOTA_EXCEEDED`, `STORAGE_FULL`), revealed only past a valid token;
   4. **store** as pending content; the token is consumed atomically with the store.
4. The outcome propagates back to the sender's client via polling. On failure the message stays **only on the client**, which may retry with the same messageId; the sender's server drops its in-memory copy as soon as the attempt concludes.

`contact-request` deliveries skip the token check, are capped in payload size (default 256 bytes, server-configurable), and are subject to a separate quota and per-origin-server rate limiting.

## Requests (draft)

| Request | Purpose |
|---|---|
| `deliver` | Deliver an envelope (message or contact request) for a local mailbox |
| `resolve` | Sealed alias-resolution query (see below) |
| `card` | Fetch the current server card (TBD whether exposed here) |

## Alias resolution

First contact with `alias@server` (or a main-server alias) needs the target user's UID, serving statement, and server card — with nothing leaked to intermediaries:

1. The client builds a query `{ alias }` sealed to the **target** server's sealing key, plus an ephemeral response key. Its own server sees only "user U queries server S", never the alias.
2. The client submits it via its own server (`resolve`, submit+poll like `send`); the own server relays it over I2P without reading it.
3. The target server unseals, asks its service node, and responds with the trust chain — **alias certificate + subscription certificate + server card** (see `../Identity.md`) — sealed to the ephemeral key, or `ALIAS_UNKNOWN`.
4. The requesting client verifies all three signatures itself; intermediaries are trusted for availability only.

## Authentication

- Requests are signed by the sending server's root key (or a delegated federation key — TBD), so the receiving side can attribute traffic to a server identity for rate limiting and abuse control.

## Open questions

- **Per-server rate limiting and spam policy** between federation peers.
- **Delegated federation keys** vs signing everything with the root key.
