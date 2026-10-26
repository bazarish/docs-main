# Bazarish Architecture

Status: draft skeleton.

Bazarish is a messenger whose architectural root is full resistance to protocol blocking and to deanonymization of servers and users. All communication is end-to-end encrypted. Any possibility of collecting metadata about a user and their contacts is a critical bug requiring architectural rework. Supported features: direct messages, calls, attachments of any type. Server-side implementation principle: minimum code, minimum dependencies, simple deployment on any server.

## Components

| Component | Form | Responsibility |
|---|---|---|
| **Server** | daemon (C++) | Core messaging: pending content storage, delivery, federation over hidden networks |
| **Service node** | separate daemon (C++) | User management: accounts, portal, moderation. Isolated from messaging logic |
| **Facade** | daemon (C++) | "Last mile" between client and server infrastructure. First implementation: HTTP over its own I2P destination (b33), answered at `/` |
| **Gateway** | optional daemon (C++) | Runs the I2P router for a client that does not run its own. An optional part of an operator's deployment: nothing else in it depends on a gateway, and a gateway needs nothing else from it |
| **Client** | separate repositories | Qt6 desktop app (`client-qt`); the same client core behind a REST API for bots (`bazarish-bot`) |
| **I2P router** | in-process or external | Transport for the client API, federation and calls. How a component hosts its router is an implementation choice, not part of the protocol |
| **Storage** | disk | Pending encrypted content |

## Topology

```
                                   operator-controlled private channel
                        +.........................................................+
[Client] -------I2P------> [Facade] --the sealed tunnel--> [Server] --> [Storage: disk]
    |                   |                                     |                   |
    |                   |          signed, user named in path  v                   |
    |                   |                              [Service node] --> [its own records]
    |                   +..........................................|..............+
    |                                                              |
    |                                     [embedded i2pd router] <==+== correspondents' clients (federation)
    |
    +--clearnet--> portal, reseed
```

The facade has exactly one backend. The service node is **not** reachable from it:
an account call travels in the same tunnel as everything else and the messaging
server is what forwards it.

- The server has **no public IP interfaces**. Its only public presence is in hidden networks (I2P first).
- Facade <-> Server / Service node communication goes over an **operator-controlled private channel**: TLS in which each side is named by the hash of the key in its certificate, written in the other's configuration, with no certificate authority and no hostname in the question. Splitting the fleet across hosts therefore needs no tunnel of its own; the operator may still place one there.
- **The facade carries one sealed tunnel and reads nothing.** A client's requests
  all arrive as opaque frames on a single path; the messaging server opens them
  and runs what was inside through its own routes. Account calls continue from there
  to the service node, forwarded under the server's own signature with the user
  named in the path it signed - so the node authorises them against the server
  card it is already bound to, without a shared secret or a new setting. The node
  is not reachable from the facade at all. See `api/Facade.md`, `api/ClientApi.md`.
- What the node gives up for that is the user's own signature on each account
  call: it now relies on the server's statement of whose call it is. What must be
  self-evident still is - the contact card is signed by its owner and the I2P
  transient carries an offline signature by the user's master key - so the
  artifacts that matter are self-verifying, and only the fact of the call is
  vouched for.

## Key decisions

