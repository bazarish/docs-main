# Federated resolution & contact discovery

Specifies how a client discovers a contact it knows only by a small scannable identifier or by
name, over I2P, with the home server learning nothing about it - it is not on the
path - and nothing between the two ends able to forge the result: on the
descriptor path the fingerprint anchors it outright, and on the name path the
signature chain leaves the central registry as the one party trusted for anything
(*Residual trust*, below). The identity/certificate formats are in `../Identity.md` and the I2P transport
in `Federation.md`. How the registry daemon is built and run is not public and
is not here: it is documented in its own repository
(`alias-resolver/docs/Daemon.md`).

## Two artifacts

- **Descriptor** - a small, single-QR pointer:
  ```
  https://bazari.sh/#invite?v=1&fp=<52-char base32 fingerprint>&dest=<b32>.b32.i2p&key=<card key>[&name=<percent-encoded>]
  ```
  identity (`fp`) + the user's **own** destination (`dest`, a `.b32.i2p` host)
  + the card key that seals the card fetch (`key`, 32 bytes as unpadded base64url;
  a link without it, or with any other length, is refused),
  plus an optional `name` the
  inviter advertises (percent-encoded). The recipient adopts `name` as the
  contact's local display name at add time - a one-time static label, never
  re-fetched or transmitted afterwards (`Messages.md`).
  Rendered for humans as `fp:b32` (the `:` distinguishes it from an alias). It
  points at the contact card.
- **Contact card** - user-signed (`t`, `issuedAt`, `dest`, `sealingKey`, `servingKey`, the owner taken from the signature;
  ~= 8.7 KB, the hybrid post-quantum signature dominates), fetched live and
  verified against `fp` (`../Identity.md`). It names **no server**: a card that
  did would tell every contact who hosts its owner, and let two cards be compared
  for co-location.

An invite carries the descriptor; the card is fetched live. The user's server
must be reachable at add-time; the invite fits one QR, the card is always fresh,
and the home server never sees who is being added (below).

## Two lookups, different trust

1. **Card fetch - `fp` is known** (from a descriptor/QR). Fetch the contact card
   for `fp` at `dest`. `fp` is the integrity anchor: the card is user-signed and
   verified against `fp`, so the serving server can only withhold, never forge.

   The fetch is **direct over I2P**: the client dials `dest` from a fresh
   transient destination, so the requester's own server is not on the path and
   learns nothing of the identity being added. Without a router the client
   reports that the fetch is unavailable.

   Query and answer are sealed under the descriptor's card key (*Card-fetch
   protocol*, below) inside an I2P stream that is encrypted and authenticated to
   the destination.

   The **destination is the capability**. A blinded `.b32.i2p` address cannot be
   guessed and cannot be recovered from the network - a floodfill holds only the
   day's blinded key - so the only way to reach the card is to have received
   the descriptor. The destination answers for its own user alone, so the fetch
   cannot be used to ask a server whether it hosts some other fingerprint. A
   caller whose descriptor is out of date reaches nothing and knows to ask for a
   fresh one. Raising a new destination retires every descriptor already shared,
   and generates a new serving key with it.
2. **Alias resolve - `fp` is unknown.** `alias` -> descriptor. This is the only
   step without an `fp` anchor, so it is the step where MITM and the trusted
   alias->identity binding apply. Served by the single central resolver.

## Address / contact forms

| Form | Meaning | Trust |
|---|---|---|
| `<fp>:<b33>` (descriptor) | identity + direct route; primary QR/link | strongest - `fp` anchor, no substitution, and nothing in between to learn `fp` |
| `<alias>`, `<alias>-<word>` | name on the central resolver, with the owner's word when one is set | trusted alias->`fp` binding; UI must show the resolved `fp` for out-of-band verification |

Aliases are global on the single resolver.

## Transport: direct dialling

Mirrors the direct file-transfer fetch (`FileTransfer.md`).

