---
layout: post
title:  "Fedora Signing Update, September edition"
date:   2025-09-18 11:34:00 -0400
categories: blog fedora signing
---

It's been a while since [my last update]({% post_url
2025-07-29-fedora-artifact-signing-p3 %}) about content signing in Fedora, so I
figured I should write one before the end of this month. While I've not been
able to devote all my time to working on Siguldry, I have made enough progress
I think it's worth covering, and I also had the opportunity to chat with
Miloslav Trmač, the original author of Sigul, about its design.


## History Lesson for Sigul

In my post about [various protocol tweaks]({% post_url
2025-07-09-fedora-artifact-signing-p2 %}) I covered most of the changes I
wanted to make to Sigul's protocol and my reasons for doing so. A major theme
of the changes was pushing more of the content-related "smarts" to the client.
For example, in sigul, the server receives a container file and prepares a JSON
document based off it to sign with GPG. In my proposed approach, the client
should handle that and simply asks the server to sign the given document.

Miloslav was able to provide me with some history on Sigul and why it pushes
the responsibility of dealing with the content to the bridge and server, and I
think it's worth recording here. In the original design, signing didn't happen
automatically: a Fedora contributor would need to request a signature using the
Sigul client. In this scenario, there are a _lot_ more Sigul clients and
they're running on machines that Fedora infrastructure doesn't manage. Thus,
the bridge and server needed to make every effort to ensure clients didn't
request signatures for things they shouldn't - this is why the bridge is able
to integrate with the Fedora Account System, and why it also contacts Koji when
RPM signatures are requested: it has every reason to expect the client to be
malicious.

These days, there's only a few Sigul clients. Fedora's infrastructure admins
have management accounts, and the robosignatory service, which runs in Fedora's
infrastructure, also acts as a client. While we should be cautious in our
design, the deployment is significantly different from Sigul's original
expectations so we can make different trade-offs.

## Siguldry Progress

Back in July, I completed the protocol implementation. Since then, I've worked
on actually doing things on top of it.

### Binding Keys to Hardware

One feature I largely ignored in Sigul because it seemed complicated was its
ability to encrypt signing keys using a combination of user-provided passwords
and hardware (a Yubikey or TPM, for example). This means that if you were to
obtain a copy of the Sigul database and you also managed to get the user's
password to access a signing key, you still wouldn't be able to access the
private keys without getting access to the signing machine's TPM or a Yubikey
that Fedora's infrastructure team has access to.

I have partially re-implemented this particular feature in Siguldry. I say
partially because Sigul had a number of different ways to bind keys to hardware
on both the server and client side, and I have opted to implement only what
Fedora infrastructure currently uses. You can use any key accessible via PKCS
#11 to encrypt signing keys in addition to user passwords. You can even use
multiple keys, and any one key is enough to unlock things (so you can have a
backup Yubikey or three).

On the client side, the expectation is that user passwords and certificates
will be bound to the TPM via systemd-credentials.

### The Server Supports PGP Signing

I've implemented creating PGP keys as well as signing content with them. The
interface supports detached signatures, the
[cleartext](https://www.rfc-editor.org/rfc/rfc9580.html#section-7) format you
see, for example, with
[CHECKSUM](https://dl.fedoraproject.org/pub/fedora/linux/releases/42/Workstation/aarch64/iso/Fedora-Workstation-42-1.1-aarch64-CHECKSUM)
files, and inline signatures.

This covers all current PGP signature types used by Fedora. However, more work
is needed on the client side to use this interface to sign RPMs. The client
needs to extract the header from the RPM, have the server produce a detached
signature, and then send that signature to Koji.

### The Server Supports RSASSA-PKCS1-v1_5 and ECDSA Signatures

These are the signatures you get with `openssl-dgst` and related commands, and
they're used for Secure Boot signatures, container signatures with
[cosign](https://github.com/fedora-infra/siguldry/issues/49), and
[IMA](https://sourceforge.net/p/linux-ima/wiki/Home/).


## What's Next

Siguldry now has all the primitive interfaces to implement signing in the
client for various types of content (RPM, containers, git tags, etc). It's now
time to work on implementing those, figure out what server interfaces need
changing, and then add a whole lot of polish: a comprehensive end-to-end test
suite for all content types, API documentation, an admin guide, and a set of
tools to migrate the Sigul database to Siguldry.

## Comments and Feedback

Thoughts, comments, or feedback greatly welcomed on [Mastodon](https://hachyderm.io/@jcline/115226068075225348)
