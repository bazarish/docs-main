# Admin panel

The `admin-panel` is one web interface to monitor and manage Bazarish
deployments. A deployment is the unit that scales: **one messaging server, the
service node bound to it, and the facades in front of the pair** - a subscription
certificate names one messaging server, and the node that issues it is bound to
that server's card, so the two do not scale apart. Facades do: several may front
one pair, and different facades may front different pairs. The central alias
service is not managed here at all - it belongs to the project, not to an
operator's server. It is a
co-located operations tool: it reaches each service on its private listen address
and is never exposed through the facade or over I2P.

## Sign-in-with-key (panel access)

Access is gated to the configured **admin identities** - no passwords, no
account store. `admin` is one fingerprint or a list of them, so a fleet with
several operators does not have to share a key. Login reuses the portal sign-in-with-key wire contract, but the
panel is the self-contained verifier, so **no backend has to be online to log
in**.

1. The operator puts their own fingerprint (`bazarish-client whoami`) in the
   panel config `admin` field.
2. `GET /login` issues a challenge: base64(JSON `{nonce, ts, aud}`), where `aud`
   is the panel's audience.
3. The operator signs it with their client: `bazarish-client sign-login <profile>
   <challenge>`, producing the standard login blob (base64(JSON `{k,t,c,p}`) - the
   four request-auth headers over the canonical `BZ-LOGIN` / `/portal/login`
   request whose body is the challenge).
4. `POST /login` verifies both signatures locally (common `auth::verifyRequest`),
   requires the recovered fingerprint to be one of the configured admins, then consumes
   the challenge and sets a session cookie (`HttpOnly; SameSite=Strict`).

The challenge is **single-use** and bound to the panel audience. The panel only
accepts challenges it issued, so a login blob captured from any other verifier
(e.g. a service-node portal) carries a challenge the panel never minted and is
rejected even though its signature is valid; conversely a panel challenge cannot
be replayed at a portal (it has no portal HMAC tag).

## Operator authentication (panel <-> services)

Privileged service endpoints are authenticated by **hybrid signature**, not a
shared secret. The panel has its own identity (generated on first start, kept in
its state directory); it signs every service call with the request-auth headers
(common `auth::signRequest`). Each service authorises a configured set of operator
fingerprints (`--operator-fingerprint`, repeatable) and verifies the request with
`auth::authorizeRequest` (verify both signatures, then require the caller
fingerprint to be authorised). The service node likewise signs its own
quota/transient pushes with its identity, so server-core authorises both the
service node and the panel.

| Service | Authorised operators |
|---|---|
| server-core | service-node, admin-panel |
| service-node | admin-panel |
| facade-http | admin-panel |

## Status: `GET /healthz`

Every service exposes the same liveness endpoint, built from the shared
`common::health::reportJson`:

```json
{"service": "server-core", "version": "0.0.2", "uptime": 1234,
 "ramBytes": 10485760, "ramPeakBytes": 12582912}
```

`uptime` is seconds since the service's API came up; `ramBytes`/`ramPeakBytes` are
the current and peak resident set (Linux `/proc/self/status` VmRSS / VmHWM). The
endpoint is operator-signed, so it discloses nothing to an unauthenticated caller.

Every service but one serves it on the same private listener as its API, which is
already unreachable from the clearnet. The exception is **facade-http**, whose API
listener *is* the clearnet one: there `/healthz` would answer a caller who does not
know the secret prefix and thereby identify the host as a facade, so it lives on a
separate private ops socket (`--ops-listen <host> <port>`) and the panel is
configured with that address, not the published facade URL (`Facade.md`).
The panel probes every configured service concurrently and renders status,
version, uptime, current/peak RAM and round-trip latency.

## The panel itself

Past the sign-in page the panel is a small script talking to a JSON API of its
own. It is an operator tool on an operator's own perimeter, so it assumes
JavaScript; the sign-in page does not, because a fleet that is down must still be
loggable into.

A sidebar carries the views:

- **Monitoring** - every configured service of every deployment probed
  concurrently (status, version, uptime, current/peak RAM, latency), grouped by
  deployment and re-probed every few seconds. A facade also reports what it is
  carrying: requests over the last minute, hour and day, and how many are in
  flight. Those are sampled from bucketed counters (one per second for the last
  minute, one per minute for the last day) - a reading, not a history: nothing is
  stored and nothing has to be pruned.
- **Users** - the accounts on the service node. Search by fingerprint or by the
  address the account is served at, filter by approval state, and scroll: the
  list loads a page at a time. Each row carries whether the account is approved
  and whether its destination is delegated right now - the two say different
  things, and an account can be approved and unreachable. Open one to read its
  terms and act on it: set the storage quota (in whole megabytes), approve it,
  undelegate it (the address is revoked and the account waits again, with a
  message its owner is shown in place of the node-wide one), delete it, or ban
  the key. The open account is in the address bar (`#/users/<fingerprint>`), so a
  link points at one account.
- **Settings** - what the service node does with new accounts: open or moderated
  registration, the message a waiting account is shown, and the storage a new
  account gets. Beside them, the two numbers that say whether those settings
  still fit: what the quotas have promised, and what the messaging server's disk
  actually holds (`GET /v1/messaging/storage`, measured by walking its state
  directory - quotas are a promise, the filesystem is the fact).

The **search runs on the service node**, not in the browser: the panel passes
`q`, `state`, `limit` and `offset` through and shows what comes back, so a node
with many accounts never has to ship them all to find one.

| Panel route | Passed through to |
|---|---|
| `GET /api/services` | each service's `/healthz` |
| `GET /api/users?q&state&limit&offset` | service-node `GET /v1/account/users` |
| `GET /api/user?user=` | service-node `GET /v1/account/user` |
| `POST /api/approve` | service-node `POST /v1/account/approve` |
| `PUT /api/quota` | service-node `PUT /v1/account/quota` |
| `POST /api/undelegate`, `POST /api/delete`, `POST /api/ban`, `POST /api/unban` | the matching service-node operator endpoints |
| `GET /api/settings`, `PUT /api/settings` | service-node `/v1/account/settings`, plus server-core `GET /v1/messaging/storage` for the disk figure |

The quota is set **on the service node**, which is the component that grants it
and pushes it to server-core. Setting it straight on server-core (as an earlier
panel did) would have been overwritten by the node's next push.

The panel serves its web files from `--assets <dir>` (mandatory): the page shell,
the sign-in page, the stylesheet, the script and the logo, read once and answered
from memory. An operator restyles the panel by editing them and restarting it.

There is no hand-driven I2P destination form. A user's destination is delegated
by their own client and published by the service node
(`PUT /v1/messaging/i2p-dest` on server-core is that push, and stays); an
operator poking at it by hand had no place in that flow.

## Configuration

```json
{
  "admin": ["<admin hybrid fingerprint, base32>"],
  "audience": "bazarish-admin-panel",
  "listen": {"host": "127.0.0.1", "port": 8460},
  "services": [
    {"name": "server-core", "kind": "server-core", "url": "http://127.0.0.1:8420"},
    {"name": "facade", "kind": "facade", "url": "http://127.0.0.1:8429"}
  ]
}
```

No secrets live in the config. The panel prints its own operator fingerprint at
startup; add it to each service's `--operator-fingerprint`.
