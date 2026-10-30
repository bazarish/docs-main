# Error Model

## Principles

- Every API-visible failure is a **typed error**: a stable machine-readable code plus a human-readable message. Clients branch on codes, never on messages.
- No error hiding (codestyle). Server-side, an unhandled exception is acceptable where no meaningful handling exists; anything that crosses an API boundary must be typed.
- Errors must not disclose information that assists deanonymisation. For example, distinguishing "user unknown" from "not a contact of the recipient" would reveal the existence of an account (see *Anti-enumeration policy*).

## Envelope

```json
{
  "error": {
    "code": "MESSAGE_TOO_LARGE",
    "message": "a message may carry at most 524288 bytes"
  }
}
```

## Registry

| Code | Meaning | Source |
|---|---|---|
| `STORAGE_FULL` | User's pending-content quota is exhausted. Transient: the mailbox takes mail again once its owner's devices have taken in what it holds | server |
| `RECIPIENT_SERVER_UNREACHABLE` | The recipient's server returned no answer: it could not be dialled, the stream failed, or it asserted a delivery without a verifiable signature. Not a refusal; the sending client may repeat the attempt, and on exhausting its attempts reports this as the reason (`Messages.md`) | client |
| `MESSAGE_TOO_LARGE` | The payload is over `kMaxMessagePayloadBytes`. Past admission, so only a correspondent ever sees it | server |
| `CONTACT_RATE_LIMITED` | This destination is taking too many contact requests, has too many waiting, or has no room for one more; or a correspondent is sending more than its pass admits. Transient; says nothing about who is served there | server |
| `SESSION_INVALID` | The session a request authenticated with no longer exists. Answered by opening a new tunnel, never by retrying the same way | server |
| `UPSTREAM_UNAVAILABLE` | The backend behind a facade did not answer: the messaging server or the service node. Transient | facade |
| `REQUEST_FAILED` | A request outside delivery failed for a reason no other code names; the status says which (below). It carries no meaning a client acts on beyond the status class | any component |
| `ACCOUNT_UNKNOWN` | The service node holds no account for the signer. The answer to it is registering (`api/ServicePortal.md`) | service node |
| `DELIVERY_REJECTED` | Delivery refused. Deliberately combines "unknown pass", "unknown mailbox" and "recipient not served here" into one externally indistinguishable result (anti-enumeration; see `Contacts.md`). A contact request, which presents nothing, is the only alternative admission path | server |
| `CONTACT_REQUEST_TOO_LARGE` | Contact request exceeds `kMaxContactRequestBytes` (17590, a protocol constant - not an operator setting) | server |
| `ALIAS_TAKEN` | Requested alias is already held. The registry's portal answers a buy request for it with 409 and this code | alias registry |
| `ALIAS_UNKNOWN` | Alias resolution found no such alias. Aliases are intended to be discoverable, so this is reported explicitly and is not subject to anti-enumeration | alias registry |
| `CLIENT_UNREGISTERED` | Request carries a client ID that is not registered for this user. Only the routes that act for one device check it: the mailbox listing and wait, fetch, ack, the pass set and the write to the account's own devices. The account routes and the client registry take any request the user signed (`DeviceSync.md`) | server |
| `ACCOUNT_PENDING_APPROVAL` | The account exists but is frozen: a moderated server has not approved it yet, so its destination is not brought up. The message carries the operator's instructions for finishing registration | service node |
| `CARD_UNKNOWN` | A `card` request named a fingerprint that is not this destination's user, or a user with no card - one answer for both (`api/Federation.md`) | server |

A code is never reused for another meaning.

## HTTP status

Over HTTP a typed error travels with the status below; inside a tunnel this is the
status carried in the seal, and the outer answer is always `200`
(`api/ClientApi.md`). A client branches on the code and reads only the status
class: a 4xx is a refusal, a 5xx or no answer is the far side failing for now.

| Code | Status |
|---|---|
| `SESSION_INVALID` | 401 |
| `CLIENT_UNREGISTERED` | 403 |
| `ACCOUNT_PENDING_APPROVAL` | 403 |
| `MESSAGE_TOO_LARGE` | 413 |
| `STORAGE_FULL` | 507 |
| `ALIAS_TAKEN` | 409 |
| `ALIAS_UNKNOWN` | 404 |
| `ACCOUNT_UNKNOWN` | 404 |
| `UPSTREAM_UNAVAILABLE` | 502 |
| `REQUEST_FAILED` | the status of the failure: 400 for a malformed request, 401 for a signature that does not verify, 403 for a signer that may not do this, 404 for a route or a thing that does not exist, 409 for a write that is not newer, 500 for an internal failure, 503 for a function this component does not offer |

`DELIVERY_REJECTED`, `CONTACT_RATE_LIMITED`, `CONTACT_REQUEST_TOO_LARGE` and
`CARD_UNKNOWN` travel only in a federation reply, which has no status (`api/Federation.md`); `STORAGE_FULL`
and `MESSAGE_TOO_LARGE` travel there as well. `RECIPIENT_SERVER_UNREACHABLE` is
raised by the sending client and never sent.

## Anti-enumeration policy

Detailed mailbox-state errors (`STORAGE_FULL`) are returned **only after a valid delivery pass has been presented** - so only actual contacts can learn anything about a mailbox. Everything else (unknown pass, no such mailbox, account not served here) collapses into `DELIVERY_REJECTED`.
