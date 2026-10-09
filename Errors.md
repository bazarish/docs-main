# Error Model

Status: draft skeleton.

## Principles

- Every API-visible failure is a **typed error**: a stable machine-readable code plus a human-readable message. Clients branch on codes, never on messages.
- No error hiding (codestyle). Server-side, an unhandled exception is acceptable where no meaningful handling exists; anything that crosses an API boundary must be typed.
- Errors must not disclose information that assists deanonymisation. For example, distinguishing "user unknown" from "not a contact of the recipient" would reveal the existence of an account (see *Open questions*).

## Envelope

```json
{
  "error": {
    "code": "MESSAGE_TOO_LARGE",
    "message": "a message may carry at most 524288 bytes"
  }
}
```

An optional `details` object may carry structured values beside the message; no component sets it at present.

HTTP transports map typed errors onto appropriate status codes; the JSON body is the source of truth.

## Initial registry

| Code | Meaning | Source |
|---|---|---|
| `STORAGE_FULL` | User's pending-content quota is exhausted | server |
| `QUOTA_EXCEEDED` | Defined, but no component emits it: an exhausted quota is reported as `STORAGE_FULL` | none |
| `RECIPIENT_SERVER_UNREACHABLE` | The recipient's server returned no answer: it could not be dialled, the stream failed, or it asserted a delivery without a verifiable signature. Not a refusal; the sending client may repeat the attempt, and on exhausting its attempts reports this as the reason (`Messages.md`) | client |
| `MESSAGE_TOO_LARGE` | The payload is over `kMaxMessagePayloadBytes`. Past admission, so only a correspondent ever sees it | server |
| `CONTACT_RATE_LIMITED` | This destination is taking too many contact requests, or has too many waiting. Transient; says nothing about who is served there | server |
| `SESSION_INVALID` | The session a request authenticated with no longer exists. Answered by opening a new tunnel, never by retrying the same way | server |
| `FACADE_UPSTREAM` | The facade could not reach the server behind it. The only error a facade generates itself | facade |
| `DELIVERY_REJECTED` | Delivery refused. Deliberately combines "unknown pass", "unknown mailbox" and "recipient not served here" into one externally indistinguishable result (anti-enumeration; see `Contacts.md`). A contact request, which presents nothing, is the only alternative admission path | server |
| `CONTACT_REQUEST_TOO_LARGE` | Contact request exceeds `kMaxContactRequestBytes` (17590, a protocol constant - not an operator setting) | server |
| `ALIAS_TAKEN` | Requested alias is already registered. Defined, but no component emits it: the alias registry reports a held name on its own portal page, not with this code | none |
| `ALIAS_UNKNOWN` | Alias resolution found no such alias. Aliases are intended to be discoverable, so this is reported explicitly and is not subject to anti-enumeration | alias registry |
| `CLIENT_UNREGISTERED` | Request carries a client ID that is not registered for this user (see `DeviceSync.md`) | server |
| `ACCOUNT_PENDING_APPROVAL` | The account exists but is frozen: a moderated server has not approved it yet, so its destination is not brought up. The message carries the operator's instructions for finishing registration | service node |
| `I2P_UNAVAILABLE` | The embedded I2P router is not available; direct I2P (e.g. placing/accepting a call) is impossible | client-side |

The registry grows as APIs are specified; codes are append-only and never reused.

## Anti-enumeration policy

Detailed mailbox-state errors (`STORAGE_FULL`) are returned **only after a valid delivery pass has been presented** - so only actual contacts can learn anything about a mailbox. Everything else (unknown pass, no such mailbox, account not served here) collapses into `DELIVERY_REJECTED`.

## Open questions

- Error-code vs HTTP-status mapping table.
