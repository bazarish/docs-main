# Bazarish

Bazarish is a messenger whose architectural root is full resistance to protocol
blocking and to deanonymization of servers and users. All communication is
end-to-end encrypted. Any possibility of collecting metadata about a user and
their contacts is treated as a critical bug requiring architectural rework.
Supported features: direct messages, group chats, calls and attachments of any
type. The server-side principle is minimum code, minimum dependencies, simple
deployment on any server.

This repository is the **canonical protocol specification** that every
component implements.

## Specification

- [Architecture](Architecture.md) — components, topology, trust boundaries, key decisions
- [Identity](Identity.md) — keys, UIDs, certificates, server card, algorithms
- [Contacts](Contacts.md) — contact admission via one-time delivery tokens
- [DeviceSync](DeviceSync.md) — multi-device enrollment and E2E state sync
- [Groups](Groups.md) — group chats (direction fixed, design deferred)
- [Errors](Errors.md) — typed error registry
- API contracts:
  - [ClientApi](api/ClientApi.md) — client ↔ facade ↔ {server, service node}
  - [Facade](api/Facade.md) — the last-mile contract
  - [Federation](api/Federation.md) — server ↔ server over hidden networks
  - [ServerServiceNode](api/ServerServiceNode.md) — internal server ↔ service node API
  - [Storage](api/Storage.md) — pending-content storage backends
  - [Calls](api/Calls.md) — audio/video calls

## Components

One repository per component, developed side by side:

| Component | Repository |
|---|---|
| Shared C++ library | [common-cpp](https://github.com/bazarish/common-cpp) |
| Messaging server | [server-core](https://github.com/bazarish/server-core) |
| Service node (user management) | [service-node](https://github.com/bazarish/service-node) |
| HTTP facade (last mile) | [facade-http](https://github.com/bazarish/facade-http) |
| Client (core, CLI, Qt6 GUI) | [client-qt6](https://github.com/bazarish/client-qt6) |
| Specification | this repository |

## Cryptography

Identity is a mandatory post-quantum hybrid (ECDSA P-256 + ML-DSA-65); every
identity statement is dual-signed. Containers are OpenSSL-native standards
(PEM, DER, CMS). No custom cryptographic constructions.