The client dials `srv` (card fetch) or the resolver's b32 (alias resolve) itself,
over a **fresh transient I2P destination**. I2P encrypts the stream end to end to
the destination, and the `.b32.i2p` address is itself a commitment to the
destination key; the target observes only an unlinkable transient. A `.b32.i2p`
host is therefore the unit of cross-server reachability (`Facade.md`).

## Card-fetch protocol (`fp` -> contact card)

```
client has descriptor {fp, dest, key}
  seal(k, m) = nonce(12, random) || ChaCha20-Poly1305(HKDF-SHA256(key, salt = "", info = k), nonce, m)
  query   = seal("bazarish card request v1", { v, fp })
  -> dial srv (direct dial over a transient I2P destination)
  srv opens the query with its user's card key; answers only for its own user's fp:
  response = seal("bazarish card reply v1", <card DER>)
  client: open, verify card signature; assert signer fingerprint == fp, card.dest is .b32.i2p
```

`key` is the account's **card key**: 32 random bytes the account makes once, keeps
with its profile (so it survives a backup and travels with device pairing) and hands
to its node with every card it publishes; the node gives it to the messaging server
with the card. It reaches a contact only inside the descriptor - an invite link, a
QR, a resolver record - or inside the `bootstrap` block of a contact request and its
acceptance, so a contact who came in by request can share the descriptor onward.
The stream is already confidential to `srv`; the seal is against whatever relays it
before that - a gateway is the client's router and reads everything the application
does not seal, and without the key it learns neither whose card was asked for nor
the card. A query sealed under another key is answered `CARD_UNKNOWN`, the same as
a fingerprint the destination does not serve. Integrity is still the `fp` anchor.

## Alias-resolve protocol (`alias` -> descriptor)

The full **signed** resolve is served over the resolver's I2P endpoint only,
for uniform client automation. The resolver returns a
**self-verifying signed record**:

```
sign_delegated({ v, t: "resolve-record", alias, descriptor: {fp, dest, key},
                 issuedAt, notAfter })
  + delegationCert     # root -> delegated
  + aliasCert          # the owner's own claim over the name, user-signed
```

The client verifies the chain `record -> delegationCert (delegated key <- root) ->
hardcoded resolver root fingerprint`, both `issuedAt` stamps against its own
clock, both `notAfter`, that the record is for the name it asked about, and the
owner's certificate - over the same name, and by the identity the descriptor
names. An answer carrying no owner certificate is refused, which is what a
registry declining to carry one would send. It then runs the card-fetch above with
the descriptor. A free/unknown name returns `ALIAS_UNKNOWN`.

- **Integrity / anti-MITM** is the signature chain to the **hardcoded root
  fingerprint** - transport-independent, so anything in the path can only DoS,
  never forge. The resolver's b32 also authenticates the destination.
- **Confidentiality** of *which* name is looked up: the client dials the resolver
  itself from a transient destination, and the lookup travels in the sealed
  channel (below), so nothing between the two learns the name.
- **Residual trust**: the resolver asserts the alias->`fp` binding (the nature of
  names), and the owner's certificate bounds that without removing it - a name
  cannot be pointed at somebody who never signed for it, but which identity holds
  a name is the registry's to say. What the hardcoded anchor removes is everything
  else: the path can withhold an answer and nothing more. The defence left is
  **surfacing the resolved `fp`** (safety-number style) - the fingerprint the
  client adds is returned to whatever asked for the add, so it can be compared out
  of band afterwards.
- **Cacheable**: the signed record is self-verifying, so clients may cache it up
  to `notAfter`, relieving load on the single resolver.

### The alias grammar

