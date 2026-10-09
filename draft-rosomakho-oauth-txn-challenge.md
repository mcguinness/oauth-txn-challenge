---
title: "OAuth Transaction Authorization Challenge"
abbrev: "Txn Challenge"
category: std

docname: draft-rosomakho-oauth-txn-challenge-latest
submissiontype: IETF  # also: "independent", "editorial", "IAB", or "IRTF"
number:
date:
consensus: true
v: 3
area: "Security"
workgroup: "Web Authorization Protocol"
keyword:
 - Transaction
 - Authorization
 - Human-in-the-loop
venue:
  group: "Web Authorization Protocol"
  type: "Working Group"
  mail: "oauth@ietf.org"
  arch: "https://mailarchive.ietf.org/arch/browse/oauth/"
  github: "yaroslavros/oauth-txn-challenge"
  latest: "https://yaroslavros.github.io/oauth-txn-challenge/draft-rosomakho-oauth-txn-challenge.html"

author:
 -
    fullname: Yaroslav Rosomakho
    organization: Zscaler
    email: yrosomakho@zscaler.com
 -
    fullname: "Brian Campbell"
    organization: "Ping Identity"
    email: bcampbell@pingidentity.com
 -
    fullname: Karl McGuinness
    organization: Independent
    email: public@karlmcguinness.com
 -
    fullname: Pieter Kasselman
    organization: Defakto Security
    email: pieter@defakto.security

normative:

informative:
  IANA.HTTP.FieldNames:
    title: Hypertext Transfer Protocol (HTTP) Field Name Registry
    target: https://www.iana.org/assignments/http-fields/
    author:
      -name: IANA
  IANA.OAuth.Parameters:
    title: OAuth Parameters
    target: https://www.iana.org/assignments/oauth-parameters/
    author:
      -name: IANA
  IANA.JSON.Web.Token:
    title: JSON Web Token (JWT)
    target: https://www.iana.org/assignments/jwt/
    author:
      -name: IANA

--- abstract

This document defines an OAuth mechanism for transaction-specific authorization challenges.
A protected resource can require additional authorization for a particular operation by
returning a transaction authorization challenge. This is useful when requests are mediated by
agents, automated workflows, or delegated services and the protected resource requires
confirmation from a human user, resource owner, or organizational authority. The client
presents the challenge to an authorization server, which validates the challenge, obtains
any required approval, and issues an OAuth 2.0 access token whose granted authorization details, expressed
using Rich Authorization Requests, describe the approved operation. The access token is then
presented to the protected resource as evidence that the challenged operation was authorized.

--- middle

# Introduction

OAuth 2.0 ({{!OAUTH-FRAMEWORK=RFC6749}}) access tokens authorize access to protected
resources. In many deployments, however, a protected resource cannot determine whether a
requested operation is acceptable based only on the access token that accompanies the request.
The access token might establish that the caller is authorized to interact with the protected
resource, but a particular operation can still require additional, transaction-specific
authorization.

This situation arises when the protected resource needs confirmation that a concrete transaction
has been approved by an appropriate party. The approving party might be the human user on whose
behalf the request is made, a different resource owner, or an organizational authority such as
an administrator, manager, data owner, or policy decision point.

This document defines a transaction authorization challenge. A protected resource uses this
challenge to request additional authorization for a specific operation. The challenge is
relayed to a client, which presents it to an authorization server. The authorization server
validates the challenge, obtains any required approval, and issues an OAuth 2.0 access token
whose granted authorization details, expressed using Rich Authorization Requests
({{!OAUTH-RAR=RFC9396}}), describe the approved operation and bind the access token to the
challenged transaction. The access token is then presented to the protected resource as
evidence that the challenged operation was authorized. Reusing the OAuth 2.0 access token
abstraction avoids defining a new bearer artifact and allows existing access-token validation
and protection mechanisms, including sender-constrained access tokens, to apply unchanged.

This mechanism is complementary to OAuth step-up authentication defined in {{?OAUTH-STEP-UP=RFC9470}}.
Step-up authentication enables a protected resource to require stronger or fresher authentication
of a user. A transaction authorization challenge instead requests authorization for a specific
transaction. The approving party can be different from the subject associated with the access token
used for the original request.

## Human Approval for Agent-Mediated Actions

Software agents, automated workflows, and delegated services can perform operations on behalf of
users. Some operations are sufficiently sensitive that a protected resource might require explicit
approval before processing them. Examples include sending a payment, publishing content, deleting
data, modifying access policy, or disclosing sensitive information.

Local confirmation mechanisms within an agent framework can reduce risk, but they are not always
visible to the protected resource and do not necessarily produce authorization evidence that the
protected resource can validate. A transaction authorization challenge allows the protected resource
to require explicit authorization for the concrete operation and to receive an access token
representing that authorization.

## Authorization by a Different Resource Owner

The party that initiated an operation is not always the party authorized to approve it. For example,
an agent acting on behalf of one user might request access to a resource owned or controlled by
another user. The protected resource can determine that approval from the resource owner, or
from another party authorized to act for that resource owner, is required before the operation
can proceed.

In this case, the transaction authorization challenge allows the protected resource to describe the
requested operation and the authorization server to determine the appropriate approving party. The
resulting access token represents authorization of the challenged operation by the party
selected according to authorization server policy.

