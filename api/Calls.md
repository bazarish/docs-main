# Calls (Audio / Video)

Status: draft skeleton.

## Principle

No direct contact attempt between peers — direct connections harm privacy. A call is negotiated through ordinary E2E messaging and carried over a hidden-network tunnel.

## Flow (first iteration, I2P)

1. **Caller** asks the local I2P router via **SAM API** to create a fresh server tunnel: UDP, encrypted leaseset ("b33"). One tunnel per call, discarded afterwards.
2. Caller sends a **call-invite** — a regular E2E message of a dedicated type — through the normal delivery path. The invite carries the contact type (audio/video) and the one-time tunnel destination + connection credentials.
3. **Callee** receives the invite, and on accept connects to the destination through I2P.
4. Media flows peer-to-peer over the I2P tunnel; servers and facades never see call traffic.
5. If no SAM API is available on either side, direct contact is impossible — typed error `SAM_UNAVAILABLE`, surfaced in the UI.

## Signaling message types (draft)

| Type | Direction | Purpose |
|---|---|---|
| `call-invite` | caller → callee | Contact type, one-time destination, credentials |
| `call-accept` | callee → caller | Callee is connecting |
| `call-decline` | callee → caller | Rejected |
| `call-end` | either | Hang up / cancel |

All signaling is E2E-encrypted message content; server-side components see only opaque blobs of a generic message.

## Open questions

- **Media encryption**: the I2P tunnel is encrypted, but end-to-end media protection independent of the transport is desirable — standard options (DTLS-SRTP) vs relying on the b33 tunnel alone. Standard primitives only.
- **Codecs and media framing** (likely Opus; video codec TBD) — client-repo concern, but the invite must negotiate it.
- **Timeouts and busy states**: ringing duration, `call-busy`, glare (simultaneous calls in both directions).
- Latency reality check over I2P UDP tunnels; may dictate audio-first scope.
