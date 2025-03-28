---
layout: post
title:  "How artifacts are signed in Fedora"
date:   2025-03-28 15:39:00 -0400
categories: blog fedora signing
---

For the last few months, one of the things I've been working on in Fedora is
adding support for SecureBoot on Arm64. The details of that work will be the
subject of a later post, but as part of this work I've become somewhat familiar
with the signing infrastructure in Fedora and how it works. This post
introduces the various pieces of the current infrastructure, and how they fit
together.


## Signed?

Pretty much anything Fedora produces and distributes is digitally signed so
users can verify it did, in fact, come from the Fedora project. Perhaps the
most obvious example of this is the RPM packages Fedora produces. However,
plenty of other artifacts are also signed, like OSTree commits.

Signing works using [public-key
cryptography](https://wikipedia.org/wiki/Public-key_cryptography). We have the
private key that we need to keep secret, and we distribute the public keys to
users so they can verify the artifact.


## Robosignatory

Signing is (mostly) an automated process. A service, called
[robosignatory](https://pypi.org/project/robosignatory/), connects to an AMQP
message broker and subscribes to several message topics. When an artifact is
created that needs signing, the creator of the artifact sends a message to the
AMQP broker using one of these topics.

Robosignatory does not sign artifacts itself. It collects requests from various
other services and submits them to the signing server on behalf of those
systems. The signing server is the system that contains and protects the
private keys.


## Sigul

[Sigul](https://pagure.io/sigul) is the signing server that holds the private
keys and performs signing operations. It is composed of three parts: the
client, the bridge, and the server.

### The Server

The Sigul server is designed to control access to signing keys and to provide a
minimal attack surface. It does not behave like a traditional server in that it
does not accept incoming network connections. Instead, it connects to the Sigul
bridge, and the bridge forwards requests to it via that connection.

This allows the firewall to be configured to allow outgoing traffic to a single
host, and to block all incoming connections. This does mean that the host
cannot be managed via normal SSH operations. Instead this is done in Fedora via
the host's out-of-band management console.

Private keys are encrypted on disk. When users are granted access to keys, the
key is encrypted for that particular user.

### The Bridge

The Sigul bridge acts as a specialized proxy. Both the client and server connect
to the bridge, and it forwards requests and responses between them.

Unlike the typical proxy, the Sigul bridge does a few interesting things.
Firstly, it requires the client to authenticate via TLS certificates. The
client then sends the bridge the request it wishes to make. The bridge forwards
that request to the server. The bridge then expects the client and server to
send an arbitrary amount of data to each other. This arbitrary data is framed
as chunks and it forwards that data back and forth until an end of stream
signal is sent by the server and client.

Finally, it forwards the server's response to the client.

### The Client

The client is a command-line interface that sends requests to the server via
the bridge. This client can be used to create users and keys, grant and revoke
access to keys, and request signatures in various formats. Robosignatory
invokes the CLI to sign artifacts.

The client connects to the bridge as described above. After authenticating with
the bridge and sending the request, it begins a _second_ TLS connection
configured with the Sigul server's hostname, and sends and receives data on
this connection over the TLS connection it has with the Sigul bridge. It relies
on this inner TLS connection to send a set of session keys to the server
without the bridge being able to intercept them. These session keys are used to
sign the Sigul server's responses so the client can be sure the bridge did not
tamper with them.

After it has established a set of secrets with the server, the remainder of the
interaction occurs over the TLS connection with the bridge, but since the
responses are signed both the client and server can be confident the bridge has
not tampered with the conversation.


## Conclusion

It's an interesting setup, and has served Fedora for many years. There is
plenty of code in Sigul which dates back to 2009, not long after Python 2.6 was
released with exciting new features like the standard library json module. It
was likely developed for RHEL 5 which means Python 2.4.

Unfortunately, at least one of the libraries (python-nss) it depends on to work
are not maintained and not in Fedora 42, so something will have to be done in
the near future. Still, it's not ready to sign off just yet.
