# Admin panel

The `admin-panel` is a web interface for monitoring and managing Bazarish
deployments. A deployment is the unit of scaling: **one messaging server, the
service node bound to it, and the facades placed in front of the pair**. A
registration certificate names one messaging server, and the node that issues it
is bound to that server's card, so the two do not scale independently. Facades do:
several may front one pair, and different facades may front different pairs. The
central alias service is not managed here; it belongs to the project rather than
to an operator's deployment. The panel is a co-located operations tool: it reaches
each service on its private listen address and is never exposed through a facade
or over I2P.

## Sign-in-with-key (panel access)

Access is restricted to the configured **admin identities**; there are no
passwords and no account store. `admin` is a fingerprint or a list of
fingerprints, so a deployment with several operators need not share one key.

Login is the project's one sign-in-with-key contract (`SignInWithKey.md`), with
the panel as a self-contained verifier: **no backend has to be online to log
in**.

1. The operator puts their own fingerprint (`bazarish-client whoami`) in the
   panel config `admin` field.
2. `GET /login` issues a challenge naming this panel as the consumer.
3. The operator signs it with their client (`bazarish-client sign-login <account>
   <challenge>`, or the Signature window), which shows those words before it
   signs.
4. `POST /login` verifies the challenge and the blob, requires the recovered
   fingerprint to be one of the configured admins, consumes the challenge, and
   sets a session cookie (`HttpOnly; SameSite=Strict`).

What the panel puts in the consumer:

| Field | Value |
|---|---|
| `name` | `consumer.name`, or "Bazarish admin panel" plus the fleet name when the config names exactly one |
| `place` | `consumer.place`, or the address the panel listens on |
| `role` | `Administrator` |

The challenge is **single-use** and names the panel. A login blob captured from
any other verifier (a service-node portal, say) carries a challenge the panel
never minted and is rejected even though its signature is valid; conversely a
panel challenge cannot be replayed at a portal, and an operator who is sent one
sees a name that is not the panel's before signing anything.

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
know the secret prefix and would thereby identify the host as a facade, so it is
served on a separate private operations socket (`--ops-listen <host> <port>`) and the panel is
configured with that address, not the published facade URL (`Facade.md`).
The panel probes every configured service concurrently and renders status,
version, uptime, current/peak RAM and round-trip latency.

## The panel itself

Past the sign-in page the panel is a small script communicating with a JSON API of
its own. It is an operator tool within an operator's perimeter and therefore
requires JavaScript; the sign-in page does not, so that an operator can still sign
in when the deployment is down.

A sidebar carries the views:

- **Monitoring** - every configured service of every deployment probed
  concurrently (status, version, uptime, current/peak RAM, latency), grouped by
  deployment and re-probed every few seconds. A facade also reports what it is
  carrying: requests over the last minute, hour and day, and how many are in
  flight. These are sampled from bucketed counters (one per second for the last
  minute, one per minute for the last day): a current reading rather than a
  history, so nothing is stored and nothing requires pruning.
- **Users** - the accounts on the service node. Search by fingerprint or by the
  address the account is served at, filter by approval state, and scroll: the
  list loads a page at a time. Each row carries whether the account is approved
  and whether its destination is currently delegated; the two are independent, and
  an account may be approved and unreachable. Opening an account shows its terms
  and the available actions: set the storage quota (in whole megabytes), approve it,
  undelegate it (the address is revoked and the account waits again, with a
  message its owner is shown in place of the node-wide one), delete it, or ban
  the key. The open account is in the address bar (`#/users/<fingerprint>`), so a
  link points at one account.
- **Settings** - what the service node does with new accounts: open or moderated
  registration, the message a waiting account is shown, and the storage a new
  account gets; and what a user sees before signing a login for the portal - the
  name, the address and the role that make up its login consumer, which the
  client shows and the user compares with their address bar. The three go to the
  node as `loginConsumer` and are written back to its config file, so a restart
  keeps them. Beside them, the two numbers that say whether those settings
  still fit: what the quotas have promised, and what the messaging server's disk
  actually holds (`GET /v1/messaging/storage`, measured by traversing its state
  directory; a quota is an allocation, the measured figure is consumption).

The **search is performed on the service node**, not in the browser: the panel
passes `q`, `state`, `limit` and `offset` through and displays the result, so a
node with many accounts does not transfer the whole set to locate one.

| Panel route | Passed through to |
|---|---|
| `GET /api/services` | each service's `/healthz` |
| `GET /api/users?q&state&limit&offset` | service-node `GET /v1/account/users` |
| `GET /api/user?user=` | service-node `GET /v1/account/user` |
| `POST /api/approve` | service-node `POST /v1/account/approve` |
| `PUT /api/quota` | service-node `PUT /v1/account/quota` |
| `POST /api/undelegate`, `POST /api/delete`, `POST /api/ban`, `POST /api/unban` | the matching service-node operator endpoints |
| `GET /api/settings`, `PUT /api/settings` | service-node `/v1/account/settings`, plus server-core `GET /v1/messaging/storage` for the disk figure |

The quota is set **on the service node**, which is the component that grants it and
pushes it to server-core. Setting it directly on server-core, as an earlier
revision of the panel did, is overwritten by the node's next push.

The panel serves its web files from `--assets <dir>` (mandatory): the page shell,
the sign-in page, the stylesheet, the script and the logo, read once at startup and
served from memory. An operator changes the presentation by editing these files and
restarting the panel.

No manual I2P destination form is provided. A user's destination is delegated by
their own client and published by the service node (`PUT /v1/messaging/i2p-dest`
on server-core is that push and is retained); manual operator intervention has no
place in that flow.

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

The configuration contains no secrets. The panel prints its own operator
fingerprint at startup; that fingerprint is added to each service's
`--operator-fingerprint`.
