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
  RFC6749:
  RFC7519:
  RFC8705:
  RFC9449:
  CAT4MOQ: I-D.ietf-moq-c4m

informative:
  RFC9110:
  RFC9000:
  RFC9576:
  RFC9577:

--- abstract

This document describes concepts
related to provisioning MoQ relay scopes on CDN infrastructure,
including scope creation, credential-to-scope mapping,
origin fallback configuration,
relay topology,
namespace authorization,
and client discovery.
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
(scopes, credential-to-scope mapping, origin fallback,
relay topology, authorization policy)
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

Edge Relay:

: A relay that accepts connections from end-user clients
  (publishers and subscribers).
  It is the first relay a client connects to.

Aggregation Relay:

: An intermediate relay
  that aggregates traffic between edge relays
  and origin relays.
  It does not accept direct client connections.

Origin:

: The authoritative source for content within a scope.
  An origin may be a relay operated by the CDN provider,
  or a customer-operated MoQT endpoint.

# Scope Provisioning

A CDN provider exposes an HTTP API endpoint
for creating scopes:

~~~
POST /moq/scopes
~~~

The request MAY include configuration
(see {{origin-fallback}}, {{auth-policy}}, {{relay-topology}}).

The response includes
a server-generated scope identifier,
connection discovery information,
and a mechanism for obtaining credentials:

~~~json
{
  "scope_id": "a1b2c3d4e5f6",
  "discovery": {
    "url": "moqt://relay.example.com",
    "regional_endpoints": [
      {"region": "us-west", "url": "moqt://uswest.relay.example.com"},
      {"region": "eu-west", "url": "moqt://euwest.relay.example.com"}
    ]
  },
  "token_endpoint": "https://auth.cdn.example.com/moq/token",
  "supported_credential_types": ["cat", "cat+dpop"],
  "default_credential_type": "cat",
  "token": "eyJhbGciOiJFZDI1NTE5..."
}
~~~

The `discovery` object provides one or more URLs
through which clients can reach relay infrastructure
serving this scope. See {{client-discovery}}.

The `token_endpoint` is an OAuth 2.0 token endpoint
that can issue additional scoped credentials.
See {{credential-issuance}}.

The `supported_credential_types` array lists
the credential mechanisms the relay accepts.
See {{pop-credentials}} for proof-of-possession options.

The `default_credential_type` indicates the preferred
credential type for new clients.

The `token` is an initial credential
returned for bootstrapping convenience.
It MUST be short-lived.
Clients use the token endpoint
to obtain properly-bound CAT credentials
for ongoing access.

## Scope Lifecycle

The API supports the full lifecycle of a scope:

~~~
POST   /moq/scopes              - Create
GET    /moq/scopes/{scope_id}   - Read configuration and status
PATCH  /moq/scopes/{scope_id}   - Update configuration
DELETE /moq/scopes/{scope_id}   - Delete (initiates drain)
~~~

When a scope is deleted,
the relay SHOULD send GOAWAY {{MOQT}}
to all connected clients
before terminating connections.
The GOAWAY MAY include a redirect URI
if the scope is being migrated.

## Scope Status

A GET request to a scope returns
both its configuration and runtime status:

~~~json
{
  "scope_id": "a1b2c3d4e5f6",
  "status": {
    "state": "active",
    "active_publishers": 3,
    "active_subscribers": 1200,
    "relay_count": 4
  },
  "config": { ... }
}
~~~

# Connecting to a Scope {#connecting}

Clients connect to a provisioned scope
by combining a discovery URL
with a credential.

## Credential Presentation

The MoQT Authorization Token parameter {{MOQT}}
carries the client's credential
during session establishment.
The supported token types are:

- CAT (CWT Authentication Token) — see {{cat-moq}}
- CAT with DPoP — see {{cat-dpop}}

These mechanisms carry both the credential
and cryptographic proof-of-possession
within the MoQT control plane,
binding authorization to the session
without relying on the transport URL.

## Scope Mapping

The relay must determine which scope
a connection belongs to.
The credential itself carries the `scope_id` claim,
so the relay extracts the scope binding
from the validated token
rather than from the URL path.

