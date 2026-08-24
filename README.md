# Bazarish

Bazarish is a messenger whose architectural root is full resistance to protocol
blocking and to deanonymization of servers and users. All communication is
end-to-end encrypted. Any possibility of collecting metadata about a user and
their contacts is treated as a critical bug requiring architectural rework.
Supported features: direct (1-1) messages, audio calls and attachments of any
type. The server-side principle is minimum code, minimum dependencies, simple
deployment on any server.

This repository is the **canonical protocol specification** that every
component implements.

## Specification

- [Architecture](Architecture.md) - components, topology, trust boundaries, key decisions
- [Identity](Identity.md) - keys, UIDs, certificates, contact card, algorithms
- [Contacts](Contacts.md) - contact admission via one-time delivery tokens
- [Messages](Messages.md) - delivery class vs content type; the content-type registry
- [Bots](Bots.md) - bots as ordinary identities; commands, inline keyboards, callbacks
- [DeviceSync](DeviceSync.md) - multi-device enrollment and E2E state sync
- [Errors](Errors.md) - typed error registry
- API contracts:
  - [ClientApi](api/ClientApi.md) - client <-> facade <-> {server, service node}
  - [Facade](api/Facade.md) - the last-mile contract
  - [Federation](api/Federation.md) - server <-> server over I2P (destination-routed)
  - [InviteAnonymity](api/InviteAnonymity.md) - destination-routed contact cards
  - [ServicePortal](api/ServicePortal.md) - sign-in-with-key portal, payments, grants
  - [ServerServiceNode](api/ServerServiceNode.md) - internal server <-> service node API
  - [FileTransfer](api/FileTransfer.md) - direct client-to-client file transfer over I2P
  - [Storage](api/Storage.md) - pending-content storage backends
  - [Calls](api/Calls.md) - audio/video calls
  - [AdminPanel](api/AdminPanel.md) - fleet status + management web panel (sign-in-with-key)

## Components

One repository per component, developed side by side:

| Component | Repository |
|---|---|
| Shared C++ library | [common-cpp](https://github.com/bazarish/common-cpp) |
| Server fleet: messaging server, service node, HTTP facade, admin panel | [bazarish-server](https://github.com/bazarish/bazarish-server) |
| Client (core, CLI, Qt6 GUI) | [client-qt6](https://github.com/bazarish/client-qt6) |
| Username registry (signed resolve + portal; the one paid component) | [alias-resolver](https://github.com/bazarish/alias-resolver) |
| Specification | this repository |

## Cryptography

Identity is a mandatory post-quantum hybrid (ECDSA P-256 + ML-DSA-65); every
identity statement is dual-signed. Containers are OpenSSL-native standards
(PEM, DER, CMS). No custom cryptographic constructions.

## Status and near-term plan

The protocol is implemented across all components, with unit tests green and the
core flows live-validated over real I2P: E2EE first contact, destination-routed
federation, the central alias resolver (claim -> resolve -> card fetch ->
conversation) including its offline-key serving-dest hot-swap, large-file blob
transfer, bots and interactive messages,
message editing and read receipts, multiple accounts, at-rest encryption of all
local state, and client-to-client audio and video calls (Opus and VP8 media over
I2P RAW datagrams to b33 destinations, per-call AES-256-GCM, minimal-length media
tunnels, strict I2P requirement; the embedded `libi2pd_bazarish` carries the
datagram-to-b33 routing in-process).

Near-term work (not yet done):

- **Incoming QR scan** - showing/sharing a QR works; decoding a scanned QR on the
  client is not yet wired (paste-link is the fallback).
- **Service-portal inline QR** - the pay page is JS-free with a refresh fallback;
  an inline payment QR is still planned (`api/ServicePortal.md`).
- **Resolver deployment** - baking the resolver coordinate into the shipped client
  (`defaultResolverCoordinate`) is an ops step once a resolver is deployed; the
  seamless offline-key hot-swap requires i2pd >= 2.61.0.

The `admin-panel` provides one web interface to monitor and manage the fleet:
every service exposes a signed `GET /healthz` (name, version, uptime, current and
peak RAM) and the panel drives the operator endpoints (per-user quota, per-user
I2P destination). Operator-internal calls are authenticated by hybrid signature -
the caller signs the request with its identity and the service authorises that
fingerprint - so there is no shared operator secret. Panel sign-in is
sign-in-with-key, verified locally against the configured admin identity with no
backend online (`api/AdminPanel.md`).

Out of scope for v1 (tracked as future concepts): per-device identity keys
(`DeviceSync.md`), and a
standalone threat-model document.