`a-z` and `0-9` only, case-insensitive: a name is folded to lowercase as the
single canonical form (`Alice` == `alice`), so case-variant duplicates cannot
exist. No separators and no non-ASCII - a separator would let two names differ by
a character nobody reads. The length bounds are `kAliasMinLength` and
`kAliasMaxLength` in the core (1-16), and **both sides are built with the same
constants**: a client normalizes what was typed against them before it asks, the
registry canonicalizes against them, and a name outside them is answered as a
name nobody holds. The cap also keeps an alias
structurally distinct from the 52-character base32 fingerprint, so the two can
never be confused.

A client reads whatever is typed into its add field that is not an invite link
(`https://bazari.sh/#invite?...`) as an alias, with or without the `!` sigil; a
fingerprint is never a way to add a contact. In message text the sigil stays
required, or every word would be a name.

### The owner's word

An owner may give a name a **word**: the same grammar as a name (`a-z`, `0-9`,
case-folded, 1 to `kAliasWordMaxLength` = 16), written after a dash -
`!alice-secret`. Once a word is set, the name is found only as `name-word`; the
bare name and any other word get `ALIAS_UNKNOWN`, the same answer a free name
gets, so a scan of the namespace learns nothing from it. The word is meant to be
published along with the name wherever the owner shares it, and changed when it
leaks into a spammer's list; nothing else about the name changes with it. The
owner sets it with `alias.word` or on the registry's website, `alias.status`
reports it back to the owner's devices, and a transfer clears it.

### Lookups under pressure

The resolver counts the lookups of every name over the last hour, whatever word
they carry and whether the name exists. While a name is looked up rarely, a
lookup costs nothing. Past a threshold the resolver answers `429` with
`{errorCode: PROOF_REQUIRED, stamp, bits}`, and the client asks again with
`?stamp=<stamp>&nonce=<n>` after finding a nonce for which

```
Argon2id(pass = n as 8 bytes big-endian, salt = the stamp's bytes,
         memory = 4096 KiB, passes = 1, lanes = 1, 32 bytes)
```

starts with `bits` zero bits. The cost is fixed in the core (`kPuzzleCost`,
about 4 ms an evaluation on a Cortex-A76 and on a desktop core, measured
2026-10-29), and `bits` never exceeds `kMaxPuzzleBits` = 14, about a minute on one
core. The stamp is the resolver's own: opaque to the client, bound to the name,
short-lived, and accepted once. How many lookups are free and how fast the work
grows with traffic is the resolver's policy (`alias-resolver/docs/Daemon.md`).
A lookup that is unusual pays nothing; a name being hammered - a word being
guessed, a list being scraped - costs every asker work in proportion to how hard
it is hammered.

### What a name can answer

`ALIAS_UNKNOWN` is the only negative, and it covers every reason a name does not
resolve: never registered, released, bound to nothing, or lapsed and inside the
registry's own grace. A client therefore cannot tell those apart and should
not attempt to.

A name that stops resolving is not a contact lost: resolve is an **onboarding**
step, and whoever already holds a descriptor keeps messaging over dest-routing
without ever consulting the resolver again. Only new lookups stop.

### The owner's own ops

On the same I2P face, an owner keeps their own names working. Each request is
signed by the owner's identity - the signature is what names them, so a request
concerns only the signer's own names - and each answer is signed by the resolver's delegated key
and travels with its delegation certificate, so a device that receives an
answer from a sibling device checks it against the same built-in root as a
record it fetched itself.

```
alias.status            every name this account holds
alias.update            point one of them at {fp, dest, key}, with
                        the owner's alias certificate over the name
alias.renew             pay for another term from the deposit
alias.autorenew         let the service pay, or stop it
alias.binding           make one of them resolve, or stop it
alias.word              set the word a name is looked up with, or take it off
alias.transfer.accept   take a name offered to this account
```

Each is an op inside the sealed channel below, as is a lookup (`resolve`).

`alias.binding` carries a boolean, like `alias.autorenew`. Switching it off
clears the stored descriptor, so the name answers nobody while the account keeps
holding it and keeps paying for it; switching it on makes the name resolvable
again and leaves the client to repoint it with `alias.update`. The same switch
exists on the registry's own website, and either side may operate it.

