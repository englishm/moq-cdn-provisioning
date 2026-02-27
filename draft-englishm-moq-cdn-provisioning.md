---
title: "MoQ CDN Provisioning"
abbrev: "moq-cdn-prov"
category: info

docname: draft-englishm-moq-cdn-provisioning-latest
submissiontype: IETF
number:
date:
consensus: true
v: 3
area: "Web and Internet Transport"
workgroup: "Media Over QUIC"
keyword:
 - moq
 - cdn
 - provisioning
 - relay
venue:
  group: "Media Over QUIC"
  type: "Working Group"
  mail: "moq@ietf.org"
  arch: "https://mailarchive.ietf.org/arch/browse/moq/"
  github: "englishm/moq-cdn-provisioning"
  latest: "https://englishm.github.io/moq-cdn-provisioning/draft-englishm-moq-cdn-provisioning.html"

author:
 -
    fullname: "Mike English"
    organization: Cloudflare
    email: "ietf@englishm.net"

normative:
  MOQT: I-D.ietf-moq-transport

informative:

--- abstract

This document describes concepts
related to provisioning MoQ relay scopes on CDN infrastructure,
including scope creation, credential-to-scope mapping,
and origin fallback configuration.
It uses a provisioning API as a vehicle
for describing these concepts
and identifying areas
where common semantics across CDN providers
may be needed for multi-CDN compatibility.

--- middle

# Introduction

Media over QUIC Transport (MoQT) {{MOQT}}
defines a pub/sub protocol for media delivery through relays.
CDN providers that deploy MoQ relays
need to configure them in ways
that are roughly analogous to
the rewrite rules, origin selection,
and routing configuration
associated with HTTP reverse proxies and CDNs.
Some of these configurations
will need common semantics across providers
to support multi-CDN deployments.

This document uses a provisioning API
as a vehicle for describing these concepts.
A customer creates a scope on a CDN relay,
gets back connection credentials,
and hands those credentials
to their publishers and subscribers.
The relay uses the credentials
to map incoming connections to the right scope.

The API itself is part of the picture,
but the more important contribution here
is describing the underlying concepts
(scopes, credential-to-scope mapping, origin fallback)
in a way that could be consistent across CDN providers.

A machine-readable OpenAPI description of the API described here
is maintained alongside this document
in the source repository (see openapi.yaml).

# Conventions and Definitions

{::boilerplate bcp14-tagged}

Scope:

: An isolated MoQ delivery context on a relay,
  as defined in {{MOQT}}.
  All namespaces and tracks within a scope
  are isolated from other scopes on the same relay.
  A scope is the resource created by the provisioning API.

# Scope Provisioning

A CDN provider exposes an HTTP API endpoint
for creating scopes:

~~~
POST /moq/scopes
~~~

The request MAY include configuration
(see {{origin-fallback}}).

The response includes
a server-generated scope identifier
and connection credentials:

~~~json
{
  "scope_id": "a1b2c3d4e5f6",
  "url": "moqt://relay.example.com",
  "token": "eyJhbGciOiJFZDI1NTE5..."
}
~~~

The `url` is the base URL of the relay service.
The `token` is a credential
that grants access to this scope.

# Connecting to a Scope {#connecting}

Clients connect to a provisioned scope
by combining the URL and token
returned at creation time:

~~~
moqt://relay.example.com/{token}
~~~

The relay extracts the token from the URL path,
validates it,
and maps the connection
to the corresponding MoQT scope.

Within that scope,
all MoQT operations (SUBSCRIBE, ANNOUNCE, etc.)
are isolated.
Publishers and subscribers in one scope
cannot see namespaces or tracks from another scope.

# Origin Fallback {#origin-fallback}

A scope MAY be configured
with an upstream origin URL.
When a subscriber requests content
that isn't available on the relay,
the relay connects to the origin to fetch it.

This is configured at provisioning time:

~~~json
{
  "config": {
    "origin_fallback": {
      "url": "https://origin.example.com"
    }
  }
}
~~~

The relay establishes a MoQT connection to the origin URL
and forwards the subscription.
The origin could be another relay
(at a different CDN provider,
or a customer's own infrastructure),
or any MoQT-speaking endpoint.

# Security Considerations

TODO

# IANA Considerations

This document has no IANA actions.

--- back

# Acknowledgments
{:numbered="false"}

Thanks to Lucas Pardue and Jacob Curtis
for design input.