## Organizational Approval

Some operations require approval by an organizational authority rather than by an individual end
user. Examples include approving access to regulated data, authorizing a production deployment,
granting elevated administrative access, or permitting data transfer to an external party.

A transaction authorization challenge allows the protected resource to request authorization evidence from
an authorization server or associated policy infrastructure. The authorization server validates the challenge,
applies organizational policy, obtains any required approval, and issues an access token only when the
required authorization has been obtained.

# Conventions and Definitions

{::boilerplate bcp14-tagged}

This document uses the terms "client", "authorization server", "access token", "refresh token", and "protected
resource" as defined by {{OAUTH-FRAMEWORK}}.

This document uses the term "authorization details" as defined by {{OAUTH-RAR}}.

This document defines the following additional terms:

Approving Party:
: The human user, resource owner, organizational authority, or policy authority whose approval is required
  before the challenged transaction can proceed.

Agent:
: A software component, automated workflow, delegated service, or other intermediary that attempts an operation
  at a protected resource and relays a transaction authorization challenge to a client.

Transaction Authorization Challenge:
: A challenge returned by a protected resource to indicate that a specific requested operation requires
  transaction-specific authorization before it can proceed.

# Architecture

A transaction authorization challenge involves a protected resource, an agent, a client, an authorization
server, and an approving party.

The protected resource receives a request and determines whether the authorization currently presented
with the request is sufficient. If the protected resource determines that the requested operation requires
transaction-specific authorization, it returns a transaction authorization challenge.

The agent is the component that attempted the operation at the protected resource. The agent relays
the transaction authorization challenge to the client and later presents the resulting access token
to the protected resource. The agent is not trusted to modify the contents of the challenge or the resulting
access token.

The client receives the transaction authorization challenge from the agent and presents it to the authorization
server. The client is responsible for mediating the interaction between the agent-facing environment and the
authorization server. The client does not need to interpret all application-specific details of the challenged
transaction, but it MUST preserve the integrity of the challenge when presenting it to the authorization server.

The authorization server validates the transaction authorization challenge, determines the approving party,
obtains any required approval, and issues an access token whose granted authorization details describe the
approved operation. The authorization server can use local policy, resource metadata, resource-owner
information, organizational policy, or other authorization context to determine whether the requested operation
can be approved.

The approving party is the human user, resource owner, organizational authority, or policy authority whose
approval is required. The approving party can be the same subject on whose behalf the agent is acting, but
this is not required.

The following figure shows the basic protocol flow:

~~~aasvg
+----------+        +-------+        +--------------------+
|  Client  |<------>| Agent |<------>| Protected Resource |
+----------+        +-------+        +--------------------+
     |
     | transaction authorization challenge
     v
+----------------------+
| Authorization Server |
+----------------------+
     |
     | approval interaction
     v
