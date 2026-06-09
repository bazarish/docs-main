# Bazarish Architecture

Status: draft skeleton.

Bazarish is a messenger whose architectural root is full resistance to protocol blocking and to deanonymization of servers and users. All communication is end-to-end encrypted. Any possibility of collecting metadata about a user and their contacts is a critical bug requiring architectural rework. Supported features: direct messages, group chats, calls, attachments of any type. Server-side implementation principle: minimum code, minimum dependencies, simple deployment on any server.

## Components

| Component | Form | Responsibility |
|---|---|---|
| **Server** | daemon (C++) | Core messaging: pending content storage, delivery, federation over hidden networks |
| **Service node** | separate daemon (C++) | User management: subscriptions, aliases. Isolated from messaging logic |
| **Facade** | daemon (C++) | "Last mile" between client and server infrastructure. First implementation: HTTP REST behind a reverse proxy at a secret URI |
| **Client** | separate repository | Qt6 desktop app (and future bots) |
| **I2P router** | external dependency | Accessed via SAM API; transport for federation and calls |
| **Storage** | disk or S3-compatible object storage | Pending encrypted content |

## Topology

```
                          clearnet          operator-controlled private channel
                             │             ┌···································┐
[Client] ──HTTPS, secret URI─┼─▶ [Reverse proxy] ─▶ [Facade] ──┬─▶ [Server] ───┼──▶ [Storage: disk / object]
                             │             │                   │       │       │
                             │             │                   └─▶ [Service    │
                             │             │                        node]      │
                             │             └·······················│··········┘
                             │                                  [SAM API]
                             │                                      │
                             │                                 [I2P router] ◀══════▶ other servers (federation)
```

- The server has **no public IP interfaces**. Its only public presence is in hidden networks (I2P first).
- Facade ↔ Server / Service node communication goes over an **operator-controlled private channel** (unix socket, localhost, or VPN — deployment choice). Splitting facade and server across hosts is an operator concern.
- The facade routes client requests by path prefix: messaging → server, account management → service node.

## Key decisions

1. **Repository scope** — one repository per component: `server-core`, `service-node`, `facade-http`, `client-qt6`, `common` (shared C++ library), and `docs-main` (this repository — the canonical protocol specification all components implement). Repositories are developed side by side in a single workspace.
2. **Service node is a separate process** with its own API, keeping messaging logic and user management separated.
3. **Facade ↔ server transport** — operator private channel, not a hidden network hop.
4. **Wire format** — JSON for control messages, raw HTTP bodies for binary content (no base64 overhead). E2E-encrypted payloads are opaque blobs to all server-side components.
5. **Server addressing** — `@server` is the fingerprint of the server's root key. A signed **server card** maps the fingerprint to transport endpoints (see `Identity.md`). Identity is not bound to any single transport, leaving room for new networks.
6. **Single facade entry point** — the client uses one secret URI; the facade fans out to server and service node.
7. **Docs language** — English (code comments are English per codestyle).
8. **Shared code** — protocol types, OpenSSL wrappers, and JSON helpers live in the `common` repository, consumed by the other components via CMake (FetchContent or submodule). No duplicated crypto code.
9. **Sending is submit + poll, with no server-side outbound persistence.** The sender's server relays in memory only and never queues outbound messages; undeliverable messages stay on the client (see `api/ClientApi.md`, `api/Federation.md`).
10. **Sealed delivery envelopes.** The recipient identity, delivery token, and messageId are encrypted by the sending client to the destination server's sealing key. The sender's server sees only the destination server; the recipient's server never sees the sender identity.
11. **Contact admission via one-time unlinkable delivery tokens** — the server enforces "contacts only" without holding a readable social graph (see `Contacts.md`).
12. **Shared identity key across devices** — new devices are enrolled by QR transfer of the identity key; per-device keys with cross-signing remain a possible future evolution (see `DeviceSync.md`).
13. **Explicit client registry** — client IDs are registered/retired via the API; the delete-when-all-acked rule operates on the registered set. The registry is owned by the server, not the service node.
14. **Device sync via the own mailbox** — an E2E event journal (contacts, token bookkeeping, full message-history echo) delivered through the standard send/pending/ack path; no dedicated server mechanism.
15. **Subscription certificate is dual-purpose** — the lifecycle anchor on the serving server and, pushed to contacts over E2E, the *serving statement* that answers "where do I deliver?". Exactly one serving server at a time; migration is a hard cutover that leans on retry-on-client send semantics (messages delayed, not lost). See `Identity.md`.
16. **All certificates are CMS SignedData over JSON bodies** (subscription, alias, server card) — OpenSSL-native standard container, consistent with the JSON wire format.
17. **Sealed alias resolution** — first-contact lookups are encrypted to the target server's sealing key and relayed blindly; the requester's own server never learns the alias. The response is a self-verifying trust chain (alias cert + subscription cert + server card).
18. **Identity is a post-quantum hybrid (mandatory)** — ECDSA P-256 + ML-DSA-65; every identity statement is dual-signed and valid only when both signatures verify. The fingerprint covers both public keys. Sealing remains classical for now (OpenSSL CMS lacks ML-KEM) — recorded as an accepted, revisitable gap in `Identity.md`.

## Trust boundaries

- **Facade compromise must not damage server infrastructure.** The facade knows only the private-channel addresses of its backends and the traffic passing through it. It holds no keys and no content at rest.
- **Server sees only ciphertext.** All message content is E2E-encrypted by clients; server-side components store and relay opaque blobs.
- **Metadata minimization is a core requirement.** Any new endpoint or stored field must be evaluated for what it reveals about users and their contact graph. Collectable metadata is a critical bug by definition.

## Repositories

One repository per component, developed side by side in a single workspace:

```
docs-main/                       # this repository — canonical protocol specification
├── Architecture.md              # this file
├── Identity.md                  # keys, UIDs, certificates, server card
├── Contacts.md                  # contact admission via one-time delivery tokens
├── DeviceSync.md                # multi-device enrollment and E2E state sync
├── Groups.md                    # group chats: deferred, direction fixed
├── Errors.md                    # typed error registry
└── api/
    ├── ClientApi.md             # client ↔ facade ↔ {server, service node}
    ├── Facade.md                # last-mile contract
    ├── Federation.md            # server ↔ server over I2P
    ├── ServerServiceNode.md     # internal server ↔ service node API
    ├── Calls.md                 # call signaling and media tunnels
    └── Storage.md               # storage backend interface

common/                          # shared C++ library: protocol types, OpenSSL wrappers, JSON helpers
server-core/                     # messaging server daemon
service-node/                    # service node daemon
facade-http/                     # HTTP facade daemon
client-qt6/                      # Qt6 desktop client
```

## Open questions

- **Group chats** — deferred until direct messaging works end to end; the architectural direction (client-side fan-out, group token pools, multi-admin roster, ≤ ~100 members) is fixed in `Groups.md` and v1 must not contradict it.
- **Threat model document** — to be written once the skeleton stabilizes; `Contacts.md` and `DeviceSync.md` already carry honest accounting of residual metadata and the shared-identity-key risk.
