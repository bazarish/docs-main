# Server <-> Service Node API

Status: draft skeleton.

Internal API over the operator private channel. The service node owns per-operator user-management state (accounts: the quota granted and the card published); the server owns content and delivery, and consults the service node to make admission decisions. The global **username** namespace is **not** here - it is a separate central service (`api/AliasResolver.md`), not the operator service node.

An account has **no term**. Registration opens an unbounded channel; the only clock is the delegated destination, and an account that has none for the node's idle window is deleted (`idleDeleteDays`, 180 by default).

## Server -> Service node (draft)

| Request | Purpose |
|---|---|
| `get-card` | fingerprint + the descriptor's `view` capability -> the user's **contact card** (prekey + destination + serving sealing key). Backs the federation `card` request - the first-contact card fetch (`api/FederatedResolve.md`). The capability must be the one the node issued for that account: a wrong one is refused **exactly** like an unknown user, so the lookup cannot be used to ask a server whether it hosts a fingerprint |

## Service node -> Server (draft)

| Event | Purpose |
|---|---|
| `account-deleted` | The node dropped user U (idle, closed by its owner, or removed by the operator); the server may purge all pending content for U |

## Client-ID registry

Owned by the **server** (it drives the delete-when-all-acked storage rule, see `../DeviceSync.md`); the service node never learns the user's device count.

## Open questions

- **Pull vs push**: does the server query account state per request (with caching), or does the service node push state changes? Affects failure modes when one daemon is down.
- **Quota usage accounting**: limits come from the account (service node), usage from stored content (server). Confirm the split: service node = limit, server = usage.
- Transport details (unix socket vs localhost HTTP) and whether this API needs authentication beyond channel isolation.