+-----------------+
| Approving Party |
+-----------------+
~~~
{: #fig-architecture title="Overall architecture of Transaction Authorization Challenge"}

The flow is as follows:

1. The agent sends a request to the protected resource using its existing authorization context.

1. The protected resource determines that the request requires transaction-specific authorization.

1. The protected resource returns a transaction authorization challenge to the agent.

1. The agent relays the transaction authorization challenge to the client.

1. The client presents the transaction authorization challenge to the authorization server.

1. The authorization server validates the challenge, determines the approving party, and obtains any required approval.

1. The authorization server issues an access token to the client. The access token's granted authorization
   details describe the approved operation, and the access token is bound to the challenged transaction.

1. The client provides the access token to the agent.

1. The agent retries or continues the request and presents the access token to the protected resource.

1. The protected resource validates the access token, including its authorization details, and processes the
   request if the token authorizes the challenged operation.

# Transaction Authorization Challenge

A transaction authorization challenge is a signed JWT ({{!JWT=RFC7519}}) generated by a protected resource to
request transaction-specific authorization for a particular operation. The challenge describes the operation to be
authorized, identifies the authorization server expected to process the challenge, identifies the intended
audience of the resulting access token, and contains freshness and integrity protection.

The challenge is consumed by the authorization server and can also be validated by the client before the client
presents it to the authorization server. The agent relays the challenge, but it is not trusted to modify the challenge
or to describe the challenged transaction.

## Challenge Capability Signal

An agent indicates that it supports the transaction authorization challenge mechanism by sending the
`Accept-Txn-Challenge` header field with a true value in requests to a protected resource.

The `Accept-Txn-Challenge` header field is an Item Structured Field; see {{Section 3.3 of !STRUCTURED-FIELDS=RFC9651}}.
Its value MUST be a Boolean. Any other value type MUST be handled by recipients as if the field were not present.
For example, if this field is included multiple times, its type will become a List and the field will be ignored.

This document does not define any parameters for the `Accept-Txn-Challenge` header field value, but future
documents might define parameters. Receivers MUST ignore unknown parameters.

The following example indicates support for transaction authorization challenges:

~~~
Accept-Txn-Challenge: ?1
~~~
{: #fig-challenge-capability title="Agent indicating support for transaction authorization"}

An `Accept-Txn-Challenge` header field with a false value has the same semantics as when the header field is not present.

An agent MUST NOT include the `Accept-Txn-Challenge` header field unless it has a client interaction path capable
of relaying the challenge to a client that can validate the challenge and present it to an authorization
server.

A protected resource MUST NOT return a transaction authorization challenge unless the request includes an
`Accept-Txn-Challenge` header field with a true value, or the protected resource has out-of-band knowledge that
the client supports this specification.

## Challenge Response

When a protected resource requires transaction-specific authorization, it returns an HTTP error response
indicating that transaction authorization is required. This document defines the OAuth error code
`transaction_authorization_required` for this purpose.

For example:

~~~
HTTP/1.1 401 Unauthorized
WWW-Authenticate: Bearer error="transaction_authorization_required",
  error_description="Transaction-specific authorization is required",
  transaction_challenge="eyJhbGciOiJFUzI1NiIsInR5cCI6InR4bi1hdXRoei1jaGFsbGVuZ2Urand0..."
~~~
{: #fig-challenge-response title="Protected resource requesting transaction authorization"}

The `transaction_challenge` parameter contains a transaction authorization challenge encoded as a JWS Compact
Serialization object as specified in {{!JWS=RFC7515}}. The challenge MUST be signed by the protected resource.

The `error_description` parameter MAY be included to provide a human-readable diagnostic description. The
`error_description` parameter MUST NOT be used as the basis for an authorization decision.

A protected resource SHOULD use status code 401 when the request was understood but requires additional
transaction-specific authorization. Other status codes MAY be used when appropriate for the application
protocol.

### Challenge Representation

A transaction authorization challenge is a JWT secured using JWS. The JOSE header `typ` value MUST be
`txn-authz-challenge+jwt`. The `alg` value MUST identify an asymmetric digital signature algorithm.
The `alg` value MUST NOT be `none`.

The `kid` value in the JOSE header is used to identify the signing key in the protected resource's JWK Set.

The challenge claims identify the protected resource, the authorization server expected to process the
challenge, the transaction being authorized, and the intended audience of the resulting access token.

### Challenge Claims

A transaction authorization challenge MUST contain the following claims:

`iss`:
: Issuer claim defined in {{JWT}}. Identifier of the protected resource that generated the challenge. The
  access token issued in response to the challenge MUST use this value as its audience unless an
  application profile defines a different audience binding.

`aud`:
: Audience claim defined in {{JWT}}. Identifier of the authorization server expected to validate the challenge
  and issue an access token for the challenged operation. The value is selected by the protected resource
  and need not be the issuer of the access token presented with the original request. The protected resource
  MUST select an authorization server that it trusts to evaluate the challenge and issue access tokens
  for the requested operation.

`iat`:
: Issued At claim defined in {{JWT}}. Time at which the challenge was issued.

`exp`:
: Expiration Time claim defined in {{JWT}}. Time at which the challenge expires.

`jti`:
: JWT ID claim defined in {{JWT}}. Unique identifier for the challenge object.

`txn`:
: Transaction Identifier claim defined in {{!SET=RFC8417}}. Transaction identifier for the challenged operation.
   in response to the challenge MUST contain the same transaction identifier. The
  transaction identifier MUST be unique within the context of the protected resource for the lifetime of the
  challenge and any access token issued in response to it.

`authorization_details`:
: Claim containing Authorization Details as defined in {{OAUTH-RAR}}. Structured description of the operation
  for which transaction-specific authorization is requested. The granted authorization details carried by the
  access token issued in response to the challenge are derived from this value.

`reason`:
: Human-readable explanation of why transaction-specific authorization is required. This value is intended for
  display to the client, the user, or the approving party. The value is integrity protected as part of the
  challenge.

A transaction authorization challenge MAY contain the following claims:

`reason_uri`:
: URI identifying additional information about the transaction authorization request. The URI MUST be controlled by
  the protected resource or by a party trusted by the protected resource. The client and authorization server MAY
  dereference this URI to obtain additional information for display or policy evaluation.

`act`:
: Actor claim defined in {{!OAUTH-TOKEN-EXCHANGE=RFC8693}}. Information about the actor or delegation context
  associated with the request that caused the challenge. The value identifies the party that attempted the
  challenged operation, or the delegated actor acting on behalf of another party.

The challenge MAY contain additional claims. The authorization server MUST ignore claims it does not understand
unless local policy requires otherwise.

Information obtained from `reason_uri` MUST NOT override the security-relevant contents of the signed challenge
unless an application profile defines how that information is authenticated and bound to the challenge.

Application profiles MAY define additional claims that bind the challenge to the original request or to selected
security-relevant request components.

The following example shows the claims of a transaction authorization challenge:

~~~
{
  "iss": "https://resource.example.com",
  "aud": "https://as.example.com",
  "iat": 1710000000,
  "exp": 1710000300,
  "jti": "f1f7c8c4-2f8c-4c6a-83d1-example",
  "txn": "97053963-771d-49cc-a4e3-20aad399c312",
  "authorization_details": [
    {
      "type": "payment",
      "actions": ["initiate"],
      "locations": ["https://payments.example.com/accounts/123"],
      "instructedAmount": {
        "currency": "GBP",
        "amount": "5000.00"
      },
      "creditorName": "Example Ltd"
    }
  ],
  "reason": "Approval is required before initiating this payment.",
  "reason_uri": "https://resource.example.com/transactions/97053963-771d-49cc-a4e3-20aad399c312",
  "act": {
    "sub": "spiffe://example.com/aiagent/6526f880-1895-400b-a929-11a7eecf9753"
  }
}
~~~
{: #fig-challenge-example title="Example transaction authorization challenge"}

## Challenge Signing and Key Discovery

A protected resource that issues transaction authorization challenges MUST sign each challenge using an
asymmetric signing key. The client and authorization server MUST validate the challenge signature before using
any claim from the challenge for an authorization decision or for display to the user.

A protected resource that issues transaction authorization challenges MUST make the public keys needed to
validate those challenges available to clients and authorization servers. This document defines protected
resource metadata parameters, using the OAuth 2.0 Protected Resource Metadata mechanism described in
{{!OAUTH-PROT-METADATA=RFC9728}}. Deployments MAY also use preconfigured trust relationships or other
mechanisms to establish the same keying information.

This document defines the following protected resource metadata parameters:

`txn_challenge_jwks_uri`:
: URL of the protected resource's JSON Web Key Set containing public keys used to validate transaction authorization challenges.

`txn_challenge_signing_alg_values_supported`:
: JSON array containing the JWS `alg` values supported by the protected resource for transaction authorization challenges.

If a protected resource uses the same JWK Set for transaction authorization challenges and for other protected
resource signing keys, the value of `txn_challenge_jwks_uri` MAY be the same as another JWK Set URI published by
the protected resource.

When the challenge-signing key is published in the JWK Set identified by `txn_challenge_jwks_uri`, the `kid` value in the JOSE header of a
transaction authorization challenge MUST identify the signing key in that JWK Set.

If the challenge cannot be validated, the client or authorization server MUST NOT treat the challenge as authentic.

## Agent Relay

After receiving a transaction authorization challenge from a protected resource, the agent relays the challenge
to the client using a deployment-specific client-agent protocol.

The agent MUST relay the transaction authorization challenge without modifying it and MUST NOT replace it with
an agent-generated description of the requested operation. The client MUST treat any agent-supplied
description as advisory. Any authorization decision, user display, or disclosure decision MUST be based on
the validated challenge or on protected resource state identified by the validated challenge.

A client-agent protocol that carries transaction authorization challenges SHOULD distinguish a transaction
authorization challenge from ordinary agent messages or natural-language content. For example, such a protocol
can carry the challenge using a field named `transaction_challenge` whose value is the JWS Compact Serialization
of the transaction authorization challenge.

In deployments where one agent delegates work to another agent, a transaction authorization challenge MAY be
relayed through one or more intermediate agents before reaching the client. Each agent that relays
the challenge MUST relay it without modification. An intermediate agent MUST NOT replace the challenge with
a new challenge unless it is itself acting as a protected resource for a distinct operation and generates a
new signed transaction authorization challenge for that operation.

## Client Processing

Before presenting the challenge to the authorization server, the client MUST validate the challenge signature,
issuer, audience, and expiration. The client MUST verify that the `aud` claim identifies the authorization
server to which the client intends to present the challenge.

The client MAY display the challenge contents to the user before sending the challenge to the authorization server.
This allows the user to decide whether to continue and whether to disclose the challenged transaction to the authorization server.

When displaying challenge information, the client MUST use information obtained from the validated challenge or from
protected resource state identified by the validated challenge. The client MUST NOT rely on an unprotected description
supplied by the agent.

If the user declines to continue, or if the client cannot validate the challenge, the client MUST NOT present the challenge
to the authorization server.

## Authorization Server Processing {#authorization-server-processing}

The authorization server MUST validate the transaction authorization challenge before accepting it for processing or
issuing an access token.

At a minimum, the authorization server MUST verify that:

* the challenge signature is valid;

* the challenge was issued by a protected resource recognized by the authorization server;

* the challenge has not expired;

* the `aud` claim identifies the authorization server;

* the authorization server is willing to issue access tokens for the protected resource identified by the `iss`
  claim and for the requested operation;

* the access token issued in response to the challenge will use the protected resource identified by the `iss` claim
  as its audience, unless an application profile defines a different audience binding;

* the `txn` claim is present, is a string, and is acceptable;

* the requested operation is sufficiently described;

* the client is permitted to request transaction authorization for the challenged operation.

When requester context, such as the `act` claim, is present in the challenge, the authorization server MUST consider that
context when determining whether the challenged operation can be approved.

The authorization server determines the approving party according to local policy. The approving party can be the subject
associated with the original request, a different resource owner, an administrator, an organizational approval workflow,
or another policy authority.

The authorization server MUST obtain any required approval before issuing an access token. The authorization server
SHOULD present the approving party with sufficient information to understand the operation being authorized. The
authorization server MUST NOT rely on an unprotected description supplied by the agent as the basis for the
authorization decision.

If the authorization server accepts the challenge for processing, the client obtains the result using the transaction
authorization flow described in {{transaction-authorization-flow}}. If the authorization server rejects the challenge,
cannot validate the challenge, or cannot obtain the required approval, it MUST NOT issue an access token for the
challenged operation.

# Transaction Authorization Flow {#transaction-authorization-flow}

The client presents the transaction authorization challenge to the authorization server's token endpoint
({{Section 3.2 of OAUTH-FRAMEWORK}}) as an OAuth grant defined by this document. The approval required to satisfy
a challenge can involve a human user, resource owner, organizational workflow, or policy authority, and can take
longer than a single request-response exchange. This document does not define its own polling endpoint or polling
protocol; the authorization server completes such requests asynchronously using the OAuth Deferred Token Response
mechanism {{!DEFERRED=I-D.ietf-oauth-deferred-token-response}}, with the transaction authorization grant as the
originating grant.

A token response to the grant (returned immediately or, after deferral, on a polling request) authorizes the
challenged operation. A deferred response is not a token response: it only indicates that the authorization
server has accepted the challenge for processing. The challenged operation is authorized only when the
authorization server returns a token response and the protected resource accepts that token.

The following figure shows the transaction authorization flow when the request is deferred:

~~~aasvg
+--------+                         +----------------------+    +-----------------+
| Client |                         | Authorization Server |    | Approving Party |
+--------+                         +----------------------+    +-----------------+
    |                                         |                         |
    | POST /token  grant: txn-authz-challenge |                         |
    | transaction_challenge, deferred         |                         |
    |---------------------------------------->|                         |
    |                                         |                         |
    | 400 authorization_pending               |                         |
    | deferral_code, interval                 |                         |
    |<----------------------------------------|                         |
    |                                         | Approval Request        |
    |                                         |------------------------>|
    |                                         |                         |
    | POST /token  grant: deferred            |                         |
    | deferral_code                           |                         |
    |---------------------------------------->|                         |
    | 400 authorization_pending               |                         |
    |<----------------------------------------|                         |
    |                                         | Approval Result         |
    |                                         |<------------------------|
    |                                         |                         |
    | POST /token  grant: deferred            |                         |
    | deferral_code                           |                         |
    |---------------------------------------->|                         |
    | 200 token response (access token)       |                         |
    |<----------------------------------------|                         |
    |                                         |                         |
~~~
{: #fig-transaction-authorization-flow title="Transaction authorization flow using a deferred token response"}

## Transaction Authorization Grant

This document defines the grant type `urn:ietf:params:oauth:grant-type:txn-authz-challenge`. The client presents a
validated challenge (relayed to it by the agent) by making a token request to the token endpoint with the
following parameters in the `application/x-www-form-urlencoded` request body:

`grant_type`:
: REQUIRED. MUST be `urn:ietf:params:oauth:grant-type:txn-authz-challenge`.

`transaction_challenge`:
: REQUIRED. The transaction authorization challenge received from the protected resource.

`completion_mode`:
: OPTIONAL. Including the value `deferred`, as defined by {{DEFERRED}}, signals that the client accepts a deferred
  token response. Because the approval that satisfies a challenge is typically asynchronous, a client SHOULD
  include `deferred`. An authorization server MUST NOT return a deferred response to a client that has not
  signaled `deferred`. If the authorization server cannot approve the operation synchronously for such a client,
  it MUST return the `access_denied` error ({{Section 3.5 of ?OAUTH-DEVICE=RFC8628}}); the client can then repeat
  the request with `deferred` included in `completion_mode`.

`client_id`:
: REQUIRED if the client is not authenticating with the authorization server as described in
  {{Section 3.2.1 of OAUTH-FRAMEWORK}}.

The client authentication requirements of {{Section 3.2.1 of OAUTH-FRAMEWORK}} apply, and requests MUST use TLS
following the best practices of {{!BCP-195=RFC7525}}. The authorization server MUST validate the challenge as
described in {{authorization-server-processing}} before accepting the request.

For example, a public client, which {{DEFERRED}} requires to present a DPoP proof ({{DPOP}}) on this request, sends
the following request:

~~~
POST /token HTTP/1.1
Host: as.example.com
Content-Type: application/x-www-form-urlencoded
DPoP: eyJ0eXAiOiJkcG9wK2p3dCIsImFsZyI6IkVTMjU2IiwiandrIjp7Imt0eSI6IkVDIiwiY3J2IjoiUC0yNTYi...

grant_type=urn%3Aietf%3Aparams%3Aoauth%3Agrant-type%3Atxn-authz-challenge
&client_id=s6BhdRkqt3
&completion_mode=deferred
&transaction_challenge=eyJhbGciOiJFUzI1NiIsInR5cCI6InR4bi1hdXRoei1jaGFsbGVuZ2Urand0In0...
~~~
{: #fig-transaction-authorization-request title="Transaction authorization grant request"}

## Deferred Processing and Polling {#deferred-processing}

If the authorization server can approve the challenged operation without further interaction, it returns a token
response as described in {{successful-access-token-response}}.

Otherwise, if the client signaled `completion_mode=deferred`, the authorization server returns a deferred token
response as defined in {{DEFERRED}}: an HTTP 400 response whose body carries the `authorization_pending` or
`interaction_required` error code, a `deferral_code`, an `expires_in`, and an `interval`. The client then polls the
token endpoint using the deferred grant of {{DEFERRED}} (`grant_type=urn:ietf:params:oauth:grant-type:deferred`
with the `deferral_code`) until it receives a token response or a terminal error. The polling cadence, the
`interaction_required`, `slow_down`, `expired_token`, and `access_denied` errors, optional completion-callback
notifications, and cancellation via the revocation endpoint are all as defined in {{DEFERRED}}; this document does
not modify them.

When approval requires interaction that the client can facilitate, the authorization server returns
`interaction_required` with an `interaction_uri`, on the deferred response or on a later polling response, as
defined in {{DEFERRED}}. The client presents the `interaction_uri` to the user or other approving party, or opens
it in a user agent, and continues polling.

The `deferral_code` is sender-constrained as defined in {{DEFERRED}}; this protects the polling credential. If the
client presents a DPoP proof on the transaction authorization grant request, as {{DEFERRED}} requires of a public
client, {{DEFERRED}} binds both the `deferral_code` and the access token later issued on a polling request to the
client's DPoP key. Otherwise, the format and binding of the issued access token are as described in
{{successful-access-token-response}}. The `txn` value correlates the challenge, the issued token, and any
re-evaluation.

> Editor's note: The agent, not the client, presents the access token to the protected resource. An access token
> bound to the client's DPoP key cannot be used by an agent that does not hold that key. {{DEFERRED}} requires
> this binding in both its sender-constraint requirements and its sender-constraining continuity guarantee.
> Resolving this requires {{DEFERRED}} to let the originating grant define how the issued access token is bound,
> separately from the binding of the `deferral_code`.

For example, the authorization server returns a deferred response, and the client then polls:

~~~
HTTP/1.1 400 Bad Request
Content-Type: application/json
Cache-Control: no-store

{
  "error": "authorization_pending",
  "deferral_code": "LRGidcSAeVs_xYpEDt5pNx0A0TBr7qPHbh_Se6pfdf4",
  "expires_in": 300,
  "interval": 5
}
~~~
{: #fig-deferred-response title="Deferred token response"}

~~~
POST /token HTTP/1.1
Host: as.example.com
Content-Type: application/x-www-form-urlencoded
DPoP: eyJ0eXAiOiJkcG9wK2p3dCIsImFsZyI6IkVTMjU2IiwiandrIjp7Imt0eSI6IkVDIiwiY3J2IjoiUC0yNTYi...

grant_type=urn%3Aietf%3Aparams%3Aoauth%3Agrant-type%3Adeferred
&client_id=s6BhdRkqt3
&deferral_code=LRGidcSAeVs_xYpEDt5pNx0A0TBr7qPHbh_Se6pfdf4
~~~
{: #fig-deferred-poll title="Polling a deferred transaction authorization request"}

## Discovery

An authorization server indicates support for this mechanism by including
`urn:ietf:params:oauth:grant-type:txn-authz-challenge` in the `grant_types_supported` value of its metadata
{{!OAUTH-AS-METADATA=RFC8414}}, and by setting the `deferred_token_response_supported` metadata parameter defined in
{{DEFERRED}} to `true`.

## Successful Access Token Response {#successful-access-token-response}

If the transaction authorization request is approved, the authorization server returns an access token response.

The response uses the access token response format defined in {{Section 5.1 of OAUTH-FRAMEWORK}} and the
`authorization_details` response parameter defined in {{Section 7 of OAUTH-RAR}}. The granted
`authorization_details` value MUST describe the operation approved for the challenged transaction and MUST
be derived from the `authorization_details` claim of the transaction authorization challenge. The
authorization server MAY normalize, narrow, or otherwise constrain the granted authorization details
relative to those requested by the challenge, but MUST NOT broaden them.

The authorization_details parameter in the token response is returned by the authorization server as required
by Section 7 of {{OAUTH-RAR}} and conveys the authorization details as granted, which MAY differ from those
requested in the challenge. This response parameter is informational for the client. The protected resource
MUST determine authorization from the granted authorization details bound to the access token
(see {{access-token}}), and MUST NOT rely on the token response parameter.

The access token MUST contain the `txn` value from the transaction authorization challenge, MUST be bound
to the granted authorization details, and MUST use the `iss` value from the transaction authorization
challenge as its audience unless an application profile defines a different audience binding. The
authorization server MAY issue this access token in any access token format negotiated with the protected
resource; when JWT access tokens defined in {{?JWT-AT=RFC9068}} are used, the `txn` and
`authorization_details` claims are included as defined in {{SET}} and {{Section 9 of OAUTH-RAR}}
respectively.

The access token issued in response to a transaction authorization challenge is scoped to the challenged
operation and MUST NOT be accepted as authorization for any operation that is not described by its granted
authorization details.

For example:

~~~
HTTP/1.1 200 OK
Content-Type: application/json
Cache-Control: no-store

{
  "access_token": "eyJhbGciOiJFUzI1NiIsInR5cCI6ImF0K2p3dCJ9...",
  "token_type": "DPoP",
  "expires_in": 120,
  "authorization_details": [
    {
      "type": "payment",
      "actions": ["initiate"],
      "locations": ["https://payments.example.com/accounts/123"],
      "instructedAmount": {
        "currency": "GBP",
        "amount": "5000.00"
      },
      "creditorName": "Example Ltd"
    }
  ]
}
~~~
{: #fig-access-token-response title="Successful access token response"}


The access token MUST have a limited lifetime. The authorization server SHOULD issue the access token with a short lifetime because it represents
authorization for a specific operation. Sender-constrained access tokens, for example using DPoP
({{?DPOP=RFC9449}}) or mutual-TLS ({{?MTLS=RFC8705}}), MAY be used and are RECOMMENDED for high-impact
operations.

# Access Token {#access-token}

The access token issued in response to a transaction authorization challenge represents authorization for the challenged operation.
It is an OAuth 2.0 access token whose granted authorization details, expressed using the `authorization_details` parameter and claim
defined in {{OAUTH-RAR}}, describe the operation that was approved.

The access token is issued by the authorization server identified by the `aud` claim of the transaction authorization challenge and is
presented to the protected resource that issued the challenge.

The access token MUST contain sufficient information for the protected resource to determine that the token authorizes the challenged
operation. At a minimum, the access token MUST be bound to the `txn` value from the challenge and MUST carry granted authorization
details that describe the approved operation. The granted authorization details MAY be the value of the `authorization_details` claim
from the challenge, a value derived from it, or a reference to protected resource state agreed between the protected resource and the
authorization server.

When requester context, such as the `act` claim, is present in the transaction authorization challenge, the authorization server MUST
include equivalent requester context in the access token or otherwise bind the access token to that context.

This access token is scoped to the challenged operation. Clients and protected resources MUST NOT assume that the
access token issued in response to a transaction authorization challenge confers authorization beyond the operation described by its
granted authorization details.

## Token Presentation

The client provides the access token to the agent. The agent presents the access token to the protected resource together with the
request for the challenged operation.

In deployments where one agent delegates work to another agent, the access token MAY be relayed through one or more intermediate agents
before being presented to the protected resource. Each agent that relays the access token MUST relay it without modification. An agent
MUST NOT use the access token for a different transaction, different protected resource, or different requester context.

When HTTP is used, the access token MUST be presented to the protected resource as defined by {{OAUTH-FRAMEWORK}}. Bearer access
tokens are presented using the `Authorization` request header field as specified in {{?BEARER=RFC6750}}. Sender-constrained access
tokens are presented using the mechanism defined by the corresponding access token format, for example {{?DPOP=RFC9449}} or
{{?MTLS=RFC8705}}.

For example:

~~~
POST /payments HTTP/1.1
Host: resource.example.com
Authorization: DPoP eyJhbGciOiJFUzI1NiIsInR5cCI6ImF0K2p3dCJ9...
DPoP: eyJ0eXAiOiJkcG9wK2p3dCIsImFsZyI6IkVTMjU2IiwiandrIjp7Imt0eSI6IkVDIiwiY3J2IjoiUC0yNTYi...
Content-Type: application/json

{
  "amount": "5000.00",
  "currency": "GBP",
  "recipient": "Example Ltd"
}
~~~
{: #fig-access-token-presentation title="Presenting the access token to a protected resource"}

The agent MUST NOT modify the access token. If the agent cannot present the access token to the protected resource, the challenged
operation cannot be completed using this mechanism.

## Protected Resource Validation

Before accepting the access token as authorization for a challenged operation, the protected resource MUST validate the access token
according to {{OAUTH-FRAMEWORK}}, the access token format in use, and the requirements of this document.

At a minimum, the protected resource MUST verify that:

* the access token was issued by the authorization server identified by the `aud` claim of the transaction authorization challenge;

* the access token audience identifies the protected resource, unless an application profile defines a different audience binding;

* the access token has not expired;

* the access token is bound to the same `txn` value as the transaction authorization challenge;

* the granted authorization details associated with the access token authorize the requested operation and are consistent with the
  `authorization_details` claim of the transaction authorization challenge;

* any requester context required by the transaction authorization challenge is present and matches the challenged transaction;

* the access token has not previously been used, if the protected resource requires single-use access tokens for the challenged operation.

A protected resource MUST reject an access token that does not correspond to the transaction authorization challenge for the requested
operation.

A protected resource SHOULD treat access tokens issued in response to a transaction authorization challenge as single-use when the
challenged operation is non-idempotent or high impact. If single-use semantics are required, the protected resource MUST maintain
sufficient state to detect replay of the access token or transaction identifier.

The protected resource MUST NOT accept an access token issued in response to a transaction authorization challenge as general
authorization for operations other than the challenged operation. In particular, the protected resource MUST NOT honour the granted
authorization details of such an access token for any operation that is not the challenged operation described by those authorization
details.

# Security Considerations

Transaction authorization challenges and the access tokens issued in response to them are security-sensitive
artifacts. A transaction authorization challenge requests authorization for a specific operation, and the
access token issued in response represents evidence that the challenged operation was authorized. Implementations
need to ensure that these artifacts cannot be modified, replayed, substituted, or used for a different operation.

A protected resource MUST sign each transaction authorization challenge using an asymmetric signing key. Clients and
authorization servers MUST validate the challenge signature before using any claim from the challenge for display,
policy evaluation, or authorization decisions. If a challenge cannot be validated, it MUST NOT be treated as
authentic.

The authorization server MUST verify that the `aud` claim identifies the authorization server and that the protected
resource identified by the `iss` claim is trusted to request transaction authorization for the requested operation. An
authorization server MUST NOT issue an access token for a challenge issued by an unrecognized or unauthorized
protected resource.

The transaction authorization challenge and the resulting access token MUST be bound to the same transaction
identifier. The protected resource MUST verify that the `txn` value bound to the access token matches the `txn`
value from the challenge. An access token issued in response to a transaction authorization challenge MUST NOT be
accepted as authorization for any operation other than the challenged operation.

Access tokens issued in response to a transaction authorization challenge can be replayed if they are not
sufficiently constrained. Authorization servers MUST issue these access tokens with short lifetimes and SHOULD
issue them as sender-constrained access tokens, for example using DPoP ({{DPOP}}) or mutual-TLS ({{MTLS}}), for
high-impact operations. Protected resources SHOULD treat these access tokens as single-use for non-idempotent or
high-impact operations, and maintain sufficient state to detect replay where single-use semantics are required.

Because the access token issued in response to a transaction authorization challenge carries granted
authorization details that describe a specific operation, protected resources MUST evaluate those authorization
details against the operation being requested. Accepting such an access token without evaluating its
authorization details would allow a token issued for one operation to authorize a different operation.

The agent is not trusted to modify or summarize the challenge. Clients and authorization servers MUST NOT rely on an
unprotected description supplied by the agent as the basis for user display, policy evaluation, or authorization
decisions. Information presented to the user or approving party SHOULD be derived from the validated challenge, from
protected resource state identified by the challenge, or from information otherwise authenticated and bound to the
challenge.

When requester context is relevant to the authorization decision, the protected resource SHOULD include that context
in the challenge, for example using the `act` claim. The authorization server MUST consider requester context when it
is present. This helps prevent an approval obtained for one agent, client, user, or delegation context from being used
to authorize a transaction initiated by another requester.

The client can inspect the challenge before presenting it to the authorization server. This is important because the
challenge can contain sensitive information about the requested operation, user intent, protected resources, or
organizational policy. Clients SHOULD allow the user to decline before disclosing the challenge to the authorization
server when the challenge contains privacy-sensitive transaction details.

Challenges and the access tokens issued in response to them can reveal sensitive information if logged or exposed
to unintended parties. In particular, the granted authorization details carried by the access token can describe
the operation in detail. Implementations SHOULD minimize the information included in challenges and in the
granted authorization details of the access token, avoid logging them unless necessary, and protect them in
transit and at rest.

Application profiles that define additional challenge claims, request binding mechanisms, or alternative audience
bindings MUST describe how those extensions preserve challenge integrity, prevent replay and substitution, and avoid
confused-deputy attacks.


# IANA Considerations

This document registers the `Accept-Txn-Challenge` HTTP field name, one
OAuth error code, one OAuth parameter, one OAuth grant type, two OAuth Protected Resource
Metadata parameters, and two JWT claims. It relies on the deferred token response registrations (the
`completion_mode` and `interaction_uri` parameters, the deferred grant type, the deferral-code type, and the
token endpoint usage of the `interaction_required` error) defined by {{DEFERRED}}.

## HTTP Field Name Registration

IANA is requested to register the following field name in the "HTTP Field Name" registry {{IANA.HTTP.FieldNames}} as a structured Header Field:

Field Name:
: Accept-Txn-Challenge

Status:
: permanent

Structured Type:
: Item

Reference:
: this document

## OAuth Extensions Error Registration

IANA is requested to register the following error value in the "OAuth Extensions Error" registry {{IANA.OAuth.Parameters}}:

Error name:
: transaction_authorization_required

Usage location:
: resource access error response

Protocol extension:
: OAuth Transaction Authorization Challenge

Change controller:
: IETF

Reference:
: this document

## OAuth Parameter Registration

IANA is requested to register the following parameters in the "OAuth Parameters" registry {{IANA.OAuth.Parameters}}:

Name:
: transaction_challenge

Parameter Usage Location:
: WWW-Authenticate response, token request

Change controller:
: IETF

Reference:
: this document


## OAuth Protected Resource Metadata Registration

IANA is requested to register the following values in the "OAuth Protected Resource Metadata" registry {{IANA.OAuth.Parameters}}.

Metadata name:
: txn_challenge_jwks_uri

Metadata description:
: URL of the protected resource's JSON Web Key Set containing public keys used to validate transaction authorization challenges.

Change controller:
: IETF

Reference:
: this document

Metadata name:
: txn_challenge_signing_alg_values_supported

Metadata description:
: JSON array containing the JWS `alg` values supported by the protected resource for transaction authorization challenges.

Change controller:
: IETF

Reference:
: this document

## OAuth URI Registration

IANA is requested to register the following value in the "OAuth URI" registry {{IANA.OAuth.Parameters}}, in
accordance with {{!OAUTH-URI=RFC6755}}.

URN:
: urn:ietf:params:oauth:grant-type:txn-authz-challenge

Common Name:
: Grant type URI for presenting an OAuth transaction authorization challenge to the token endpoint.

Change controller:
: IETF

Reference:
: this document

## JSON Web Token Claims Registration

IANA is requested to register the following claims in the "JSON Web Token Claims" registry {{IANA.JSON.Web.Token}}.

Claim Name:
: reason

Claim Description:
: Human-readable explanation of why authorization, confirmation, or other processing is required.

Change Controller:
: IETF

Reference:
: this document

Claim Name:
: reason_uri

Claim Description:
: URI identifying additional information about why authorization, confirmation, or other processing is required.

Change Controller:
: IETF

Reference:
: this document


--- back

# Acknowledgments
{:numbered="false"}

TODO acknowledge.
