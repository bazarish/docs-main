# Sign-in-with-key (login blob)

Status: implemented.

A Bazarish key is a login. A site issues a **challenge**, the user signs it in
their client, and pastes the **login blob** back; the site recovers the signer's
fingerprint and starts a session. The private key never enters the browser and no
password exists anywhere.

This document is the whole of authentication by signature: the signed request
(section 1), the challenge and the blob built on it (sections 2 and 3), how a
verifier checks a login (section 4) and what a client must do before it signs
(section 5). Other documents refer here and do not restate it.

Every verifier in the project speaks this one contract: the service-node portal
(`ServicePortal.md`), the alias registry and the admin panel
(`AdminPanel.md`). A site outside the project verifies logins with
[auth-sdk](https://github.com/bazarish/auth-sdk), which implements the verifier
side in Go, Rust, Python and JavaScript.

## 1. The signed request

A signed request is five values that say who made it, when, and that nothing in
it was changed. It is the project's one signature scheme for authentication, and
three things carry it:

- a **login blob** (section 3);
- the client API's **first contact**: the tunnel hello, which carries the five
  values inside its sealed frame and signs method `BZ-TUNNEL`, path `/hello`,
  body the hello's fresh secret (`ClientApi.md`);
- every **operator call** - from the admin panel to a service, and between the
  messaging server and the service node - where the callee then requires the
  signer's fingerprint to be in its configured operator set (`AdminPanel.md`).

### Encodings

- base64 is the standard alphabet with padding (RFC 4648 section 4);
- hex is lowercase;
- a time is whole seconds since the Unix epoch, in decimal.

### The canonical string

The **base string** names the request:

```
"v2\n" + t + "\n" + METHOD + "\n" + path + "\n" + hex(sha256(body)) + "\n"
```

followed by `clientId + "\n"` when the request names a device (`ClientApi.md`).
`body` is the request body's bytes, empty when there is none. The **signed
string** is the base string followed by `n + "\n"`, with `n` exactly as it
travels.

### The values

| Header | Holds |
|---|---|
| `X-Bazarish-Keys` | base64 of the JSON `{"c": base64(Ed25519 SPKI DER), "pq": base64(ML-DSA-65 SPKI DER)}`: the signer's two public keys |
| `X-Bazarish-Timestamp` | `t`, when it was signed |
| `X-Bazarish-Nonce` | `n`, 16 random bytes, base64, fresh for every request |
| `X-Bazarish-Sig-Classical` | base64 of the Ed25519 signature (RFC 8032, pure) over the signed string |
| `X-Bazarish-Sig-Pq` | base64 of the ML-DSA-65 signature (FIPS 204, empty context) over the signed string |

Both halves sign the same bytes, and both must verify - one alone proves
nothing. The five headers come to 8258 bytes, measured.

### Verifying one

1. `t` is within 300 s of the verifier's clock, either side.
2. The `c` key is Ed25519 and the `pq` key is ML-DSA-65. A key of any other type
   in either slot fails: a classical key in the pq slot is a downgrade.
3. Both signatures verify over the signed string, rebuilt from the request as it
   arrived.
4. The signer's **fingerprint** is derived from the keys: lowercase unpadded
   base32 (RFC 4648) of `sha256(Ed25519 SPKI DER || ML-DSA SPKI DER)`, 52
   characters - the identity fingerprint of `Identity.md`. This is *who signed*,
   with no prior key exchange.

Where a request must be accepted once, the verifier also remembers every
accepted `n` for the window and refuses it a second time. A legitimate resend
carries a fresh nonce and passes; a captured request replayed verbatim does not,
and editing its nonce breaks both signatures. Ed25519 is deterministic, so the
nonce is the only thing that makes a resend distinct. The client API keeps such a
cache, and so does every operator endpoint: there the nonce is remembered only
after the caller is found among the operators, so a stranger cannot fill the
cache. A login does not need one: its challenge is single-use (section 4).

## 2. The challenge

Minted by the site, `base64(JSON)`:

```json
{
  "v": 1,
  "nonce": "5f3c...",
  "ts": 1756600000,
  "consumer": {
    "name": "Bazarish admin panel - north fleet",
    "place": ["https://panel.example", "http://panel.b32.i2p"],
    "role": "Administrator"
  },
  "tag": "9ab1..."
}
```

| Field | Meaning |
|---|---|
| `v` | Envelope version, the number `1`. A client that does not know the version does not guess: it refuses. |
| `nonce` | 16 random bytes, hex. Single use. Not the blob's `n`: this one belongs to the challenge. |
| `ts` | When the challenge was minted, seconds, a number. Verified within a freshness window (300 s, either side). |
| `consumer` | Who consumes the signature. Shown to the user before signing. |
| `tag` | `hex(HMAC-SHA256(secret, nonce + "\n" + ts + "\n" + canonical(consumer)))` under a portal-local secret, so the portal recognises its own challenge without keeping state, and **the label cannot be changed after minting**. |

`canonical(consumer)` is the compact JSON object with keys in lexicographic
order - `{"name":...,"place":...,"role":...}` - which is what the tag covers and
what a verifier compares. Strings are UTF-8 with no `\u` escapes: a non-ASCII
name is written as itself. Only the verifier that minted a challenge ever computes
its tag; the client does not check it.

The site gives the challenge to the user and receives it back together with the
blob. Because of the tag it does not have to remember what it issued, only which
nonces have been consumed.

### The consumer

| Field | What it holds | Limit, UTF-8 bytes |
|---|---|---|
| `name` | The resource as it calls itself, in the words on its own pages | 96 |
| `place` | **A list** of every address it answers at - URL, b32, host:port - in a stable order, because the tag covers the list. At most 4: a list nobody reads is not a check | 256 each |
| `role` | What is being signed into: `Administrator`, `Account owner`, `Alias owner`, `Operator` | 48 |

All three are required and non-empty, and `place` holds at least one address.
Each field is well-formed UTF-8, and none may carry a code point that breaks a
line or reorders the text around it:

| Code points | What they are |
|---|---|
| U+0000-U+001F, U+007F-U+009F | C0 controls, DEL, C1 controls (NEL among them) |
| U+2028, U+2029 | Line and paragraph separators |
| U+061C, U+200E, U+200F | Arabic letter mark, left-to-right and right-to-left marks |
| U+202A-U+202E, U+2066-U+2069 | Bidi embeddings, overrides and isolates |

A `name` with a newline in it could render a `place` line of its own in the
client's window, and a bidi override could show an address in an order other
than the one it has - which is also why `place` is a list rather than one field
with a separator in it. The set is closed and written out here rather than taken
from a Unicode category, so every implementation refuses exactly the same
strings. Joiners (U+200C, U+200D) stay allowed: they are part of the spelling in
Persian, Indic scripts and emoji, and break nothing. A challenge that breaks any
of these rules is not signed.

**Why more than one.** A place is often reachable two ways at once - over
clearnet and over I2P - and a challenge naming only one of them is refused by the
very user it is meant to serve: they arrived by the other. The check a person
performs changes from "does this match where I am" to "is where I am in this
list", which is still a question they can answer. It stays answerable only while
the list is short, hence the cap, and the client says **"If the address you came
to is not in the list, do not sign."** when there is more than one.

What each of the project's verifiers puts there:

| Verifier | name | place | role |
|---|---|---|---|
| Service node portal | `portal.name`, else "Bazarish server" | `portal.address`, else all of `portal.links` | `portal.role`, else Account owner |
| Alias registry, visitor face | `portal.name`, else "Bazarish alias resolver" | `portal.address` (required) | `portal.role`, else Alias owner |
| Alias registry, operator face | `admin.name`, else "Bazarish alias resolver admin" | `admin.address` (required) | `admin.role`, else Operator |
| Admin panel | `consumer.name`, else "Bazarish admin panel" plus the fleet name when the config names exactly one | `consumer.place`, else where it listens | `consumer.role`, else Administrator |

Each of those settings is a **list** in the configuration file.

A deployment that cannot state where it is reachable does not start. An address
nobody can compare is the same as no address at all. `place` is what the user
has in their address bar, so a verifier fronted under a name of its own says so
in its config.

The service node's three are **live**: they are among the settings an operator
edits in the admin panel (`loginConsumer` on `/v1/account/settings`), a change
takes effect on the next challenge, and it is written back to `portal.name` /
`portal.address` / `portal.role` in the node's config file. A challenge issued
before the change names the old consumer and stops verifying - it was shown to a
user under words the portal no longer asserts, and re-labelling it is what the
tag exists to prevent. The user loads a fresh challenge. The other verifiers
keep their consumer in their own config and read it at start.

## 3. The blob

Produced by the client, `base64(JSON)`:

```json
{"k": "...", "t": "1756600004", "n": "...", "c": "...", "p": "..."}
```

It is a signed request (section 1) whose five values travel as the blob's fields
rather than as headers, over method `BZ-LOGIN`, path `/portal/login` and body =
the challenge string exactly as issued, still in base64. A login blob names no
device, so its base string has no device line. The string signed is therefore

```
"v2\n" + t + "\nBZ-LOGIN\n/portal/login\n" + hex(sha256(challenge)) + "\n" + n + "\n"
```

| Field | Value of | Meaning |
|---|---|---|
| `k` | `X-Bazarish-Keys` | The signer's public keys, Ed25519 + ML-DSA. The verifier derives the fingerprint from them: this is *who signed*. |
| `t` | `X-Bazarish-Timestamp` | When it was signed, a decimal string. Outside the freshness window the blob is invalid. |
| `n` | `X-Bazarish-Nonce` | The per-request nonce, signed along with the rest. Without it a re-signed blob would be byte-identical to the first. |
| `c` | `X-Bazarish-Sig-Classical` | Ed25519 signature. |
| `p` | `X-Bazarish-Sig-Pq` | ML-DSA-65 signature. Both must verify - one alone proves nothing. |

The fixed method and path are a domain separator: a login blob is not a signature
over any real API call, and no API signature is a login.

## 4. Verification, in order

1. The challenge decodes and `v` is the version this verifier speaks.
2. `consumer` equals this deployment's own consumer, compared canonically.
3. `ts` is within the freshness window.
4. `tag` matches - the challenge was minted here. The comparison is constant-time.
5. The blob verifies as a signed request (section 1) over the login's method,
   path and challenge: `t` fresh, both key types correct, both signatures valid.
   The fingerprint is derived from `k`.
6. The challenge's `nonce` is consumed: remembered for the window and refused a
   second time. It is consumed **after** the signature check, so an invalid blob
   cannot consume a valid challenge.

What the fingerprint is then allowed to open is for the verifier to decide: an
account at the portal, a configured administrator at the panel.

Steps 2 and 4 are what bind a signature to one place: a blob captured at one
verifier carries a challenge no other verifier minted, and re-labelling that
challenge breaks the tag.

## 5. What the client must do

Before signing, the client **reads the consumer out of the challenge and shows
all three fields**, and it **refuses to sign** a challenge that does not decode,
is of another version, or does not name a complete consumer. Both halves matter:
a window that shows nothing leaves the user with nothing to compare, and a client
that signs an unnamed challenge issues a signature for a place it could not
name.

The words are shown as a claim, never as a verdict. The client says who the
challenge says it is for and tells the user to compare that with the site in
front of them; it does not say "trusted", it does not rank sites, and it does not
mark anything as safe.

## 6. What this stops, and what it does not

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