1. **Repository scope** - the server fleet lives in one repository, `bazarish-server` (`server-core`, `service-node`, `facade-http`, `admin-panel` and the optional `gateway`, a host that runs the I2P router for a client that will not, sharing one build of `common`); separate from it are `client-qt` (client core + Qt6 GUI), `bazarish-bot` (the same core with a REST API and a web UI instead of a window), `common-cpp` (the shared C++ library, checked out as each repository's `common` submodule), `alias-resolver` (the alias registry, the only paid component) and `docs-main` (this repository - the canonical protocol specification all components implement). Repositories are developed side by side in a single workspace.
2. **Service node is a separate process** with its own API, keeping messaging logic and user management separated.
3. **Facade <-> server transport** - operator private channel, not a hidden network hop.
4. **Wire format** - JSON for control messages, raw HTTP bodies for binary content (no base64 overhead). E2E-encrypted payloads are opaque blobs to all server-side components.
5. **Destination routing** - delivery routes by the recipient's I2P **destination** (its b33), carried in the user-signed contact card, not by server fingerprint; the sender dials it directly (no resolver, no `peers.json`). A server has **no single federation address**: every user is served by their own destination, which admits what arrives and initiates no connections. The sending **client** performs the dialling, from a destination held for that one correspondent; a server that initiates no connections cannot derive a correspondence graph from the addresses it contacted. A contact is shared as a **descriptor** (`fp:b33`); an **alias** resolves at the central resolver. Every routing target is a `.b32.i2p` host. See `Identity.md`, `api/Federation.md`, `api/InviteAnonymity.md`, `api/FederatedResolve.md`.
6. **Single facade entry point** - the client uses one facade address, and behind it one path: the tunnel. The facade fans out to nothing, because there is one backend; the messaging server is what reaches the service node.
6a. **Routing travels with every message** - a message carries its sender's current destination and serving sealing key (`routing`), and a receiver adopts the pair whenever it differs. Regenerating an I2P key changes the address, and a contact who learned the old one at introduction would dial it forever; one message from the moved side repairs the return path, in either direction and of any type. There is no second recovery path: a request for someone's new address could only go to their old one. The rule is that a party which changes its address sends first. See `Messages.md`.
6b. **Tunnel length is a user setting** - three levels: minimal (1 hop each way, no variance), middle (1 hop, variance 1: I2P adds between zero and one hop and never subtracts), maximum (3 hops, no variance, the depth I2P itself defaults to). The client defaults to minimal and states what one hop does and does not conceal; call media remains minimal at every setting, three hops in each direction introducing audible delay in a live call. Server components accept `i2p.privacy` and default to maximum.
7. **Docs language** - English (code comments are English per codestyle).
8. **Shared code** - protocol types, OpenSSL wrappers, and JSON helpers live in the `common` repository, consumed by the other components via CMake (FetchContent or submodule). No duplicated crypto code.
9. **A client delivers its own outgoing messages; there is no server-side delivery path.** It dials the recipient's serving destination over I2P from a destination held for that correspondent; single-use destinations are used only for discovery lookups (card fetch, alias resolve), device pairing, file transfers and call media. Nothing is spooled on a server or on disk: the retry policy belongs to the sending client, and on exhausting it the client reports the failure to its user rather than continuing unattended. Repetition is safe at any time, the delivery id being derived from the message and the recipient mailbox. The recipient's server returns a signed confirmation in the reply on the same stream; a read receipt reports a read. See `api/ClientApi.md`, `api/Federation.md`, `Messages.md`.
10. **Sealed delivery envelopes.** The recipient identity, delivery pass and deliveryId are encrypted by the sending client to the **serving sealing key of the recipient's destination** (per-destination, held by the recipient's server). The sender's own server observes nothing, the envelope not passing through it; the recipient's server does not learn the sender's identity. **Federation signing is per-destination too**: each serving destination has its own signing key (the only key that signs live federation traffic, e.g. the confirmation in a deliver reply) stored bound to the destination, so no two users share a signer; the server root identity is reserved for non-federation use (the `bazarish://server` configuration link, the service-node binding). See `api/InviteAnonymity.md`, `api/Federation.md`.
11. **Contact admission via delivery passes** - one per correspondent, derived and revocable, so the server enforces "contacts only" without holding a readable social graph (see `Contacts.md`).
12. **Shared identity key across devices** - a new device receives the identity key from an existing one over a one-shot I2P destination, named by a pairing link (shown as a QR code) and unlocked by a four-digit code; per-device keys with cross-signing remain a possible future evolution (see `DeviceSync.md`).
13. **Explicit client registry** - client IDs are registered/retired via the API; the delete-when-all-acked rule operates on the registered set. The registry is owned by the server, not the service node.
14. **Device sync via the own mailbox** - E2E self-messages (contacts, settings, an echo of each message sent) written through a dedicated endpoint of their own and read back through the standard pending/ack path.
15. **The contact card is what answers** "where and how do I deliver?" - the user-signed sealing key, destination and serving key, published to the serving server and fetched by contacts. It names no server and carries no expiry; the account's lifecycle on that server is clocked by the delegation of its destination alone. Exactly one serving server at a time; migration is a hard cutover: mail in flight during it fails at the sender, who resends when the new routing has arrived (messages delayed, not lost). Registration is grant-gated through the service portal. See `Identity.md`, `api/ServicePortal.md`, `api/InviteAnonymity.md`.
16. **All certificates are one composite CBOR frame over a JSON body** (contact card, alias, server card) - both signatures and both public keys in one frame, consistent with the JSON wire format.
17. **Direct alias resolution** - first-contact lookups are dialled by the client itself from a transient I2P destination; the requester's own server is not on the path and never learns the alias. Nothing is sealed at the application layer, the I2P stream being encrypted to the destination dialled. The resolver's answer is a self-verifying chain (a record signed by a delegated key, its delegation certificate to the built-in root, and the owner's alias certificate) naming a descriptor, from which the client fetches the contact card. Cross-server resolution (central-resolver `resolve` + direct card fetch) is implemented and live-validated over real I2P, as is invite-based first contact, and the shipped client is built with the production resolver's coordinate (`defaultResolverCoordinate`). See `api/FederatedResolve.md`.
18. **The post-quantum hybrid covers sealing as well as signing (mandatory)** - signing is Ed25519 + ML-DSA-65, and every identity statement is dual-signed and valid only when both verify; the fingerprint covers both public keys. Sealing is X25519 + ML-KEM-768 and covers everything encrypted to a public key anywhere in the project, because it all goes through one `hybrid::seal`, whose key is derived from both shared secrets at once (`Identity.md`).

## Trust boundaries

- **Facade compromise must not damage server infrastructure.** The facade knows the private-channel address of its backend and the traffic passing through it - and that traffic is sealed end to end, so what it can learn is the flow (a stable source, timing, a padded length), never who or what. It holds no keys and no content at rest.
- **Server sees only ciphertext.** All message content is E2E-encrypted by clients; server-side components store and relay opaque blobs.
- **A user's own server receives; it does not send.** Its functions for its user are: accept deliveries on that user's destination, hold them until the user's devices have collected them, carry a write addressed to those devices, and register delivery passes. It is never given an outgoing message and never dials a correspondent, so the set of a user's correspondents is not available to it.
- **Metadata minimization is a core requirement.** Any new endpoint or stored field must be evaluated for what it reveals about users and their contact graph. Collectable metadata is a critical bug by definition.

## Repositories

One repository per component, developed side by side in a single workspace:

```
docs-main/                       # this repository - canonical protocol specification
|-- Architecture.md              # this file
|-- Identity.md                  # keys, UIDs, certificates, contact card
|-- Contacts.md                  # contact admission via delivery passes
|-- DeviceSync.md                # multi-device enrollment and E2E state sync
|-- Messages.md                  # two-layer message typing, content-type registry
|-- Bots.md                      # bots & interactive messages
|-- Errors.md                    # typed error registry
+-- api/
    |-- ClientApi.md             # client <-> facade <-> {server, service node}
    |-- Facade.md                # last-mile contract
    |-- Federation.md            # client -> serving destination over I2P (destination-routed)
    |-- InviteAnonymity.md       # destination-routed contact cards (invite anonymity)
    |-- FederatedResolve.md      # descriptor + card fetch, alias -> descriptor
    |-- SignInWithKey.md         # authentication by signature
    |-- ServicePortal.md         # sign-in-with-key portal, account onboarding, grants
    |-- ServerServiceNode.md     # internal server <-> service node API
    |-- FileTransfer.md          # direct client-to-client file transfer over I2P
    |-- Calls.md                 # call signaling and media tunnels
    |-- AdminPanel.md            # fleet status + management web panel
    +-- Storage.md               # storage backend interface

common-cpp/ (dir: common/)      # shared C++ library: protocol types, OpenSSL wrappers, embedded i2pd
bazarish-server/                 # the server fleet, one build tree:
    server-core/                 #   messaging server daemon (I2P federation, destination-routed)
    service-node/                #   service node daemon (accounts, portal, moderation)
    facade-http/                 #   HTTP facade daemon (I2P last mile)
    admin-panel/                 #   fleet status + management web panel (sign-in-with-key)
    gateway/                     #   optional: runs the I2P router for a client that will not
client-qt/                       # client core + Qt6 GUI
bazarish-bot/                    # the same core with no interface: a REST API for bots and a web UI
alias-resolver/                  # alias registry: signed resolve over I2P + portal (paid)
```

## Open questions

- **Threat model document** - to be written once the specification stabilises; `Contacts.md` and `DeviceSync.md` already record the residual metadata and the shared-identity-key risk.
