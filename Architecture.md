# Bazarish Architecture

Status: draft skeleton.

Bazarish is a messenger whose architectural root is full resistance to protocol blocking and to deanonymization of servers and users. All communication is end-to-end encrypted. Any possibility of collecting metadata about a user and their contacts is a critical bug requiring architectural rework. Supported features: direct messages, calls, attachments of any type. Server-side implementation principle: minimum code, minimum dependencies, simple deployment on any server.

## Components

| Component | Form | Responsibility |
|---|---|---|
| **Server** | daemon (C++) | Core messaging: pending content storage, delivery, federation over hidden networks |
| **Service node** | separate daemon (C++) | User management: accounts, aliases. Isolated from messaging logic |
| **Facade** | daemon (C++) | "Last mile" between client and server infrastructure. First implementation: HTTP REST behind a reverse proxy at a secret URI |
| **Client** | separate repository | Qt6 desktop app (and future bots) |
| **I2P router** | in-process or external | Transport for federation and calls. How a component hosts its router is an implementation choice, not part of the protocol |
| **Storage** | disk or S3-compatible object storage | Pending encrypted content |

## Topology

```
                          clearnet          operator-controlled private channel
                             |             +...................................+
[Client] --HTTPS, secret URI---> [Reverse proxy] -> [Facade] ----> [Server] ------> [Storage: disk / object]
                             |             |                   |       |       |
                             |             |                   +-> [Service    |
                             |             |                        node]      |
                             |             +.......................|..........+
                             |                                      |
                             |                          [embedded i2pd router] <======> other servers (federation)
```

- The server has **no public IP interfaces**. Its only public presence is in hidden networks (I2P first).
- Facade <-> Server / Service node communication goes over an **operator-controlled private channel** (unix socket, localhost, or VPN - deployment choice). Splitting facade and server across hosts is an operator concern.
- **The facade carries one sealed tunnel and reads nothing.** A client's requests
  all arrive as opaque frames on a single path; the messaging server opens them
  and runs what was inside through its own routes. Account calls go on from there
  to the service node, forwarded under the server's own signature with the user
  named in the path it signed - so the node authorises them against the server
  card it is already bound to, without a shared secret or a new setting. The node
  is not reachable from the facade at all. See `api/Facade.md`, `api/ClientApi.md`.
- What the node gives up for that is the user's own signature on each account
  call: it now has the server's word for whose call it is. What must be
  self-evident still is - the contact card is signed by its owner and the I2P
  transient carries an offline signature by the user's master key - so the
  artifacts that matter prove themselves, and only the fact of the call is
  vouched for.

## Key decisions

