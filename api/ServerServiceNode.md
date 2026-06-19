# Server <-> Service Node API

Status: draft skeleton.

Internal API over the operator private channel. The service node owns per-operator user-management state (subscriptions); the server owns content and delivery, and consults the service node to make admission decisions. The global **username** namespace is **not** here - it is a separate central service (`api/AliasResolver.md`), not the operator service node.

## Server -> Service node (draft)

| Request | Purpose |
|---|---|
| `subscription-status` | Is user U actively subscribed? Returns expiry and quota (MB) |
| `get-card` | fingerprint -> the user's subscription certificate (its **contact card**: prekey + destination + serving sealing key). Backs the federation `card` request - the first-contact card fetch (`api/FederatedResolve.md`). Realized as an account-API lookup by fingerprint |

## Service node -> Server (draft)

| Event | Purpose |
|---|---|
| `subscription-expired` | Subscription for user U lapsed; the server may purge all pending content for U |

## Client-ID registry

Owned by the **server** (it drives the delete-when-all-acked storage rule, see `../DeviceSync.md`); the service node never learns the user's device count.

## Open questions

- **Pull vs push**: does the server query subscription status per request (with caching), or does the service node push state changes? Affects failure modes when one daemon is down.
- **Quota usage accounting**: limits come from the subscription (service node), usage from stored content (server). Confirm the split: service node = limit, server = usage.
- Transport details (unix socket vs localhost HTTP) and whether this API needs authentication beyond channel isolation.