This avoids placing credentials
or scope identifiers in the URL,
which would expose them to logging,
browser history, and intermediary caches.

Alternative scope-mapping approaches:

- **Claim-based** (RECOMMENDED):
  The token's `scope_id` claim
  identifies the target scope.
  The relay validates the token first,
  then maps to the scope.
  This works identically over WebTransport
  and raw QUIC, since the claim is carried
  inside the MoQT Authorization Token parameter
  after the QUIC handshake completes.
  No scope information leaks to the network.

- **SNI-based**:
  Each scope is assigned a unique hostname
  (e.g., `a1b2c3d4e5f6.relay.example.com`).
  The relay extracts the scope from the TLS SNI field
  during QUIC connection establishment.
  This applies to both WebTransport and raw QUIC,
  since SNI is present in the TLS ClientHello
  within the QUIC handshake.
  This leaks scope identifiers to network observers
  (SNI is sent in cleartext unless ECH is used)
  but keeps credentials out of the transport layer.

- **URL path / connection token** (NOT RECOMMENDED):
  For WebTransport, the scope identifier
  or a short-lived opaque token
  is placed in the URL path.
  For raw QUIC, the equivalent is a
  QUIC initial token or a well-known
  ALPN extension that encodes the scope.
  This is simple but exposes information
  to intermediary logs, browser history
  (WebTransport), and on-path observers
  (QUIC initial packets are not encrypted).
  If used, the path component or initial token
  MUST be treated as sensitive
  and SHOULD be short-lived.

## Connection Validation

The relay validates the credential
presented in the Authorization Token parameter:

