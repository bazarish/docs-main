# Admin panel

The `admin-panel` is one web interface to monitor and manage a Bazarish fleet
(server-core, service-node, facade-http, alias-resolver). It is a
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
| alias-resolver | admin-panel |

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

## Management

The panel drives the operator endpoints the services already provide:

| Action | Endpoint | Services |
|---|---|---|
| Set a user's mailbox quota | `PUT /v1/messaging/quota` | server-core |
| Provision/revoke a user's personal I2P destination | `PUT /v1/messaging/i2p-dest` | server-core |

## Configuration

```json
{
  "admin": "<admin hybrid fingerprint, base32>",
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
