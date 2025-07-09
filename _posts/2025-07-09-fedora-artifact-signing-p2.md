---
layout: post
title:  "Re-designing signing in Fedora"
date:   2025-09-09 12:17:00 -0400
categories: blog fedora signing
---

Over the past few months I've spent some time on-and-off working on
[Sigul](https://pagure.io/sigul) and some related tools. In particular, I
implemented most of a new [Sigul
client](https://github.com/fedora-infra/siguldry/tree/siguldry-0.3.1/siguldry),
primarily to enable the sigul-pesign-bridge to run on recent Fedora releases
(since the sigul client relies on python-nss, which is not in Fedora anymore).

At this point, I have a reasonably good understanding of how Sigul works.
Originally, my plan was to completely re-implement the client, then the bridge,
and finally the server using the existing Sigul protocol, version 1.2, as
defined by the Python implementation. However, as I got more familiar with the
implementation, I felt that it would be better to use this opportunity to also
change the protocol. In this post I'm going to cover the issues I have with
the current protocol and how I'd like to address them.


## Mixing TLS sessions

In protocol version 1.2, the client and server start "outer" TLS sessions with
the bridge, and then the client starts a nested "inner" TLS session with the
server. Data is sent in chunks which indicate how big the chunk is and whether
it's part of the "outer" session (and destined for the bridge) or the "inner"
session. While it's perfectly doable to parse the two streams out, it's a
complication. Maybe we can introduce some rules to make it easier?

After looking at the implementation, every command follows the same pattern. The client would:

1. Open a connection to the bridge and send the _bridge_ the command to pass on to the server.
2. Open the inner TLS session and send some secrets to the server (a key to use
   for HMAC and a key passphrase to unlock a signing key, typically).
3. Close the inner TLS session.
4. Send HMAC-signed messages to the bridge, which it relays to the server.
5. Receive HMAC-signed messages from the server, via the bridge.

Critically, the inner TLS session was only used to exchange secrets so the
bridge couldn't see them, and was never used again. One option would be to only
allow the inner TLS session once, right at the beginning of the connection.

However, the whole point of the HMAC-signed messages is that the client and
server don't seem to really "trust" the bridge won't have tampered with the
messages. Why not just use the inner TLS session exclusively so that we get
confidentiality in addition to integrity?


## Homegrown serialization

When Sigul was originally written things like JSON weren't part of the Python
standard library. It implemented its own, limited format. Now, however, there
are a number of widely supported serialization options. JSON is the obvious
choice as it is fairly human-readable, ubiquitous, and fairly simple. The
downside is that for signing requests, the client needs to send some binary
data. However, we explicitly do not want to be sending huge files to be signed,
so base64-encoding small pieces of binary data should be acceptable.


## The bridge is too smart

The bridge includes features that are not used, and that complicate the
implementation.

### Fedora Account System integration

The bridge supports configuring a set of required Fedora Account System groups
the user needs to be in to make requests. It checks the user against the
account system when it connects by using the Common Name in the client
certificate as the username.

However, this feature is [not used by
Fedora](https://github.com/fedora-infra/ansible/blob/cf00289c066041ad32d0a8147114df54ec2bb616/roles/sigul/bridge/templates/bridge.conf.j2#L15),
and since Fedora is probably the only deployment of this service, we probably
don't need this feature.

### The bridge alters commands

For most commands, the bridge shovels bits between the client connection and
the server connection. However, before it shovels, it parses out the requests
and responses before forwarding them. There's really only one good reason for
this. Two particular commands _may_ alter the client request before sending it
to the server. Those two commands are "sign-rpm" and "sign-rpms".

In the event that the client requests a signature for an RPM or multiple RPMs
and the request doesn't include a payload, the bridge will download the RPM
from Koji. Now, the bridge doesn't have the HMAC keys used in other commands to
sign the request, so the client _also_ contacts Koji and includes the RPM's
checksum in the request headers.

This particular design choice might have been done to save a hop when
transferring large RPMs, but these days you don't need to send the whole RPM,
just the header to be signed. 

It's confusing to have the bridge take an active role in client requests.
What's more, if we push this responsibility to the client, there's no reason
the bridge needs to see the requests and responses at all.

### Usernames

As noted in the Fedora Account System integration section, the bridge uses the
client certificate's Common Name to determine the username. However, all
requests _also_ include a "user" field.

The server checks to ensure either the request username matches the client
certificate's Common Name, or if the `lenient_username_check` configuration
option is set which disables that check, or if the Common Name is in a
configuration option, `proxy_usernames`, listing that that Common Name can use
whatever username it wants.

The [Fedora
configuration](https://github.com/fedora-infra/ansible/blob/cf00289c066041ad32d0a8147114df54ec2bb616/roles/sigul/server/templates/server.conf.j2#L1)
doesn't define either of those configuration options, and it's confusing to
have two places to define the username.

### Passwords

Users have several types of passwords.

Users are given access to signing keys by setting a user-specific passphrase
for each key. This passphrase is used to encrypt a copy of the "real" key
password, so each user can access the key without ever knowing what the key
password is.

However, each user also has an "account password" which is only needed for
admin commands, and only works if the account is flagged as an admin. Given
that the client certificate can be password-protected it's not clear to me that
this adds any value, but it is confusing.


## Summary

To summarize, the major changes I'm considering in a new version of the Sigul protocol are:

- *All* client-server communication happens over the nested TLS session. The
  client and server no longer need to manually HMAC-sign requests and responses
  as a result.

- The bridge is a simple proxy that authenticates the client and server
  connections via mutual TLS and then shovels bits between the two connections
  without any knowledge of the content. Drop the Fedora Account System
  integration, and push the responsibility of communicating with Koji to the
  client.

- Switch from the homegrown serialization format to JSON for requests and responses.

- Rely exclusively on the client certificate's Common Name for the username.

- Remove the admin password from user accounts as they can password-protect
  their client key and also use features like systemd-creds to encrypt the key
  with the host's TPM.
