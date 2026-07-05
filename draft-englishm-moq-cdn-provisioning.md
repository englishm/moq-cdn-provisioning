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
    email: ietf@englishm.net

normative:
  MOQT: I-D.ietf-moq-transport
  CAT4MOQ: I-D.ietf-moq-c4m
  MOQDPOP: I-D.nandakumar-moq-generic-dpop-proof

informative:
  RFC9110:
  RFC9449:

--- abstract

This document describes concepts
related to provisioning MoQ Scopes on CDN infrastructure,
including scope creation,
MoQ Access Token validation key provisioning,
upstream fallback configuration,
and namespace authorization.
It uses a provisioning API as a vehicle
for describing these concepts
and identifying areas
where common semantics across CDN providers
may be needed.

--- middle

# Introduction

Media over QUIC Transport (MoQT) {{MOQT}}
defines a pub/sub protocol for media delivery through relays.
CDN providers that deploy MoQ relays
need a mechanism for applications
to provision delivery infrastructure
on their behalf.

This document describes
how an Application Provider
provisions a MoQ Scope on a CDN,
provides the CDN with the cryptographic keys
needed to validate MoQ Access Tokens,
and configures optional behaviors
such as upstream fallback.

The Application Provider mints MoQ Access Tokens
and distributes them to its clients.
The CDN validates those tokens
using public keys provisioned by the Application Provider.
This separation ensures
the CDN does not need to be involved
in per-client or per-session token creation,
reducing load, latency, and single points of failure.

A machine-readable OpenAPI description of the API described here
is maintained alongside this document
in the source repository (see openapi.yaml).

## Relationship to Accounts and Billing

Obtaining an account with a CDN provider,
establishing billing arrangements,
and authenticating to the provisioning API
are prerequisites outside the scope of this document.
This specification assumes
the caller has already been authenticated and authorized
by the CDN provider's account management system.

# Conventions and Definitions

{::boilerplate bcp14-tagged}

Application Provider:

: The entity (e.g., a conferencing service, a live streaming platform)
  that provisions MoQ Scopes on a CDN
  and mints MoQ Access Tokens for its clients.
  The Application Provider uses the provisioning API
  to set up delivery infrastructure
  and provides token validation keys to the CDN.

MoQ Scope:

: An isolated MoQ delivery context on a relay,
  as defined in {{MOQT}}.
  All namespaces and tracks within a MoQ Scope
  are isolated from other scopes on the same relay.
  A MoQ Scope is the primary resource
  managed by the provisioning API.

MoQ Access Token:

: A token presented by a MoQ client to a relay
  during session establishment
  via the MoQT Authorization Token parameter.
  MoQ Access Tokens are minted by the Application Provider,
  not by the CDN.
  Relays validate them using public signing keys
  provisioned for the scope.

Edge Relay:

: A relay that accepts connections from end-user clients
  (publishers and subscribers).
  It is the first relay a client connects to.

# MoQ Scope Provisioning

A CDN provider exposes an HTTP API endpoint
for creating MoQ Scopes:

~~~
POST /moq/scopes
~~~

The request MAY include configuration
(see {{upstream-fallback}}, {{auth-policy}}).

The response includes
a server-generated scope identifier
and a MoQT URL through which clients
can reach the relay infrastructure:

~~~json
{
  "scope_id": "a1b2c3d4e5f6",
  "url": "moqt://relay.example.com",
  "supported_token_types": ["c4m", "c4m+dpop-proof+cwt"]
}
~~~

The `url` field provides the MoQT endpoint
for this scope.
How the CDN routes traffic behind this URL
(anycast, regional steering, load balancing)
is an operational concern of the CDN provider
and is not configured through this API.

The `supported_token_types` array lists
the MoQ Access Token mechanisms the relay accepts.
CDN providers MUST support both `c4m` and `c4m+dpop-proof+cwt`.
See {{token-validation}}.

## Scope Lifecycle

The API supports the full lifecycle of a MoQ Scope:

~~~
POST   /moq/scopes              - Create
GET    /moq/scopes/{scope_id}   - Read configuration
PATCH  /moq/scopes/{scope_id}   - Update configuration
DELETE /moq/scopes/{scope_id}   - Delete (initiates drain)
~~~

