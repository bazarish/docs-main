# Admin panel

The `admin-panel` is a web interface for monitoring and managing Bazarish
deployments. A deployment is the unit of scaling: **one messaging server, the
service node bound to it, and the facades placed in front of the pair**. A
registration certificate names one messaging server, and the node that issues it
is bound to that server's card, so the two do not scale independently: a
deployment holds **one server-core and one service-node**, and the panel refuses a
second of either - a second server is a second deployment. Facades do scale:
several may front one pair, and different facades may front different pairs. A
deployment may also list private gateways, as many as it has or none: a gateway is
an optional part of a deployment and runs without the panel. Every service in a
deployment has a **name of its own**, because the name is what tells two facades
or two gateways apart on every page. The
central alias service is not managed here; it belongs to the project rather than
to an operator's deployment. The panel reaches each service on that service's
private operator socket, and is never exposed through a facade or over I2P.
Those sockets are mutually pinned TLS, so they need not be on the panel's own
host: a deployment may be spread over several machines without a tunnel between
them.

## Sign-in-with-key (panel access)

Access is restricted to the configured **admin identities**; there are no
passwords and no account store. `admin` is a fingerprint or a list of
fingerprints, so a deployment with several operators need not share one key.

Login is the project's one sign-in-with-key contract (`SignInWithKey.md`), with
the panel as a self-contained verifier: **no backend has to be online to log
in**. What follows is only what the panel adds to it.

1. The operator puts their own fingerprint - the one their client shows for their
   account - in the panel config `admin` field.
2. `GET /login` issues a challenge naming this panel as the consumer.
3. The operator signs it with their client, which shows those words before it
   signs (the desktop client's Signature window, or any client that implements
   the contract).
4. `POST /login` verifies the challenge and the blob, requires the recovered
   fingerprint to be one of the configured admins, consumes the challenge, and
   sets a session cookie (`HttpOnly; SameSite=Strict`).

What the panel puts in the consumer is listed with the other verifiers' in
`SignInWithKey.md`.

## Operator authentication (panel <-> services)

Privileged service endpoints are authenticated by **hybrid signature**, not a
shared secret. The panel has its own identity (generated on first start, kept in
its state directory); every service call it makes is a signed request
(`SignInWithKey.md`, section 1). Each service verifies it and then requires the
signer's fingerprint to be one of its configured operators (`operators`,
repeatable). The service node likewise signs its own
quota/transient pushes with its identity, so server-core authorises both the
service node and the panel.

| Service | Authorised operators |
|---|---|
| server-core | service-node, admin-panel |
| service-node | admin-panel |
| facade-http | admin-panel |
| gateway | admin-panel |

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

Every service serves it on its private operator socket (`opsListen`), a TLS
listener separate from its API, and the panel is configured with that address and
the key the service prints at startup (`pin`). The pinning runs both ways: the
socket asks for a client certificate and admits only the keys in its
`opsClients`, so an unlisted caller is refused at the handshake and never
reaches the signature check. Neither certificate decides what a caller may do -
that is still the signature and the `operators` list; TLS binds the two sockets
to each other. For **facade-http** this is what keeps the endpoint off the
sockets its clients reach: every client of the server holds that address, and
none of them needs the operator's endpoints (`Facade.md`).
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
  history, so nothing is stored and nothing requires pruning. A service's name
  opens its settings: the Server tab, the node's Addresses tab, or the page of
  that one facade or gateway.
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
- **Settings** - every setting of the deployment, in eight tabs, because one long
  page of similar-looking blocks is where an operator cannot find the one they
  came for. Each tab saves on its own and is in the address bar
  (`#/settings/<tab>`, and `#/settings/<tab>/<name>` for one facade or gateway),
  so a link points at one subject. The singletons have tabs of their own; the
  services a deployment may hold many of have one tab each that lists them by
  name, and a name opens that one's page:

  | Tab | What it holds |
  |---|---|
  | Registration | open or moderated, the message a new client is told and the one a waiting account is shown |
  | Portal | the login consumer (name, address, role) the client shows before a user signs, the portal's public links, and the socket the portal binds |
  | Addresses | the facades' addresses as they report them (read-only: the panel writes them, see below), the reseeds, the descriptor built from both, the node's own operator API and the messaging server it is bound to, and the log level |
  | Accounts | the storage a new account gets and how long an idle one is kept, beside the two numbers that show whether that promise can still be kept: what the quotas have allocated and what the disk holds (`GET /v1/messaging/storage`) |
  | Server | server-core's own file: log level and libi2pd's logging, its I2P privacy and tunnel count, its private listener, the listener its gateways reach it on (`gatewayListen`) and the node it reports to |
  | Facades | every facade of the deployment with its address and whether the descriptor carries it; a facade's page holds its own file, read and written over its ops socket - its three listeners (`listen`, `directListen`, `opsListen`), the messaging server it forwards to, its I2P address's privacy and tunnel count - its address, read-only, and **New address** |
  | Gateways | every gateway with its sessions and destinations; a gateway's page holds the tunnels it builds for all its clients (privacy, count), its two warm pools (streams, calls), and its link to server-core and its routes to the facades, read-only (the panel writes them, see below) |
  | Panel | the panel's own file: who may sign in, the words it signs under, the services it manages, and where it listens - plus what it cannot change (its fingerprint, its config file, its state and web directories, its version) |

  Every tab that holds a start-only setting also carries a **restart** control
  for the service that owns it (and the Panel tab one for the panel itself).
  Without it the rule this page exists for would not hold: a setting saved into a
  config file is inert until the process reads it again, and the operator would
  have to return to a terminal. The service exits cleanly with a distinct code
  and its unit starts it again; a service whose unit has no restart policy says
  so instead of offering a button that does nothing.

  Two distinctions are made visually rather than in prose. A value the operator
  owns is an input; a value the panel only knows is plain text against a rule,
  never a field that looks editable. And a setting the service reads only at
  start carries a **restart** mark beside the field - shown next to the control,
  because by the time the value has been typed the sentence under it has been
  read and forgotten. The log level is the exception on the Addresses tab: it
  applies at once, which is the reason it is there.