1. **Repository scope** - the server fleet lives in one repository, `bazarish-server` (`server-core`, `service-node`, `facade-http`, `admin-panel`, sharing one build of `common`); separate from it are `client-qt6` (client core + CLI + GUI), `common-cpp` (the shared C++ library, checked out as each repository's `common` submodule), `alias-resolver` (the username registry, the only paid component) and `docs-main` (this repository - the canonical protocol specification all components implement). Repositories are developed side by side in a single workspace.
2. **Service node is a separate process** with its own API, keeping messaging logic and user management separated.
3. **Facade <-> server transport** - operator private channel, not a hidden network hop.
4. **Wire format** - JSON for control messages, raw HTTP bodies for binary content (no base64 overhead). E2E-encrypted payloads are opaque blobs to all server-side components.
5. **Destination routing** - delivery routes by the recipient's I2P **destination** (its b33), carried in the user-signed contact card, not by server fingerprint; the sender dials it directly (no resolver, no `peers.json`). A server has **no single federation address**: every user is served by their own destination, which admits what arrives and initiates no connections. The sending **client** performs the dialling, from a destination held for that one correspondent; a server that initiates no connections cannot derive a correspondence graph from the addresses it contacted. A contact is shared as a **descriptor** (`fp:b33`); a **username** resolves at the central resolver. Every routing target is a `.b32.i2p` host. See `Identity.md`, `api/Federation.md`, `api/InviteAnonymity.md`, `api/FederatedResolve.md`, `api/AliasResolver.md`.
6. **Single facade entry point** - the client uses one secret URI, and behind it one path: the tunnel. The facade fans out to nothing, because there is one backend; the messaging server is what reaches the service node.
6a. **Routing travels with every message** - a message carries its sender's current destination and serving sealing key (`routing`), and a receiver adopts the pair whenever it differs. Regenerating an I2P key changes the address, and a contact who learned the old one at introduction would dial it forever; one message from the moved side repairs the way back, in either direction and of any type. There is no second recovery path: a request for someone's new address could only go to their old one. The rule is that a party which changes its address sends first. See `Messages.md`.
6b. **Tunnel length is a user setting** - three levels: minimal (1 hop each way, no variance), middle (1 hop, variance 1: I2P adds between zero and one hop and never subtracts), maximum (3 hops, no variance, the depth I2P itself defaults to). The client defaults to minimal and states what one hop does and does not conceal; call media remains minimal at every setting, three hops in each direction introducing audible delay in a live call. Server components accept `--privacy` and default to maximum.
7. **Docs language** - English (code comments are English per codestyle).
8. **Shared code** - protocol types, OpenSSL wrappers, and JSON helpers live in the `common` repository, consumed by the other components via CMake (FetchContent or submodule). No duplicated crypto code.
9. **A client delivers its own outgoing messages; there is no server-side delivery path.** It dials the recipient's serving destination over I2P from a destination held for that correspondent; single-use destinations are reserved for file transfers and call media. Nothing is spooled on a server or on disk: the retry policy belongs to the sending client, and on exhausting it the client reports the failure to its user rather than continuing unattended. Repetition is safe at any time, the delivery id being derived from the message and the recipient mailbox. The recipient's server returns a signed confirmation in the reply on the same stream; a read receipt reports a read. See `api/ClientApi.md`, `api/Federation.md`, `Messages.md`.
10. **Sealed delivery envelopes.** The recipient identity, delivery token and deliveryId are encrypted by the sending client to the **serving sealing key of the recipient's destination** (per-destination, held by the recipient's server). The sender's own server observes nothing, the envelope not passing through it; the recipient's server does not learn the sender's identity. **Federation signing is per-destination too**: each serving destination has its own signing key (the only key that signs live federation traffic, e.g. the confirmation in a deliver reply) stored bound to the destination, so no two users share a signer; the server root identity is reserved for non-federation use (the `bazarish://server` configuration link, the service-node binding). See `api/InviteAnonymity.md`, `api/Federation.md`.
11. **Contact admission via one-time unlinkable delivery tokens** - the server enforces "contacts only" without holding a readable social graph (see `Contacts.md`).
12. **Shared identity key across devices** - new devices are enrolled by QR transfer of the identity key; per-device keys with cross-signing remain a possible future evolution (see `DeviceSync.md`).
13. **Explicit client registry** - client IDs are registered/retired via the API; the delete-when-all-acked rule operates on the registered set. The registry is owned by the server, not the service node.
14. **Device sync via the own mailbox** - an E2E event journal (contacts, token bookkeeping, full message-history echo) delivered through the standard send/pending/ack path; no dedicated server mechanism.
15. **The contact card is what answers** "where and how do I deliver?" - the user-signed sealing key, destination and serving key, published to the serving server and fetched by contacts. It also anchors the account's lifecycle on that server. It answers "where and how do I deliver?". Exactly one serving server at a time; migration is a hard cutover: mail in flight during it fails at the sender, who resends when the new routing has arrived (messages delayed, not lost). Registration is grant-gated through the service portal. See `Identity.md`, `api/ServicePortal.md`, `api/InviteAnonymity.md`.
16. **All certificates are CMS SignedData over JSON bodies** (contact card, alias, server card) - OpenSSL-native standard container, consistent with the JSON wire format.
17. **Sealed alias resolution** - first-contact lookups are encrypted to the target server's resolve key and dialled by the client itself; the requester's own server is not on the path and never learns the alias. The response is a self-verifying chain (alias cert + the user's contact card). Cross-server resolution (central-resolver `resolve` + sealed card fetch) is implemented and live-validated over real I2P, as is invite-based first contact; only the production resolver deployment (baking the resolver coordinate into the client) remains.
18. **The post-quantum hybrid covers sealing as well as signing (mandatory)** - signing is ECDSA P-256 + ML-DSA-65, and every identity statement is dual-signed and valid only when both verify; the fingerprint covers both public keys. Sealing is ECDH P-256 + ML-KEM-768 and covers everything encrypted anywhere in the project, because it all goes through one `cms::seal` - layered over CMS since OpenSSL has no KEMRecipientInfo yet (`Identity.md`).

## Trust boundaries

- **Facade compromise must not damage server infrastructure.** The facade knows the private-channel address of its backend and the traffic passing through it - and that traffic is sealed end to end, so what it can learn is the flow (a stable source, timing, a padded length), never who or what. It holds no keys and no content at rest.
- **Server sees only ciphertext.** All message content is E2E-encrypted by clients; server-side components store and relay opaque blobs.
- **A user's own server receives; it does not send.** Its functions for its user are: accept deliveries on that user's destination, hold them until the user's devices have collected them, carry a write addressed to those devices, and register delivery tokens. It is never given an outgoing message and never dials a correspondent, so the set of a user's correspondents is not available to it.
- **Metadata minimization is a core requirement.** Any new endpoint or stored field must be evaluated for what it reveals about users and their contact graph. Collectable metadata is a critical bug by definition.

## Repositories

One repository per component, developed side by side in a single workspace:

```
docs-main/                       # this repository - canonical protocol specification
|-- Architecture.md              # this file
|-- Identity.md                  # keys, UIDs, certificates, contact card
|-- Contacts.md                  # contact admission via one-time delivery tokens
|-- DeviceSync.md                # multi-device enrollment and E2E state sync
|-- Messages.md                  # two-layer message typing, content-type registry
|-- Bots.md                      # bots & interactive messages
|-- Errors.md                    # typed error registry
+-- api/
    |-- ClientApi.md             # client <-> facade <-> {server, service node}
    |-- Facade.md                # last-mile contract
    |-- Federation.md            # server <-> server over I2P (destination-routed)
    |-- InviteAnonymity.md       # destination-routed contact cards (invite anonymity)
    |-- ServicePortal.md         # sign-in-with-key portal, payments, grants
    |-- ServerServiceNode.md     # internal server <-> service node API
    |-- FileTransfer.md          # direct client-to-client file transfer over I2P
    |-- Calls.md                 # call signaling and media tunnels
    +-- Storage.md               # storage backend interface

common-cpp/ (dir: common/)      # shared C++ library: protocol types, OpenSSL wrappers, embedded i2pd
bazarish-server/                 # the server fleet, one build tree:
    server-core/                 #   messaging server daemon (I2P federation, destination-routed)
    service-node/                #   service node daemon (accounts, portal, moderation)
    facade-http/                 #   HTTP facade daemon (secret-path last mile)
    admin-panel/                 #   fleet status + management web panel (sign-in-with-key)
client-qt6/                      # client core + CLI + Qt6 GUI
alias-resolver/                  # username registry: signed resolve over I2P + portal (paid)
```

## Open questions

- **Threat model document** - to be written once the specification stabilises; `Contacts.md` and `DeviceSync.md` already record the residual metadata and the shared-identity-key risk.
