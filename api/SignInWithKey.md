# Sign-in-with-key (login blob)

Status: implemented.

A Bazarish key is a login. A site issues a **challenge**, the user signs it in
their client, and pastes the **login blob** back; the site recovers the signer's
fingerprint and starts a session. The private key never enters the browser and no
password exists anywhere.

Every verifier in the project speaks this one contract: the service-node portal
(`ServicePortal.md`), the alias registry (`AliasResolver.md`) and the admin panel
(`AdminPanel.md`).

## 1. The challenge

Minted by the site, `base64(JSON)`:

```json
{
  "v": 1,
  "nonce": "5f3c...",
  "ts": 1756600000,
  "consumer": {
    "name": "Bazarish admin panel - north fleet",
    "place": "https://panel.example",
    "role": "Administrator"
  },
  "tag": "9ab1..."
}
```

| Field | Meaning |
|---|---|
| `v` | Envelope version. A client that does not know the version does not guess: it refuses. |
| `nonce` | 16 random bytes, hex. Single use. |
| `ts` | When the challenge was minted, seconds. Verified within a freshness window (300 s). |
| `consumer` | Who consumes the signature. Shown to the user before signing. |
| `tag` | `hmacSha256Hex(secret, nonce + "\n" + ts + "\n" + canonical(consumer))` under a portal-local secret, so the portal recognises its own challenge without keeping state, and **the label cannot be changed after minting**. |

`canonical(consumer)` is the compact JSON object with keys in lexicographic
order - `{"name":...,"place":...,"role":...}` - which is what the tag covers and
what a verifier compares.

### The consumer

| Field | What it holds | Limit |
|---|---|---|
| `name` | The resource as it calls itself, in the words on its own pages | 96 |
| `place` | Where it lives, as the user reached it: URL, b32, host:port | 256 |
| `role` | What is being signed into: `Administrator`, `Account owner`, `Alias owner` | 48 |

All three are required and non-empty. No field may carry control characters: a
`name` with a newline in it could paint a `place` line of its own in the client's
window. A challenge that breaks any of these rules is not signed.

What each of ours puts there:

| Verifier | name | place | role |
|---|---|---|---|
| Service node portal | `portal.name`, else "Bazarish service node" | `portal.address`, else the first advertised portal link | Account owner |
| Alias registry | `portal.name`, else "Bazarish alias registry" | `portal.address` (required) | Alias owner |
| Admin panel | `consumer.name`, else "Bazarish admin panel" plus the fleet name when one is named | `consumer.place`, else where it listens | Administrator |

A deployment that cannot say where it lives does not start. An address nobody can
compare is the same as no address at all.

The alias registry is specified here but has not adopted it yet: its build still
carries an older core, and it moves to this envelope when that is caught up.

## 2. The blob

Produced by the client, `base64(JSON)`:

```json
{"k": "...", "t": "1756600004", "c": "...", "p": "..."}
```

These are the four request-auth headers over a **canonical request**: method
`BZ-LOGIN`, path `/portal/login`, body = the challenge string exactly as issued.

| Field | Header | Meaning |
|---|---|---|
| `k` | `X-Bazarish-Keys` | The signer's public keys, classical + ML-DSA. The verifier derives the fingerprint from them: this is *who signed*. |
| `t` | `X-Bazarish-Timestamp` | When it was signed. Outside the auth freshness window the blob is dead. |
| `c` | `X-Bazarish-Signature` | ECDSA P-256 signature. |
| `p` | `X-Bazarish-Signature-PQ` | ML-DSA-65 signature. Both must verify - one alone proves nothing. |

The fixed method and path are a domain separator: a login blob is not a signature
over any real API call, and no API signature is a login.

## 3. Verification, in order

1. `v` is the version this verifier speaks.
2. `consumer` equals this deployment's own consumer, compared canonically.
3. `ts` is within the freshness window.
4. `tag` matches - the challenge was minted here.
5. The blob's two signatures verify over the canonical request; the fingerprint
   falls out of `k`.
6. The `nonce` is consumed. It is consumed **after** the signature check, so a
   bad blob cannot burn a good challenge.

Steps 2 and 4 are what bind a signature to one place: a blob captured at one
verifier carries a challenge no other verifier minted, and re-labelling that
challenge breaks the tag.

## 4. What the client must do

Before signing, the client **reads the consumer out of the challenge and shows
all three fields**, and it **refuses to sign** a challenge that does not decode,
is of another version, or does not name a complete consumer. Both halves matter:
a window that shows nothing leaves the user with nothing to compare, and a client
that signs an unnamed challenge hands over a signature for a place it could not
name.

The words are shown as a claim, never as a verdict. The client says who the
challenge says it is for and tells the user to compare that with the site in
front of them; it does not say "trusted", it does not rank sites, and it does not
badge anything as safe.

## 5. What this stops, and what it does not

It stops the **quiet** version of the attack. A site cannot mint a challenge for
somewhere else - the label it carries is the label the verifier will check - so
whatever a user is asked to sign, it says truthfully where the resulting session
will be opened, and a mismatch with the address bar is visible.

It does not decide for the user. Someone can still relay a genuine challenge from
a real service and hope the user signs without reading; the client's display, and
the user comparing it with where they are, is what catches that.

A consumer key fingerprint was considered as a fourth field and dropped. A client
cannot check a fingerprint it does not already know, so it would be one more
unverifiable string - and shown as a badge it would do harm, turning "this says
it is your server" into "this is safe" at exactly the moment the user should be
reading the two lines above it.