The **search is performed on the service node**, not in the browser: the panel
passes `q`, `state`, `limit` and `offset` through and displays the result, so a
node with many accounts does not transfer the whole set to locate one.

| Panel route | Passed through to |
|---|---|
| `GET /api/services` | each service's `/healthz` |
| `GET /api/users?q&state&limit&offset` | service-node `GET /v1/account/users` |
| `GET /api/account?user=` | server-core `GET /v1/messaging/account` (mailbox usage and device count) |
| `POST /api/approve` | service-node `POST /v1/account/approve` |
| `PUT /api/quota` | service-node `PUT /v1/account/quota` |
| `POST /api/undelegate`, `POST /api/delete`, `POST /api/ban`, `POST /api/unban` | the matching service-node operator endpoints |
| `GET /api/settings`, `PUT /api/settings` | service-node `/v1/account/settings`, plus server-core `GET /v1/messaging/storage` for the disk figure |
| `GET /api/panel`, `PUT /api/panel` | nothing - the panel's own configuration file |
| `POST /api/panel/restart` | nothing - the panel's own process |
| `GET /api/service-settings`, `PUT /api/service-settings` (`?kind=` for server-core or service-node, `?service=<name>` for anything) | server-core `/v1/messaging/settings`, service-node `/v1/account/settings`, a facade's or a gateway's `/settings` (ops socket) |
| `POST /api/restart` (same) | server-core `POST /v1/messaging/restart`, service-node `POST /v1/account/restart`, a facade's or a gateway's `POST /restart` (ops socket) |
| `GET /api/facades` | each facade's `GET /settings` for its address, and the node's `GET /v1/account/settings` for the descriptor |
| `POST /api/facade/new-address?service=<name>` | that facade's `POST /new-address`, then the node's `PUT /v1/account/settings` and every gateway's `GET`/`PUT /settings` (below) |

### Facade addresses and the descriptor

A facade mints its own address - its I2P destination - and reports it
(`Facade.md`); the operator never types one. The panel is what puts them into
the server descriptor: it asks every facade of the deployment for its address
and writes the list, in the order the facades are listed on the Panel tab,
into the node's `facades` (`PUT /v1/account/settings`). It does so whenever that
list is saved and whenever a facade is given a new address. If a facade does not
answer, the descriptor is left as it was and the panel says which facade it was:
a descriptor missing a working facade is worse than one that is a step behind.
A deployment with no facade keeps the addresses it had, and the panel says that
too. An address that changes behind the panel's back - a facade's `state`
wiped by hand - shows on the Facades tab as **not in the descriptor** until the
list is saved again.