**Two flags per name, and only one of them is the client's.** `alias.status`
reports both. `bindingWanted` is whether the name resolves, and either the client
or the website may set it. The second flag says whether the name belongs in the
messenger at all, and **only the website sets it**: an account may hold a hundred
names for other purposes while five are the ones it wants to be reachable by, and
a client lists those five alone. Switching a name's resolving off in the client
does not take it out of that list - it stays, switched off, and can be switched
back on without the website. A client therefore filters its own display on the
website flag and services every name that carries `bindingWanted`, including ones
it does not show: after a move, each of them has to be repointed by the key that
signed it, and nobody else can do it.

A request carries its own `issuedAt` and is refused outside a freshness window;
the op named in the envelope must be the op the signature covers, so relabelling
an envelope cannot turn a status into an update. A status answer carries its own
window too, which is how long it may stand in for a freshly fetched one when a
device forwards it to the account's other devices.

### The sealed channel

Everything a client asks the resolver travels sealed: the name it looks up, the
puzzle answer, and every owner's op. The stream to the resolver is encrypted to
its address, but a gateway is the client's own router and reads whatever the
application does not seal. The face answers two paths and nothing else:

```
GET  /v1/channel   {"key": <signed channel key>, "delegation": <delegation cert>}
POST /v1/sealed    hybrid_seal(channel key, pad({op, body, replyKey}))
                   -> 200, KeyedSeal(replyKey, "bazarish resolver reply v1", pad32k({status, body}))
```

- **The channel key** is a hybrid sealing key (X25519 + ML-KEM-768) the daemon makes
  at start and keeps in memory only, so each restart changes it and each node of a
  pair has its own. It is signed by the delegated key (`{v, t: "resolver-channel",
  key, issuedAt}`), and a client checks it like any other answer: delegation
  certificate to the built-in root, signer is the delegated key, `issuedAt` not
  ahead of its clock. A gateway cannot put its own key in place, having no key the
  root delegated to.
- **A request** is CBOR `{op, body, replyKey}` padded to the ladder (`Padding.hpp`)
  and sealed to the channel key; `replyKey` is 32 random bytes the client makes for
  this one request. `op` is `resolve` with the lookup `{v, alias, word?, stamp?,
  nonce?}` as body, or one of the owner's ops with its signed request as body.
- **The reply** is CBOR `{status, body}` - the status and the JSON a plain face would
  have answered (tables below) - padded to a multiple of 32 KiB and sealed under
  the request's `replyKey`. The outer answer is `200` whatever the inner status, so
  a found name, an unknown one and a puzzle look the same from outside.
- **A key the daemon no longer holds** - after a restart, or a request sealed for
  the other node - is answered `409 CHANNEL_KEY_UNKNOWN` in the clear. A client
  holds the key it checked in memory until it hears that, then fetches the key
  again and asks once more.

### What the face refuses, and what it counts

The resolve face answers nobody it cannot name: the i2pd tunnel in front puts the
caller's own `.b32.i2p` on each request, having stripped that header from what
the caller sent, and a request arriving without one is refused. **Resolves are
counted against a per-caller window** and a client must expect to be told it has
asked too often; an owner's ops are not counted, because they are signed, are one
per name held, and a client with twenty names has to be able to repoint all twenty
in one run.

### Error answers

Outside the channel there are two refusals: `403 NO_CALLER` (no caller address
from the tunnel) and `409 CHANNEL_KEY_UNKNOWN`; the channel key itself is counted
against the caller's window (`429 TOO_MANY_REQUESTS`). Inside, every answer is a
status and a JSON body `{"errorCode": <code>}`; `PROOF_REQUIRED` adds `stamp` and
`bits`. A client branches on the code.

A lookup, op `resolve`:

