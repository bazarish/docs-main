# Threat model

This document names what Bazarish protects, who can act against it, and what each
of them learns and can do. The mechanisms are specified in the documents it
links.

## Assets

- **Content**: message bodies, pictures, voice notes, files and call audio.
- **The contact graph**: who corresponds with whom.
- **Co-location**: which users one server hosts.
- **Network location**: the IP address of a user's device.
- **Keys and local state**: the identity keys and the account database on a device.

## Assumptions

- The primitives in `README.md`, Cryptography, are secure as OpenSSL implements
  them.
- A device runs an unmodified client and is under its user's control.
- I2P hides which router operates a destination from anyone who cannot observe
  enough of the network to correlate traffic at both ends of a tunnel.
- A server operator is relied on to store mail and hand it over. The operator is
  not trusted with content, the contact graph or co-location.

## A network observer

An observer of a device's link sees I2P transport traffic. A device without a
network database also fetches a reseed over https (`api/Facade.md`). The observer
does not see which destinations the device talks to or what it carries.

An observer can block I2P or the reseed. The router can reach I2P through a SOCKS5
proxy, and a device can start from any https reseed it is given
(`client-qt/docs/I2pRouter.md`).

## I2P routers

A router that carries a tunnel sees encrypted traffic between its neighbours.
Every published destination is a b33, so a floodfill holds only encrypted lease
sets and cannot list the destinations behind them (`Identity.md`). Tunnel length
is the user's setting; call media always uses one hop (`Architecture.md`, decision
6b).

## The facade

The facade relays one sealed tunnel. It learns the flow of each device: a stable
source address, the time of each request and its padded size. It does not learn
whose flow it is or what it carries (`api/Facade.md`, What the facade still
knows). It holds no key of the server or of a user.

## The user's own server

The messaging server and the service node serve the account. They learn:

- the account's fingerprint, its contact card, and when it registered and
  delegated its destination;
- the destination they operate for it, with that destination's serving sealing
  key and signing key;
- the number of the account's devices and of the delivery passes it holds, which
  is its number of contacts;
- the arrival time, size and delivery class of every item in its mailbox;
- that a pass registered right after a contact request belongs to that request
  (`Contacts.md`, Residual disclosure to the server).

They do not learn content. A delivery envelope names a mailbox and a pass and
never the sender (`api/Federation.md`). The server initiates no connections and
carries no outgoing mail, so it does not learn whom the user writes to
(`Architecture.md`, Trust boundaries).

The server can refuse service, delay or drop mail, and confirm a delivery it then
discards. A read receipt is the only end-to-end proof of arrival (`Messages.md`).
The server cannot publish anything in the user's name: the contact card carries
the user's signature alone.

A user opens the service node's portal in a browser to finish registration on a
moderated server. The portal sees what the browser's connection shows
(`api/ServicePortal.md`).

## A correspondent's server

When the user writes to a contact, the contact's server learns that an item
arrived from a destination the sending client holds for that one correspondent,
with that correspondent's pass. It can link the items that present one pass. It
does not learn who sent them (`api/Federation.md`). A contact request arrives from
an address that has carried nothing before it.

## Several servers together

Nothing a contact receives is shared by two users of one server, so cards and
invites collected from several users do not show that one server hosts them
(`api/InviteAnonymity.md`). Two cases are known:

- A group host delivers to all its members from one address. A server that holds
  several members learns that they share a group, not which group (`Groups.md`).
- A gateway linked to its operator's server lets its users tell from latency
  which recipients that server hosts (`api/InviteAnonymity.md`, No local
  delivery).

## A gateway

A gateway is optional and chosen by its user. It carries the client's I2P traffic
and sees the client's IP address and every destination the client dials and
publishes (`bazarish-server/gateway/PROTOCOL.md`, Trust). A client that runs its
own router uses no gateway.

## A contact

A contact holds the user's descriptor and card. It may write within its pass, at
most 120 messages a minute (`Contacts.md`). The user stops a contact by revoking
its pass. A contact listed as a presence reader sees the user's status and typing
while it watches (`Presence.md`, What anyone learns). The card names no server, so
a contact cannot tell which server hosts the user.

## Anyone with a descriptor

A descriptor is enough to fetch the card and send a contact request. Requests are
bounded per destination: 3 a minute, 30 waiting, and one a day from each sending
address (`api/Federation.md`, Limits). A destination answers a card fetch for its
own user only (`Contacts.md`).

## The alias registry

The registry binds names to fingerprints. It sees the name of each lookup, asked
from a one-time destination of the asker (`api/FederatedResolve.md`). Owners pay
it, so it holds a payment record for each owner.

The registry cannot point a name at an identity that did not sign for it. A name
with a word resolves only with that word. The registry is not on the delivery
path: a contact added once is reached without it.

## A site that accepts sign-in with the key

A site the user signs in to with the account key learns the account's fingerprint
(`api/SignInWithKey.md`). Two such sites can link their visitors by it.

## A group host

The host of a group knows every member's fingerprint and name and every post with
its author. Authorship inside a group is the host's statement. Members see the
member count and each other's names in the chat (`Groups.md`).

## A device in other hands

Every device of an account holds the same identity keys (`DeviceSync.md`). A device
in other hands keeps the identity. Retiring it stops deliveries to it, and the
identity itself cannot be revoked. The account database is SQLCipher, opened with a
key sealed under the user's passphrase by Argon2id
(`client-qt/docs/AccountStorage.md`).

Pairing a device takes a one-shot destination and a four-digit code. Ten wrong
codes take the destination down (`DeviceSync.md`, Enrollment).

## Outside this model

- An adversary that observes enough of I2P to correlate a device's traffic with
  the destinations it uses.
- A modified client, or a device controlled by someone other than its user.
