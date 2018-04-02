---
layout: post
title:  "Fedora CI and Infrastructure Hackathon 2017"
date:   2017-05-18 12:36:13 -0400
categories: blog fedora
---

Last week, the Fedora Infrastructure folks joined forces with people from
CentOS, Fedora QE, Factory 2.0, and others to work on the design and
implementation of continuous integration for the Fedora Project in the
great [Fedora CI and Infrastructure Hackathon of 2017](
https://fedoraproject.org/wiki/CI_and_Infrastructure_Hackathon_2017).

This was especially interesting for me since I had not had the opportunity
to meet the majority of the people I've been working with for the last eight
months.

The work we did can be broadly split into two categories as the title
implies: general infrastructure work, and CI work specifically.


# Infrastructure

The general infrastructure mostly took the form of documentation. Fedora
Infrastructure had a set of Standard Operating Procedures (SOPs) used to
document how to maintain the various applications we use. I recently took
this documentation (which was already conveniently written in ReStructured
Text) and turned it into a [Sphinx](http://www.sphinx-doc.org/en/stable/index.html)
project. The project, [infra-docs](https://pagure.io/infra-docs), aims to
cover both developer and system administrator best practices.

As part of the Hackathon we worked on expanding the developer documentation
with two sections.

## Authentication Documentation

The first day of the hackathon we covered how authentication works in Fedora
Infrastructure with [Ipsilon](https://pagure.io/ipsilon/). The goal is to
move applications from the old OpenID 2.0 to OpenID Connect which is based
on OAuth 2.0.

The [documentation](https://pagure.io/infra-docs/pull-request/40) introduces
developers to OpenID Connect and points them to the various RFCs and specs
for in-depth reading. It also overs the set of libraries that should be used
for OpenID Connect so that all our applications use the same set of libraries.


## BDR Documentation

[Bi-directional replication](http://bdr-project.org/docs/stable/index.html) is
a multi-master replication system for PostgreSQL designed for replication
across geographically distributed clusters via asynchronous logical replication.

Our staging environment has a BDR cluster, but applications need to be aware of
(and designed for) BDR. The initial [developer documentation](
https://fedora-infra-docs.readthedocs.io/en/latest/dev-guide/db.html) for BDR in
Fedora were written as part of the hackathon.


# Continuous Integration

The second big focus of the hackathon for me was the application changes required
for the continuous integration effort in Fedora.

## A New FedMSG Relay

One task we needed to tackle was getting the [CentOS CI infrastructure](https://ci.centos.org/)
publishing [fedmsgs](https://fedmsg.readthedocs.io/en/latest/) that the Fedora infrastructure
could consume. We require that messages be signed using x509 certificates since messages are
transmitted without any other form of authenticity. The ultimate plan is for the Jenkins plugin
to sign the messages before initially sending them, but it cannot currently do that, so as a
temporary work-around I worked with [Brian Stinson](https://wiki.centos.org/BrianStinson) to
introduce a new type of fedmsg relay that [signs messages it relays](
https://github.com/fedora-infra/fedmsg/pull/409). This allows all messages coming out of
ci.centos.org to be signed as they're relayed out to the world.


## GreenWave

[GreenWave](https://pagure.io/greenwave/) (formally known as PolicyEngine) is a new web service
we're introducing to Fedora Infrastructure that allows users to define policies about what checks
need to pass before an artifact is considered "good". Policies are defined globally, per product
(e.g. Fedora 26) and for a product and components (e.g. python-requests in Fedora 26). For example,
we could define that all builds must pass a ``depcheck`` test to be considered "good".

This will be used by, among other things, Bodhi to automatically gate updates. Given that, we need
a prototype of the service sooner rather than later, so I started working on that.