**New address** replaces a facade's key and restarts it (`POST /new-address`).
The panel takes the new address from that answer rather than
asking the facade, which is restarting, writes the descriptor, and then links
the deployment's gateways again (below), so the route to this facade names the
new destination. Otherwise a gateway that reached this facade without I2P would
keep a route to an address nobody serves, and its clients would quietly fall back
to I2P. The button says what it costs: a client that knows only the old address
loses the server until it is given the new descriptor.

### Gateways and the server

The panel links every gateway of a deployment to its server-core and its
facades; nothing of it is typed by hand. Over the ops sockets it writes:

- into server-core (`PUT /v1/messaging/settings`): `gateways`, the key of every
  gateway of the deployment, as each is pinned on the Panel tab;
- into each gateway (`PUT /settings`): `server`, server-core's `gatewayListen`
  and key, and `direct`, one route for every facade with a `directListen`: the
  facade's destination, that socket and the facade's key.

A listener bound to every interface is reached at the host the panel reaches
that service on. The panel writes these whenever the service list, server-core's
settings or a facade's settings are saved, and when a facade is given a new
address; a value already in place is not written again. A server-core without a
`gatewayListen` leaves its gateways linked to the facades only. Traffic goes
from the gateway to server-core and to the facades directly
(`bazarish-server/gateway/PROTOCOL.md`, The server link): the panel is not in
it.

The quota is set **on the service node**, which is the component that grants it and
pushes it to server-core. Setting it directly on server-core, as an earlier
revision of the panel did, is overwritten by the node's next push.

The panel serves its web files from `assets <dir>` (mandatory): the page shell,
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
  "state": "/var/lib/bazarish/admin-panel",
  "assets": "/opt/bazarish/bazarish-server/admin-panel/web",
  "listen": {"host": "127.0.0.1", "port": 8460},
  "fleets": [
    {"name": "home", "services": [
      {"name": "server-core", "kind": "server-core",
       "url": "https://127.0.0.1:8428", "pin": "<key server-core prints>"},
      {"name": "service-node", "kind": "service-node",
       "url": "https://127.0.0.1:8427", "pin": "<key service-node prints>"},
      {"name": "facade", "kind": "facade",
       "url": "https://127.0.0.1:8429", "pin": "<key facade-http prints>"},
      {"name": "gateway", "kind": "gateway",
       "url": "https://127.0.0.1:8439", "pin": "<key the gateway prints>"}
    ]}
  ]
}
```

Each service is reached on its operator socket over TLS and is named by the key
it prints at startup (`pin`); an `http` URL or a missing pin is refused. A
private gateway is reached the same way: the panel shows its health and edits its
tunnels and warm pools. Names are unique within a deployment, and a deployment
holds one `server-core` and one `service-node` at most; a configuration that
breaks either is refused at start and on save. An optional `consumer` (`name`,
`place`, `role`) sets the words the panel signs under.

The configuration contains no secrets. The panel prints its own operator
fingerprint at startup; that fingerprint is added to each service's
`operators`, and the socket pin it prints beside it to each service's
`opsClients`.

Everything above is also editable from the Panel tab, and the panel writes it
back into this same file one value at a time - the operator's key order, spacing
and comments survive a save.

Between them, the tabs cover every value in the four fleet config files except the
ones that would let the panel lock itself out or move a service's data: the state
and asset directories, the operator fingerprint lists, and a server's own
identity. Those are shown read-only. **Everything else an operator would edit
over SSH is editable here, and every service can be restarted from here**, which
is the rule this page is built to: managing a Bazarish server should not require a
shell on it. A gateway's file is the exception in the other direction: the panel
edits its tunnels and pools, writes its link and routes itself, and leaves the
rest - the secret path and the token above all - to the operator, and never reads
them.

## First run

A fresh install has no `admin` key at all (or an empty one), and that is not an
error: it is an **unclaimed** panel. In that state `/` and `/login` redirect to
`/setup`, which asks for one signature and assigns the panel to the key that
provides it. The panel says so on its console at startup, because that is where
the person who installed it is looking:

```
no administrator configured yet: open http://127.0.0.1:8460/setup and sign in -
the first key to sign there administers this panel
```

The claim is first-come, checked and written under one lock, so two people
opening the wizard at the same moment cannot both become the owner. The moment
it succeeds `/setup` stops answering and redirects to `/login` permanently;
there is no second wizard and no way back to it except editing the config file
manually. **The window is open from the first start until that first
signature**, so a panel that will be reachable from outside should be claimed
before it is published, or published only after it is claimed.

Removing the last administrator through the panel is refused for the same
reason: an empty list would reopen the wizard to whoever found the page next.