| Status | Code | Meaning |
|---|---|---|
| 403 | `NO_CALLER` | The request arrived without the caller's address the tunnel puts on it |
| 429 | `TOO_MANY_REQUESTS` | This caller's window of resolves is spent |
| 429 | `PROOF_REQUIRED` | The name is looked up too often; ask again with a solved stamp (*Lookups under pressure*). A wrong, stale or spent answer gets a new stamp |
| 400 | `MALFORMED_REQUEST` | The lookup does not read |
| 503 | `BUSY` | Too many answers are being checked at once; ask again |
| 404 | `ALIAS_UNKNOWN` | The name does not resolve with this word, or is not a name (*What a name can answer*) |
| 503 | `RESOLVER_NO_SIGNING_KEY` | The resolver holds no delegated key and cannot sign an answer |

The owner's ops:

| Status | Code | Meaning |
|---|---|---|
| 403 | `NO_CALLER` | As above |
| 404 | `UNSUPPORTED_OP` | No op of this face goes by that name |
| 400 | `MALFORMED_REQUEST` | The request could not be processed |
| 404 | `REQUEST_REFUSED` | The signature does not verify, the request is outside its freshness window, it names another op than the one it was sent as, or it was carried out already |
| 404 | `NOT_OWNER` | The signer does not hold the name (`alias.update`, `alias.binding`, `alias.word`, `alias.autorenew`, `alias.renew`) |
| 404 | `BAD_DESCRIPTOR` | `alias.update`: the descriptor or the owner's certificate does not check out |
| 404 | `WRONG_STATE` | `alias.renew`: the name is not in a state that renews |
| 404 | `INSUFFICIENT_DEPOSIT` | `alias.renew`: the deposit does not cover the term |
| 404 | `NO_OFFER` | `alias.transfer.accept`: nothing is offered to this account under that name |
| 404 | `UNAVAILABLE` | This resolver does not run renewals or transfers |
| 503 | `RESOLVER_NO_SIGNING_KEY` | As above |

The registry's website is not this API. It refuses a purchase of a held name
with `409` and `ALIAS_TAKEN` in the common envelope (`../Errors.md`).

### The coordinate a client is built with

Two values: the resolver's **root fingerprint** and the **b32 it answers at**.
The channel key is fetched and checked against the root each time the resolver
changes it, so nothing a client is built with goes stale when the resolver rotates its signing or channel key behind the
same root. The root fingerprint is the single
constant `bazarish::kResolverRootFingerprint`, the same one the daemon is built
with. With either half empty the alias path reports that resolution is
unavailable. A test or a client pointed at another resolver is given its
coordinate explicitly (`Session::setResolverCoordinate`).

### Why one registry at all

A globally unique sellable name needs a single registry, as a domain does. The
centralization is bounded to the name binding: the resolver is **out of the
delivery path** (routing is federated and never consults it after the one-time
lookup), it cannot forge an identity (descriptors and cards are user-signed), and
it cannot see who talks to whom. Names are a convenience layer over a base
that works without them - two people can always exchange `https://bazari.sh/#invite?...`
descriptors by QR or link and never contact the resolver (`../Identity.md`). How
that registry is built, what it charges and how it is operated are its own
concern and are not specified here.

## What is hidden from whom

| Party | Card fetch | Alias resolve |
|---|---|---|
| The requester's own server | Nothing: it is not on the path | Nothing: it is not on the path |
| I2P network | nothing: the stream is encrypted and authenticated to the destination | the same |
| Target (the serving server / resolver) | the `fp` (it serves that card) and that someone holding the descriptor asked | the alias (it owns the namespace); requester unlinkable on direct path |
| Client | the verified card | the verified descriptor -> card |

Namespace anti-enumeration is out of scope for the resolver (resolve is unauthenticated by design). The **card fetch** is protected: it answers only on the account's own destination, which has to have been shared, and never through a facade, so a contact cannot try the descriptor it holds against other servers to learn which one hosts its owner.