When a scope is deleted,
the relay SHOULD send GOAWAY {{MOQT}}
to all connected clients
before terminating connections.

# Connecting to a Scope {#connecting}

Clients connect to a provisioned MoQ Scope
by presenting a MoQ Access Token
to the relay at the scope's MoQT URL.

## MoQ Access Token Presentation

The MoQT Authorization Token parameter {{MOQT}}
carries the MoQ Access Token
during session establishment.
The supported token types are:

- `c4m` — CAT for MoQT {{CAT4MOQ}}
- `c4m+dpop-proof+cwt` — C4M with DPoP proofs encoded as CWT {{MOQDPOP}}

## Scope Mapping

The relay determines which MoQ Scope
a connection belongs to
by extracting the `scope_id` claim
from the validated MoQ Access Token.
The relay validates the token first,
then maps to the scope.

This works identically over WebTransport
and raw QUIC, since the claim is carried
inside the MoQT Authorization Token parameter
after the QUIC handshake completes.
No scope information leaks to the network.

# MoQ Access Token Validation {#token-validation}

The Application Provider mints MoQ Access Tokens
and distributes them to its publishers and subscribers.
The CDN does not issue tokens.
Instead, the Application Provider provisions
one or more public signing keys
that the CDN uses to validate tokens.

## Signing Key Provisioning

The Application Provider registers
public signing keys for each MoQ Scope:

~~~
POST /moq/scopes/{scope_id}/signing-keys
~~~

~~~json
{
  "key_id": "key-2026-q3",
  "algorithm": "ES256",
  "public_key": "-----BEGIN PUBLIC KEY-----\nMFkwEwYHKoZIzj0CAQYIKoZIzj0DAQcDQgAE..."
}
~~~

Multiple keys MAY be provisioned simultaneously
to support high availability and key rotation.
The relay MUST accept tokens signed
by any active key for the scope.

Key management endpoints:

~~~
POST   /moq/scopes/{scope_id}/signing-keys           - Add a key
GET    /moq/scopes/{scope_id}/signing-keys           - List keys
DELETE /moq/scopes/{scope_id}/signing-keys/{key_id}  - Remove a key
~~~

When a key is removed,
the relay MUST reject tokens signed by that key.
Application Providers SHOULD provision a new key
before removing the old one
to avoid disruption during rotation.

## Token Structure

A MoQ Access Token is a CWT (CBOR Web Token)
signed with ES256.
The token carries MoQ Claims as defined in {{CAT4MOQ}}:

- `moqt`: An array of permitted scopes,
  each specifying allowed MoQT actions
  and namespace/track match patterns.
  The Application Provider uses this claim
  to express which operations (PUBLISH, SUBSCRIBE, etc.)
  and which namespaces a client is authorized for.
- `moqt-reval`: Optional revalidation interval
  for long-lived sessions.
- `cnf`: Confirmation claim binding the token to the client's key.
- `exp`: Token expiration time.

In addition to MoQ Claims,
the token MUST contain a `scope_id` claim
identifying the MoQ Scope.
The relay uses this claim
to map the connection to the correct scope.

The Application Provider mints tokens
with the minimum set of actions and namespace patterns
required by the client.

## Token Types {#c4m}

MoQ Access Tokens use C4M (CAT for MoQT) {{CAT4MOQ}}
for proof-of-possession within MoQT control messages.
C4M defines how tokens are presented,
how clients prove possession of their bound key,
and how relays validate both the token signature
and the client's proof.

CDN providers MUST support C4M
and additionally C4M with CWT-encoded DPoP proofs (`c4m+dpop-proof+cwt`).
DPoP provides per-request replay protection
by binding the credential
to a specific relay endpoint and timestamp.
The CWT-based DPoP proof encoding for MoQT
is defined in {{MOQDPOP}}.

Relays validate C4M tokens
using the Application Provider's provisioned public signing keys
and the client's bound key from the `cnf` claim.
The processing rules for token validation
and DPoP proof verification
are defined in {{CAT4MOQ}} and {{MOQDPOP}} respectively.

## Token Lifecycle

The Application Provider manages the full lifecycle
of MoQ Access Tokens:

- Minting tokens for each client session
  (e.g., per meeting in a conferencing application)
- Setting appropriate expiration times
- Rotating signing keys without service disruption

The CDN does not need to be informed
about individual tokens.
It validates tokens on presentation
using the provisioned public keys.

When a signing key is compromised,
the Application Provider removes it
from the scope's key set.
All tokens signed by that key
are immediately invalidated
across all relays serving the scope.

# Authorization Policy {#auth-policy}

A MoQ Scope MAY be configured with
namespace-level authorization rules
that restrict which operations
are permitted for a given MoQ Access Token.

~~~json
{
  "config": {
    "auth_policy": {
      "rules": [
        {
          "namespace_prefix": ["live", "meeting-42"],
          "operations": ["publish", "subscribe"]
        },
        {
          "namespace_prefix": ["live", "meeting-42", "screen"],
          "operations": ["publish"],
          "max_publishers": 1
        }
      ],
      "default_deny": true
    }
  }
}
~~~

When `default_deny` is true,
any operation on a namespace
not matching a rule is rejected.

Authorization rules are evaluated
using longest-prefix match:
the most specific matching rule applies.

The authorization policy configured at the scope level
defines the maximum permissions available.
Individual MoQ Access Tokens
carry their own namespace claims
which MUST be a subset of the scope's policy.
The relay enforces both:
the token's claims limit what the client requests,
and the scope's policy limits
what any token can authorize.

# Upstream Fallback {#upstream-fallback}

A MoQ Scope MAY be configured
with one or more upstream MoQT URLs
belonging to the Application Provider.
When a subscriber requests content
that isn't available on the relay,
the relay connects to the upstream endpoint to fetch it.

~~~json
{
  "config": {
    "upstream_fallback": {
      "url": "moqt://publish.app-provider.example.com",
      "namespaces": [{"prefix": ["live"]}]
    }
  }
}
~~~

The relay establishes a MoQT connection
to the upstream URL
and forwards the subscription.
The upstream endpoint is operated
by the Application Provider
(or colocated with a relay
that reads from the Application Provider's content source).

This is the mechanism by which
the CDN connects back to the Application Provider's
publishing infrastructure
when content is not already flowing
through the relay network.

# Security Considerations

## Provisioning API Security

The provisioning API endpoint
MUST require authentication and authorization.
It SHOULD use OAuth 2.0 bearer tokens
or mutual TLS for API authentication.
The API MUST be served over HTTPS.

## Token Security

MoQ Access Tokens minted by the Application Provider:

- MUST be signed using ES256.
- MUST include an expiration time.
  Tokens without expiration
  cannot be effectively revoked
  in a distributed relay network.
- MUST be scoped to a single scope_id.
  Tokens that span multiple scopes
  violate scope isolation.
- SHOULD include the minimum set of namespace prefixes
  and operations required by the client.
  Over-privileged tokens increase blast radius
  if compromised.
- MUST use proof-of-possession mechanisms
  ({{c4m}}).
  C4M binds credentials
  to the client's cryptographic key,
  ensuring a captured token is unusable
  without the corresponding private key.

## Signing Key Security

Application Providers MUST protect
their signing private keys.
A compromised signing key
allows an attacker to mint valid MoQ Access Tokens
for the associated scope.

Application Providers SHOULD:

- Use hardware security modules (HSMs)
  or equivalent key protection
- Provision multiple signing keys
  to limit the blast radius of a compromise
- Rotate signing keys periodically
- Remove compromised keys immediately
  via the provisioning API

## Proof-of-Possession Security

The security properties of C4M and DPoP
are defined in {{CAT4MOQ}} and {{MOQDPOP}} respectively.
Relays MUST follow the validation requirements
specified in those documents.

## Scope Isolation

Relays MUST enforce strict isolation between MoQ Scopes.
A connection authenticated to scope A
MUST NOT observe namespaces, tracks, or objects
belonging to scope B,
even if both scopes are served
by the same relay process.

# IANA Considerations

This document has no IANA actions.

--- back

# Acknowledgments
{:numbered="false"}

Thanks to Lucas Pardue and Jacob Curtis
for design input.
