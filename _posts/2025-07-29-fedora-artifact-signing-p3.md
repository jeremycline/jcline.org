---
layout: post
title:  "Fedora signing protocol tweaks"
date:   2025-07-29 10:01:00 -0400
categories: blog fedora signing
---

In [my last post]({% post_url 2025-07-09-fedora-artifact-signing-p2 %}) on
Fedora's signing infrastructure, I ended with some protocol changes I would be
interested in making. Over the last few weeks, I've tried them all in a
proof-of-concept project and I'm fairly satisfied with most of them. In this
post I'll cover the details of the new protocol, as well as what's next.

## Sigul Protocol 2.0

The major change to the protocol, as I mentioned in the last post, is that all
communication between the client and the server happens over the nested TLS
session. Since the bridge cannot see any of the traffic, its role is reduced
significantly and it is now a proxy server that requires client authentication.

I considered an approach where there was no communication from the client or
server to the bridge directly (beyond the TLS handshake). However, adding a
handshake between the bridge and the server/client makes it possible for all
three to share a few pieces of data useful for debugging and error cases. While
it feels a little silly to add this complexity primarily for debugging
purposes, the services are designed to be isolated and there's no opportunity
for live debugging.

### The handshake

After a server or client connects to the bridge, it sends a message in the
outer TLS session to the bridge. The message contains the protocol version the
server/client will use on the connection, as well as its role (client or
server). The bridge listens on two different ports for client and server
connections, so the role is only included to catch mis-configurations where the
server connects to the client port or vice versa.

The bridge responds with a message that includes a status code to indicate it
accepts the connection (or not), and a UUID to identify the connection.

The bridge sends the same UUID to both the client and server so it can be used
to identify the inner TLS session on the client, bridge, and server. This makes
it easy to, for example, collect logs from all three services for a connection.
It also can be used during development with OpenTelemetry to [trace a
request](https://opentelemetry.io/docs/concepts/signals/traces/) across all
three services.

Reasons the bridge might reject the connection include (but is not limited
to): the protocol version is not supported, the role the connection announced
is not the correct role for the port it connected to, or the client certificate
does not include a valid Common Name field (which is used for the username).

After the bridge responds with an "OK" status, servers accept incoming inner
TLS session on the socket, and clients connect. All further communication is
opaque to the bridge.

### Client/Server

In the last blog post, I discussed requests and responds being JSON
dictionaries, and content that needed to be signed could be base64 encoded, but
I didn't go into the details of RPM header signatures. While I'd still love to
have everything be JSON, after examining the size of RPM headers, I opted to
use a request/response format closer to what Sigul 1.2 currently uses. The
reason is that base64-encoding data increases its size by around 33%. After
poking at a few RPMs with the
[rpm-head-signing](https://github.com/fedora-iot/rpm-head-signing) tool, I
concluded that headers were still too large (cloud-init's headers were hundreds
of kilobytes) to pay a 33% tax for a _slightly_ simpler protocol.

So, the way a request or response works is like this: first, a frame is sent.
This is two unsigned 64-bit integers. The first `u64` is the size of the JSON,
and the second is the size of the arbitrary binary payload that follows it.
What that binary is depends on the command specified in the JSON. The
alternative was to send a single `u64` describing the JSON size alone, so the
added complexity here is minimal. The binary size can also be 0 for commands
that don't use it, just like the Sigul 1.2 protocol.

Unlike Sigul 1.2, none of the messages need HMAC signatures since they all
happen inside the inner TLS session. Additionally, the protocol does not allow
any further communication on the outer TLS session, so the implementation to
parse the incoming requests is delightfully straightforward.

### Authentication

Both the client and server authenticate to the bridge using TLS certificates.
Usernames are provided in the Common Name field of the client certificate. For
servers, this is not terribly useful, although the bridge could have an
allowlist of names that can connect to the server socket.

Clients and servers _also_ mutually authenticate via TLS certificates on the
inner TLS session. Technically, the server and client could use a separate set
of TLS certificates for authentication, but in the current proof-of-concept it
uses the same certificates for both authenticating with the bridge and with the
server/client. I'm not sure there's any benefit to introducing additional sets
of certificates, either.

For clients, commands no longer need to include the "user": it's pulled from
the certificate by the server. Additionally, there's no user passwords. Users
that want a password in addition to their client certificate can encrypt their
client key with a password. If this doesn't sit well with users (mostly Fedora
Infrastructure) we can add passwords back, of course.

Users _do_ still need to exist in the database or their requests will be
rejected, so administrators will need to create the users via the command-line
interface on the server or via a client authenticated as an admin user.

### Proof of concept

Everything I've described has been implemented in my [proof-of-concept
branch](https://github.com/jeremycline/siguldry/tree/sigul-protocol-2). I'm in
the process of cleaning it up to be merge-able, and command-line interfaces
still need to be added for the bridge and client. There's plenty of TODOs still
sprinkled around, and I expect I'll reorganize the APIs a bit. I've also made
no effort to make things fast. Still, it will happily process hundreds or
thousands of connections concurrently. I plan to add benchmarks to the test
suite prior to merging this so I can be a little less handwave-y about how
capable the service is.


## What's next?

Now that the protocol changes are done (although there's still time to tweak
things), it's time to turn our attention to what we actually want to do:
managing keys and signing things.

### Management

In the current Sigul implementation, some management is possible remotely using
commands, while some are only possible from on the server. One thing I would
like to consider is moving much of the uncommon or destructive management tasks
to be done locally on the server rather than exposing them as remote commands.
These tasks could include:

- All user management (adding, removing, altering)
- Removing signing keys

Users could still do any non-destructive actions remotely. This includes
creating signing keys, read operations on users and keys, granting and revoking
access to keys for other users, and of course signing content.

Moving this to be local to the server makes the permission model for requests
simpler. I would love feedback on whether this would be inconvenient from
anyone in the Fedora Release Engineering or Infrastructure teams.

### Signing

When Sigul was first written, Fedora was not producing so many different kinds
of things that needed signatures. Mostly, it was RPMs. Since then, many other
things have needed signatures. For some types of content (containers, for
example), we still don't sign them. For other types, they aren't signed in an
automated fashion.

Even the RPM case needs to be reevaluated. RPM 6.0 is on the horizon with a new
format that supports multiple signatures - something we will want to support
post-quantum algorithms. However, we still need to sign the old v4 format for
EPEL 8.0. We probably don't want to continue to shell out to `rpmsign` on the
server.

Ideally, the server should not need to be aware of all these details. Instead,
we can push that complexity to the client and offer a few signature types. We
can, for example, produce SecureBoot signatures and container signatures [the
same way](https://github.com/fedora-infra/siguldry/issues/49). I need to
understand the various signature formats and specifications, as well as the
content we sign. The goal is to offer broad support without requiring
constantly teaching the server about new types of content.


## Comments and Feedback

Thoughts, comments, or feedback greatly welcomed on [Mastodon](https://hachyderm.io/@jcline/114937982823195105)
