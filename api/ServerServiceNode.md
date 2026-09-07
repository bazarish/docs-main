# Server <-> Service Node API

Status: draft skeleton.

Internal API over the operator private channel. The service node owns per-operator user-management state (accounts: the quota granted and the card published); the server owns content and delivery, and consults the service node to make admission decisions. The global **username** namespace is **not** here - it is a separate central service (`api/AliasResolver.md`), not the operator service node.

An account has **no term**. Registration opens an unbounded channel; the only clock is the delegated destination, and an account that has none for the node's idle window is deleted (`idleDeleteDays`, 180 by default).

## Server -> Service node (draft)

| Request | Purpose |
|---|---|
| `get-card` | fingerprint + the descriptor's `view` capability -> the user's **contact card** (sealing key + destination + serving sealing key). Backs the federation `card` request - the first-contact card fetch (`api/FederatedResolve.md`). The capability must be the one the node issued for that account: a wrong one is refused **exactly** like an unknown user, so the lookup cannot be used to ask a server whether it hosts a fingerprint |

## Service node -> Server (implemented)

Requests are signed with the node's own identity and authorised by the server on
that fingerprint; no shared secret is used.

| Request | Purpose |
|---|---|
| `PUT /v1/messaging/i2p-dest` | Supply the user's signed offline transient; an empty transient revokes the destination |
| `PUT /v1/messaging/quota` | Set the mailbox quota granted by the account, in place of the server's default |
| `PUT /v1/messaging/serving-key` | Rotate a destination's serving sealing key in two phases: prepare, then commit |
| `GET /v1/messaging/account?user=` | State held for one account: registered devices, bytes in use, effective quota; displayed by the administration panel |
| `DELETE /v1/messaging/account?user=` | The account has been deleted: discard the mailbox, the replay markers, the delivery tokens, the device registry and the quota override. Issued whenever an account is deleted, by its owner or by an operator. Idempotent: a fingerprint for which the server holds no state is already in the requested condition |

## Client-ID registry

Owned by the **server** (it drives the delete-when-all-acked storage rule, see `../DeviceSync.md`); the service node never learns the user's device count.

## Open questions

- **Pull vs push**: does the server query account state per request (with caching), or does the service node push state changes? Affects failure modes when one daemon is down.
- **Quota usage accounting**: limits come from the account (service node), usage from stored content (server). Confirm the split: service node = limit, server = usage.
- Transport details (unix socket vs localhost HTTP) and whether this API needs authentication beyond channel isolation.