1. Verifies the token signature (issuer's key)
2. Verifies proof-of-possession (client's key)
3. Checks expiration and replay constraints
4. Extracts the `scope_id` and maps the connection

If validation fails,
the relay MUST reject the connection
with the appropriate MoQT error code.

Within a scope,
all MoQT operations (SUBSCRIBE, PUBLISH, etc.)
are isolated.
Publishers and subscribers in one scope
cannot see namespaces or tracks from another scope.

# Credential Model {#credential-model}

## Credential Issuance {#credential-issuance}

The provisioning API returns a `token_endpoint`
conforming to OAuth 2.0 {{RFC6749}}.
Credential consumers (publishers, subscribers, relays)
obtain tokens by authenticating to this endpoint.

The token endpoint issues JWTs {{RFC7519}} containing:

- `scope_id`: The MoQT scope this token grants access to
- `namespaces`: Authorized namespace prefixes (see {{auth-policy}})
- `role`: One of `publish`, `subscribe`, or `pubsub`
- `exp`: Token expiration time

~~~json
{
  "alg": "EdDSA",
  "typ": "JWT"
}
.
{
  "sub": "publisher-001",
  "scope_id": "a1b2c3d4e5f6",
  "namespaces": [
    {"prefix": ["live", "meeting-42"], "ops": ["publish"]}
  ],
  "role": "publish",
  "exp": 1717372800
}
~~~

Relays MUST validate the token signature,
expiration, and scope_id
before mapping the connection.

## Proof-of-Possession Credentials {#pop-credentials}

Proof-of-possession (PoP) mechanisms
bind a credential to a cryptographic key
held by the client,
so that a captured token is useless
without the corresponding private key.

The MoQT Authorization Token parameter {{MOQT}}
supports the following PoP credential types,
all of which operate at the MoQT control plane layer
and are carried within MoQT session messages.

### CAT for MoQT {#cat-moq}

The CWT Authentication Token (CAT) for MoQT {{CAT4MOQ}}
defines a mechanism for conveying
proof-of-possession credentials
within MoQT control messages.

A CAT credential is a CWT (CBOR Web Token)
that contains a `cnf` (confirmation) claim
binding the token to the client's public key.
The client proves possession
by signing a challenge derived from
the MoQT session context.

The provisioning API indicates CAT support
in the credential issuance response:

~~~json
{
  "scope_id": "a1b2c3d4e5f6",
  "credential_type": "cat",
  "token_endpoint": "https://auth.cdn.example.com/moq/token",
  "cat": {
    "cnf_alg": "EdDSA",
    "challenge_method": "session_binding"
  }
}
~~~

When using CAT, the client:

1. Generates an asymmetric key pair
2. Obtains a CWT from the token endpoint
   that binds the token to the client's public key
   via the `cnf` claim
3. Presents the CWT in the MoQT CLIENT_SETUP
   or subsequent control messages
4. Signs the session-bound challenge
   to prove possession of the private key

Relays MUST validate both the CWT signature
(from the issuer) and the client's proof
(from the bound key) before granting access.

### CAT with DPoP {#cat-dpop}

CAT credentials MAY be combined with
Demonstrating Proof-of-Possession (DPoP) {{RFC9449}}
for additional replay protection.

DPoP adds a per-request proof
that binds the credential
to a specific relay endpoint and timestamp,
preventing token replay across relays
or outside a narrow time window.

~~~json
{
  "scope_id": "a1b2c3d4e5f6",
  "credential_type": "cat+dpop",
  "token_endpoint": "https://auth.cdn.example.com/moq/token",
  "cat": {
    "cnf_alg": "EdDSA",
    "challenge_method": "session_binding"
  },
  "dpop": {
    "alg": "EdDSA",
    "nonce_required": true
  }
}
~~~

When using CAT+DPoP, the client:

1. Generates a DPoP key pair
   (MAY reuse the CAT confirmation key)
2. Obtains a CWT bound to the DPoP public key
3. For each MoQT control message
   requiring authorization,
   generates a DPoP proof JWT containing:
   - `htm`: The MoQT message type
     (e.g., "SUBSCRIBE", "PUBLISH")
   - `htu`: The relay endpoint URI
   - `iat`: Current timestamp
   - `nonce`: Server-provided nonce (if required)
4. Presents both the CWT and DPoP proof
   in the MoQT control message

Relays MUST reject DPoP proofs
with timestamps outside the acceptable window
or with previously-seen `jti` values.

### Credential Type Selection

The provisioning API advertises
supported credential types
in the scope configuration:

~~~json
{
  "scope_id": "a1b2c3d4e5f6",
  "supported_credential_types": [
    "cat",
    "cat+dpop"
  ],
  "default_credential_type": "cat"
}
~~~

Clients SHOULD use the strongest credential type
supported by both client and relay.
The preference order from strongest to weakest is:

1. `cat+dpop` — Proof-of-possession with replay protection
2. `cat` — Proof-of-possession

Relays MAY reject connections
using credential types weaker than
the scope's configured minimum:

~~~json
{
  "config": {
    "minimum_credential_type": "cat"
  }
}
~~~

## Credential Rotation

Tokens SHOULD have a lifetime
no longer than the expected session duration.
When a token is near expiration,
clients SHOULD obtain a new token
from the token endpoint
and present it using
the MoQT Authorization Token update mechanism.

The provisioning API supports credential revocation:

~~~
DELETE /moq/scopes/{scope_id}/tokens/{token_id}
~~~

Relays that receive a revoked token
MUST reject the connection
with the UNAUTHORIZED termination code.

## Relay-to-Relay Credentials

Relay-to-relay peering connections
use a separate credential class
from client-to-relay connections.
Relay credentials:

- MUST use mutual TLS {{RFC8705}} or
  long-lived signed tokens
  with a distinct issuer from client tokens
- SHOULD have longer lifetimes
  than client tokens
  (hours to days rather than minutes to hours)
- MUST be scoped to the relay's role
  (edge, aggregation, or origin)

~~~json
{
  "relay_credentials": {
    "method": "mutual_tls",
    "ca_certificate": "-----BEGIN CERTIFICATE-----...",
    "allowed_identities": ["*.relay.cdn.example.com"]
  }
}
~~~

When mutual TLS is used,
the relay presents a client certificate
during QUIC connection establishment.
The peer relay validates the certificate
against the configured CA
before accepting the peering session.

# Authorization Policy {#auth-policy}

A scope MAY be configured with
namespace-level authorization rules
that restrict which operations
are permitted for a given credential.

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
        },
        {
          "namespace_prefix": ["live", "meeting-42", "chat"],
          "operations": ["subscribe"]
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
The relay MUST check authorization
before forwarding PUBLISH messages
or establishing subscriptions
on behalf of a client.

Authorization rules are evaluated
using longest-prefix match:
the most specific matching rule applies.

## Namespace Isolation

Within a scope,
the authorization policy controls
which namespaces are visible to which participants.
A subscriber whose credential
only authorizes `["live", "meeting-42", "chat"]`
MUST NOT receive PUBLISH_NAMESPACE notifications
for other prefixes within the scope.

This allows a single scope
to host multiple logical channels
(e.g., audio, video, screen share, chat)
with differentiated access.

# Client Discovery {#client-discovery}

The provisioning response includes
a `discovery` object
that describes how clients locate relay infrastructure.
Multiple discovery methods
can be provided simultaneously.

## Anycast

A single global URL
routed via anycast to the nearest edge relay:

~~~json
{
  "type": "anycast",
  "url": "moqt://edge.cdn.example.com"
}
~~~

The CDN provider operates network load balancers
behind the anycast address
to ensure connection affinity.

## Regional Endpoints

Explicit per-region URLs
for clients that can determine their own region
or want deterministic relay selection:

~~~json
{
  "type": "regional",
  "endpoints": [
    {"region": "us-west-2", "url": "moqt://usw2.relay.example.com"},
    {"region": "eu-west-1", "url": "moqt://euw1.relay.example.com"}
  ]
}
~~~

## Orchestration Service

A URL pointing to a client steering service
that selects the optimal relay
based on client location, network conditions,
current load, and application context:

~~~json
{
  "type": "orchestration",
  "url": "https://steering.cdn.example.com/v1/select"
}
~~~

The orchestration service is out of scope
for this document
but typically accepts client hints
(IP geolocation, RTT measurements)
and returns a relay URL.

## Local Discovery

For deployments with local relay infrastructure
(enterprise, event venues, edge compute),
mDNS service discovery MAY be used.
The well-known service type is `_moqt._udp`.

# Relay Topology {#relay-topology}

A scope's relay infrastructure
consists of one or more relays
organized in a directed graph.
The provisioning API configures
this topology.

## Relay Roles

Each relay in the topology has a role:

Edge:
: Accepts client connections.
  Forwards subscriptions upstream.
  Caches and serves content downstream.

Aggregation:
: Aggregates traffic between edges and origins.
  Does not accept direct client connections.
  Provides fan-out efficiency
  and geographic distribution.

Origin:
: Authoritative source for content.
  May be a CDN-operated relay
  or customer infrastructure.

## Topology Configuration

The topology is configured
as part of the scope:

~~~json
{
  "config": {
    "topology": {
      "relays": [
        {
          "relay_id": "edge-usw-01",
          "role": "edge",
          "region": "us-west-2",
          "upstream": ["agg-usw-01"]
        },
        {
          "relay_id": "edge-euw-01",
          "role": "edge",
          "region": "eu-west-1",
          "upstream": ["agg-euw-01"]
        },
        {
          "relay_id": "agg-usw-01",
          "role": "aggregation",
          "region": "us-west-2",
          "upstream": ["origin-01"]
        },
        {
          "relay_id": "agg-euw-01",
          "role": "aggregation",
          "region": "eu-west-1",
          "upstream": ["origin-01"]
        },
        {
          "relay_id": "origin-01",
          "role": "origin",
          "region": "us-east-1",
          "upstream": []
        }
      ]
    }
  }
}
~~~

## Peering Between Relays

A relay establishes peering with its configured upstreams
by opening a MoQT session
and authenticating with relay credentials
(see {{credential-model}}).

Once peered, the downstream relay:

- Sends SUBSCRIBE_NAMESPACE upstream
  for namespaces its clients are interested in
- Receives PUBLISH_NAMESPACE from upstream
  for namespaces available in the scope
- Forwards SUBSCRIBE upstream
  when a local cache miss occurs
- Caches content received from upstream
  for serving to other downstream subscribers

## Path Selection

When a relay has multiple upstream peers configured,
it selects among them based on:

- **Priority**: Lower priority value is preferred.
  Used for primary/backup configurations.
- **Weight**: For equal-priority upstreams,
  distributes load proportionally.
- **Health**: Unhealthy upstreams are excluded.
  See {{health-monitoring}}.
- **Latency**: When configured,
  the relay MAY prefer the upstream
  with lowest measured round-trip time.

~~~json
{
  "upstream": [
    {
      "relay_id": "agg-usw-01",
      "priority": 1,
      "weight": 80
    },
    {
      "relay_id": "agg-usw-02",
      "priority": 1,
      "weight": 20
    },
    {
      "relay_id": "agg-euw-01",
      "priority": 2,
      "weight": 100
    }
  ]
}
~~~

# Origin Fallback {#origin-fallback}

A scope MAY be configured
with one or more upstream origin URLs.
When a subscriber requests content
that isn't available on the relay
or its immediate upstream peers,
the relay connects to the origin to fetch it.

This is configured at provisioning time:

~~~json
{
  "config": {
    "origin_fallback": {
      "origins": [
        {
          "url": "moqt://origin.example.com",
          "priority": 1,
          "namespaces": [{"prefix": ["live"]}],
          "credential": "relay-credential-ref"
        },
        {
          "url": "moqt://backup-origin.example.com",
          "priority": 2,
          "namespaces": [{"prefix": ["live"]}],
          "credential": "relay-credential-ref"
        }
      ],
      "connect_on": "cache_miss",
      "health_check": {
        "interval_seconds": 5,
        "timeout_seconds": 2
      }
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

## Origin Selection

When multiple origins are configured,
the relay selects based on:

- **Namespace matching**: The origin whose `namespaces` field
  has the longest prefix match
  for the requested track.
- **Priority**: Among matching origins,
  lower priority value is preferred.
- **Health**: Unhealthy origins are skipped.

## Connection Policy

The `connect_on` field controls
when the relay establishes the upstream connection:

- `cache_miss`: Connect only when a subscriber
  requests content not in the local cache (default).
- `eager`: Connect at scope activation time.
  Reduces first-subscriber latency
  at the cost of an idle connection.
- `subscribe_namespace`: Connect when the relay
  receives a SUBSCRIBE_NAMESPACE from a downstream client,
  before any individual track subscription.

# Namespace Propagation {#namespace-propagation}

The provisioning API controls
how namespace information flows
through the relay topology.

## Publish Advertisements

When an origin or publisher announces a namespace
via PUBLISH_NAMESPACE,
the relay propagates this advertisement downstream
to all peers that have expressed interest
via SUBSCRIBE_NAMESPACE.

The provisioning API MAY configure
which namespaces a relay advertises downstream,
independent of what it has received from upstream:

~~~json
{
  "config": {
    "namespace_propagation": {
      "advertise_downstream": [
        {"prefix": ["live"], "propagate": true},
        {"prefix": ["internal"], "propagate": false}
      ],
      "subscribe_upstream": [
        {"prefix": ["live"], "on": "client_interest"},
        {"prefix": ["catalog"], "on": "always"}
      ]
    }
  }
}
~~~

## Subscribe Propagation

When a relay receives SUBSCRIBE_NAMESPACE
from a downstream client or peer,
it decides whether to propagate upstream:

- `client_interest`: Propagate upstream
  only when a client expresses interest (default).
- `always`: Propagate immediately at peering time,
  regardless of downstream interest.
- `never`: Do not propagate.
  Used for locally-scoped namespaces.

# Health Monitoring {#health-monitoring}

Relays in the topology report health status.
The provisioning API configures health monitoring
and defines thresholds for steering decisions.

~~~json
{
  "config": {
    "health": {
      "reporting_endpoint": "https://health.cdn.example.com/v1/relay",
      "interval_seconds": 10,
      "drain_threshold": {
        "subscriber_count": 10000,
        "bandwidth_mbps": 8000
      },
      "goaway_on_drain": true
    }
  }
}
~~~

When a relay exceeds its drain threshold,
it signals overload to the orchestration service.
If `goaway_on_drain` is true,
the relay sends GOAWAY to new connections
with a redirect to a less-loaded relay.

## Graceful Relay Maintenance

When a relay needs maintenance,
the operator initiates a drain:

~~~
POST /moq/scopes/{scope_id}/relays/{relay_id}/drain
~~~

The relay:

1. Stops accepting new client connections
2. Sends GOAWAY to all connected clients
   with a redirect URI for the replacement relay
3. Waits for existing subscriptions to migrate
4. Shuts down after a configurable timeout

# Cache Policy {#cache-policy}

Relays cache MoQT objects for distribution efficiency.
The provisioning API configures cache behavior per scope.

~~~json
{
  "config": {
    "cache": {
      "enabled": true,
      "max_duration_seconds": 300,
      "max_objects_per_track": 1000,
      "eviction": "lru"
    }
  }
}
~~~

A relay with caching enabled
serves FETCH requests from cache
without forwarding to upstream,
when the requested objects are available locally.

# Multi-CDN Provisioning {#multi-cdn}

A MoQT scope MAY span multiple CDN providers.
{{MOQT}} defines a scope as
"a set of servers (as identified by their connection URIs)
for which a Full Track Name is guaranteed to be unique."
When an application requires geographic reach
or redundancy beyond a single provider,
the scope must be provisioned consistently
across all participating CDNs.

## Scope Federation

Multiple CDN providers serving the same scope
form a federation.
Each provider provisions the scope independently
using its own provisioning API,
but all providers agree on:

- A common scope federation identifier
- Namespace ownership boundaries
- A shared or federated credential issuer

~~~json
{
  "config": {
    "federation": {
      "federation_id": "urn:moq:scope:live-event-2026",
      "role": "secondary",
      "primary_provider": {
        "provisioning_api": "https://api.cdn-primary.example.com/moq",
        "peering_url": "moqt://peering.cdn-primary.example.com"
      },
      "auth_issuers": [
        "https://auth.cdn-primary.example.com",
        "https://auth.cdn-secondary.example.com"
      ],
      "namespace_ownership": {
        "local": [{"prefix": ["live", "eu"]}],
        "remote": [{"prefix": ["live", "us"]}]
      }
    }
  }
}
~~~

## Inter-CDN Peering

CDN providers establish peering connections
at the federation boundary.
A peering relay at CDN A
connects to a peering relay at CDN B
and exchanges namespace advertisements.

Each provider's relay infrastructure
treats the peer CDN as an upstream origin
for namespaces owned by that peer,
and as a downstream subscriber
for namespaces owned locally.

~~~
CDN A (owns ["live", "us"])         CDN B (owns ["live", "eu"])
+----------------------+           +----------------------+
|  edge -> agg -> peer |<--------->| peer <- agg <- edge  |
+----------------------+   MoQT   +----------------------+
                          peering
~~~

The peering connection uses relay-class credentials
(mutual TLS or long-lived signed tokens).
Each side advertises only the namespaces
it is authoritative for.

## Credential Federation

For clients to move between CDN providers
(e.g., during failover or geographic rebalancing),
tokens issued by one provider
must be accepted by others in the federation.

Approaches:

- **Shared issuer**: All providers trust
  a common token-signing authority
  operated by the customer or a neutral party.
- **Cross-signed tokens**: Each provider's token endpoint
  is configured as a trusted issuer
  at all other providers.
- **Token exchange**: A client presents a token
  from provider A to provider B's token endpoint
  and receives a locally-valid token.

The federation configuration
lists all trusted `auth_issuers`.
A relay MUST accept tokens
signed by any configured issuer,
provided the token's `scope_id` matches
or the `federation_id` matches.

## Namespace Ownership

Each provider in the federation
declares which namespace prefixes it owns.
Ownership means:

- The provider's origin is authoritative
  for content in those namespaces
- The provider's relays originate
  PUBLISH_NAMESPACE for those prefixes
- Other providers treat those namespaces
  as remote and fetch via peering

A namespace prefix MUST NOT be owned
by more than one provider simultaneously.
Overlapping ownership causes content conflicts
and violates the MoQT requirement
that a Full Track Name uniquely identifies content
within a scope.

## Failover Between Providers

When a provider becomes unavailable,
its namespaces can be failed over
to another provider in the federation.
This requires:

1. The backup provider has access
   to the content source
   (e.g., direct origin connection)
2. The backup provider takes ownership
   of the affected namespace prefixes
3. Client steering directs traffic
   to the backup provider's infrastructure
4. After recovery, ownership reverts
   and clients are steered back

The provisioning API supports ownership transfer:

~~~
PATCH /moq/scopes/{scope_id}
{
  "config": {
    "federation": {
      "namespace_ownership": {
        "local": [
          {"prefix": ["live", "eu"]},
          {"prefix": ["live", "us"]}
        ]
      }
    }
  }
}
~~~

# Security Considerations

## Provisioning API Security

The provisioning API endpoint
MUST require authentication and authorization.
It SHOULD use OAuth 2.0 bearer tokens
or mutual TLS for API authentication.
The API MUST be served over HTTPS.

## Token Security

Credentials issued by the provisioning API:

- MUST be signed using an asymmetric algorithm
  (EdDSA or ECDSA).
  Symmetric algorithms (HMAC)
  allow any relay to forge tokens
  and MUST NOT be used.
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
  ({{pop-credentials}}).
  CAT binds credentials
  to the client's cryptographic key,
  ensuring a captured token is unusable
  without the corresponding private key.

## Proof-of-Possession Security

When using CAT ({{cat-moq}}),
the session-binding challenge
MUST incorporate entropy from both
the client and the relay
to prevent pre-computation attacks.

When using DPoP ({{cat-dpop}}),
relays MUST enforce:

- Nonce freshness to prevent replay
- Timestamp bounds (RECOMMENDED: 60 seconds)
- `jti` uniqueness within the replay window

## Scope Mapping Security

Claim-based scope mapping ({{connecting}})
is preferred because it avoids
exposing scope identifiers or credentials
in the transport URL.

SNI-based mapping leaks scope identifiers
to passive network observers
but is acceptable when scope names
are not considered sensitive.

URL-path-based mapping is NOT RECOMMENDED
because:

- URLs are logged by intermediaries,
  load balancers, and CDN access logs
- URLs appear in browser history
  (for WebTransport connections)
- URLs may be shared inadvertently
  (copy-paste, referrer headers)
- Credentials in URLs cannot be rotated
  without reconnection

If URL-path mapping is used despite these risks,
implementations MUST treat the path as sensitive,
the path token MUST be short-lived,
and the token endpoint SHOULD be used
for credential renewal via the MoQT
Authorization Token update mechanism.

## Relay-to-Relay Security

Peering connections between relays
MUST use TLS 1.3 or later.
Mutual authentication is REQUIRED:
each relay MUST verify the identity
of its peer before exchanging
namespace advertisements or forwarding content.

A compromised relay can:

- Observe all content flowing through it
  (mitigated by end-to-end encryption of payloads)
- Inject content into namespaces
  it is authorized for
- Deny service by dropping subscriptions

Operators SHOULD deploy end-to-end encryption
of media payloads (opaque to relays)
and monitor relay behavior
for anomalous subscription or publication patterns.

## Scope Isolation

Relays MUST enforce strict isolation between scopes.
A connection authenticated to scope A
MUST NOT observe namespaces, tracks, or objects
belonging to scope B,
even if both scopes are served
by the same relay process.

## Authorization Enforcement

Relays MUST validate authorization
before performing any operation
on behalf of a client:

- Before forwarding a PUBLISH:
  verify the publisher's token
  authorizes publication
  to the target namespace.
- Before establishing a subscription:
  verify the subscriber's token
  authorizes subscription
  to the target namespace.
- Before relaying PUBLISH_NAMESPACE:
  verify the advertiser is authoritative
  or has received authorization
  from an authoritative upstream.

Failure to enforce authorization
allows namespace squatting,
content injection,
and information disclosure.

# IANA Considerations

This document has no IANA actions.

--- back

# Acknowledgments
{:numbered="false"}

Thanks to Lucas Pardue and Jacob Curtis
for design input.
