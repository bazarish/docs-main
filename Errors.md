# Error Model

Status: draft skeleton.

## Principles

- Every API-visible failure is a **typed error**: a stable machine-readable code plus a human-readable message. Clients branch on codes, never on messages.
- No error hiding (codestyle). Server-side, an unhandled exception is acceptable where no meaningful handling exists; anything that crosses an API boundary must be typed.
- Errors must not leak information that helps deanonymization (e.g., distinguishing "user unknown" from "not your contact" where it would reveal account existence - see open questions).

## Envelope

```json
{
  "error": {
    "code": "QUOTA_EXCEEDED",
    "message": "Blob size 12582912 exceeds the per-message limit of 10485760 bytes",
    "details": { "limit": 10485760, "size": 12582912 }
  }
}
```

HTTP transports map typed errors onto appropriate status codes; the JSON body is the source of truth.

## Initial registry

| Code | Meaning | Source |
|---|---|---|
| `QUOTA_EXCEEDED` | Blob is larger than the receiving server accepts; the error must explicitly explain the problem to the sender | server |
| `STORAGE_FULL` | User's pending-content quota is exhausted | server |
| `SUBSCRIPTION_EXPIRED` | User's subscription certificate is no longer valid | server / service node |
| `SUBSCRIPTION_TERM_TOO_LONG` | Requested certificate validity exceeds the server's maximum | service node |
| `RECIPIENT_SERVER_UNREACHABLE` | Internal/transient: the recipient's server could not be reached this round. After retries are exhausted the attempt becomes `unconfirmed` (the message stays **grey**, not a failure - it may have been stored while only the `delivered-ack` was lost), so this is **not** surfaced as a red failed state | server |
| `DELIVERY_TIMEOUT` | Delivery attempt did not complete in time; the attempt becomes `unconfirmed` (stays grey), resendable by the user with the same messageId | server |
| `ATTEMPT_UNKNOWN` | Polled send attempt is not known (state expired or server restarted); the client stops tracking it (the message stays grey and is resendable), not an auto-resubmit | server |
| `DELIVERY_REJECTED` | Delivery refused. Deliberately collapses "invalid/consumed token", "unknown mailbox", and "unsubscribed recipient" - externally indistinguishable (anti-enumeration, see `Contacts.md`). A tokenless contact request is the only alternative path | server |
| `CONTACT_REQUEST_TOO_LARGE` | Attached contact-request message exceeds the limit (default 8192 bytes, server-configurable) | server |
| `ALIAS_TAKEN` | Requested alias is already registered | service node |
| `ALIAS_UNKNOWN` | Alias resolution found no such alias (aliases are findable by design - explicit error, no anti-enumeration) | service node |
| `CLIENT_UNREGISTERED` | Request carries a client ID that is not registered for this user (see `DeviceSync.md`) | server |
| `ACCOUNT_PENDING_APPROVAL` | The account exists but is frozen: a moderated server has not approved it yet, so its destination is not brought up. The message carries the operator's instructions for finishing registration | service node |
| `I2P_UNAVAILABLE` | The embedded I2P router is not available; direct I2P (e.g. placing/accepting a call) is impossible | client-side |

The registry grows as APIs are specified; codes are append-only and never reused.

## Anti-enumeration policy

Detailed mailbox-state errors (`QUOTA_EXCEEDED`, `STORAGE_FULL`) are returned **only after a valid delivery token has been presented** - so only actual contacts can learn anything about a mailbox. Everything else (bad token, no such mailbox, expired subscription) collapses into `DELIVERY_REJECTED`.

## Open questions

- Error-code vs HTTP-status mapping table.
