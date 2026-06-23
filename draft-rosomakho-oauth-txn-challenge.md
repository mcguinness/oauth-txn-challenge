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
 - DPoP
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
  AAUTH:
    title: "The AAuth Protocol"
    target: https://datatracker.ietf.org/doc/draft-hardt-oauth-aauth-protocol/
    author:
      -name: D. Hardt

--- abstract

This document defines an OAuth mechanism for transaction-specific authorization. A protected resource can
require additional authorization for a particular operation by returning a transaction authorization challenge:
a signed object that describes the operation, identifies the authorization server expected to authorize it, and
names the client key to which the resulting authorization is to be bound. This is useful when requests are mediated by
agents, automated workflows, or delegated services and the protected resource requires confirmation from a
human user, resource owner, or organizational authority. The client presents the challenge to the
authorization server with proof of possession of its key. The authorization server validates the challenge,
obtains any required approval, and issues an access token that is sender-constrained to the client's
key and narrowed to the authorized operation. The client presents this access token to the protected resource
as evidence that the challenged operation was authorized for that specific client.

--- middle

# Introduction

OAuth 2.1 ({{!OAUTH-FRAMEWORK=I-D.ietf-oauth-v2}}) access tokens authorize access to protected
resources. In many deployments, however, a protected resource cannot determine whether a
requested operation is acceptable based only on the access token that accompanies the request.
The access token might establish that the caller is authorized to interact with the protected
resource, but a particular operation can still require additional, transaction-specific
authorization.

This situation arises when the protected resource needs confirmation that a concrete transaction
has been approved by an appropriate party. The approving party might be the human user on whose
behalf the request is made, a different resource owner, or an organizational authority such as
an administrator, manager, data owner, or policy decision point.

This document defines a transaction authorization challenge mechanism. A protected resource
uses a transaction authorization challenge to request additional authorization for a specific operation. The
approach adapts the resource token concept from {{AAUTH}}: rather than relaying an opaque challenge to a separate
party for it to interpret, the protected resource names, in the challenge, the key that the requesting client
has demonstrated possession of, and the resulting authorization is cryptographically bound to that key. The
client presents the challenge to an authorization server while proving possession of its key. The
authorization server validates the challenge, obtains any required approval, and issues an access token
that is sender-constrained to the client's key, using DPoP {{!DPOP=RFC9449}} or mutual-TLS {{!MTLS=RFC8705}},
and narrowed to the authorized operation. The access token is then presented to the protected resource as
evidence that the challenged operation was authorized for that specific client.

The cryptographic binding to the client's key is the central property of this mechanism. It ensures that the
authorization obtained for an operation cannot be stolen, replayed, or used by a different party, and it
upgrades requester context from an asserted identifier to a verifiable proof of possession. The client's key
provides integrity and sender-constraint; it does not confer authority. The description of the operation
remains signed by the protected resource, and the decision to authorize the operation remains with the
authorization server and the approving party.

This mechanism is general and is not specific to any particular kind of client. It is described here in terms
of the OAuth client that makes the request, holds the proof-of-possession key, and uses the resulting access
token. Software agents, automated workflows, and delegated services are example clients; the agent-mediated
scenarios in {{use-cases}} motivate the mechanism but do not constrain it.

The key-bound flow above is the default. For requesters that cannot hold a key, and for topologies in which an
untrusted agent relays the request to a separate client, this document also defines a relay profile that issues a
bearer transaction token instead; see {{assurance-profiles}}. Where an operation is carried out by more than one
component, {{delegation}} describes how the authorization is re-bound to each component's key so that every hop
remains sender-constrained.

This mechanism is complementary to OAuth step-up authentication defined in {{?OAUTH-STEP-UP=RFC9470}}.
Step-up authentication enables a protected resource to require stronger or fresher authentication
of a user. A challenge instead requests authorization for a specific transaction. The approving
party can be different from the subject associated with the access token used for the original request.

## Use Cases {#use-cases}

The following scenarios motivate transaction-specific authorization. In each, a client acts on behalf of a
user or organization and the protected resource requires explicit approval for a concrete operation before
processing it.

### Human Approval for Mediated Actions

Software agents, automated workflows, and delegated services can perform operations on behalf of
users. Some operations are sufficiently sensitive that a protected resource might require explicit
approval before processing them. Examples include sending a payment, publishing content, deleting
data, modifying access policy, or disclosing sensitive information.

Local confirmation mechanisms within a client or agent framework can reduce risk, but they are not always
visible to the protected resource and do not necessarily produce authorization evidence that the
protected resource can validate. A challenge allows the protected resource to require explicit
authorization for the concrete operation and to receive an access token, bound to the client that will
act, representing that authorization.

### Authorization by a Different Resource Owner

The party that initiated an operation is not always the party authorized to approve it. For example,
a client acting on behalf of one user might request access to a resource owned or controlled by
another user. The protected resource can determine that approval from the resource owner, or
from another party authorized to act for that resource owner, is required before the operation
can proceed.

In this case, the challenge allows the protected resource to describe the requested operation and
the authorization server to determine the appropriate approving party. The resulting access token
represents authorization of the challenged operation by the party selected according to authorization
server policy.

### Organizational Approval

Some operations require approval by an organizational authority rather than by an individual end
user. Examples include approving access to regulated data, authorizing a production deployment,
granting elevated administrative access, or permitting data transfer to an external party.

A challenge allows the protected resource to request authorization evidence from
an authorization server or associated policy infrastructure. The authorization server validates the challenge, applies organizational policy, obtains any required approval, and issues an access token only when the
required authorization has been obtained.

# Conventions and Definitions

{::boilerplate bcp14-tagged}

This document uses the terms "client", "authorization server", "access token", "refresh token", and "protected
resource" as defined by {{OAUTH-FRAMEWORK}}.

This document uses the term "sender-constrained access token" and the confirmation claim (`cnf`) as defined by
{{!POP-KEY=RFC7800}}. It uses "DPoP proof", "DPoP-bound access token", and the JWK Thumbprint confirmation
method (`jkt`) as defined by {{DPOP}}, and the certificate-bound access token and the `x5t#S256` confirmation
method as defined by {{MTLS}}.

This document defines the following additional terms:

Approving Party:
: The human user, resource owner, organizational authority, or policy authority whose approval is required
  before the challenged transaction can proceed.

Client Key:
: The asymmetric key controlled by the client, of which the client proves possession on requests to the protected
  resource and the authorization server, and to which the issued access token is sender-constrained. The client is
  typically an autonomous component such as an agent, automated workflow, or delegated service, but the mechanism
  does not depend on this.

Proof of Possession:
: A demonstration that the client controls the private portion of the client key. This document uses the
  sender-constraining mechanisms of {{DPOP}} or {{MTLS}}; see {{client-key-pop}}.

Transaction Authorization Challenge:
: A signed object issued by a protected resource that describes an operation requiring transaction-specific
  authorization, identifies the authorization server expected to authorize it, and names the client key to which
  the resulting authorization is to be bound. A transaction authorization challenge is a request for
  authorization; it does not itself grant access.

# Architecture

A transaction authorization flow involves a protected resource, a client, an authorization server, and an
approving party.

The client is the party that makes the request, controls the client key, and uses the resulting access token.
It proves possession of the client key, using a sender-constraining mechanism (see {{client-key-pop}}), on its
requests to both the protected resource and the authorization server. The client is the party to which the
authorization is bound. In agent-mediated deployments the client may itself be an autonomous agent or may act on
behalf of one; the protocol role is the same.

The protected resource receives a request and determines whether the authorization currently presented
with the request is sufficient. If the protected resource determines that the requested operation requires
transaction-specific authorization, it observes the client key from the proof of possession on the request and
returns a challenge that names that key. The protected resource describes the operation in the challenge and signs it; the client cannot modify this description.

The authorization server validates the challenge, determines the approving party, obtains any required
approval, and issues a sender-constrained access token. The authorization server can use local policy, resource
metadata, resource-owner information, organizational policy, or other authorization context to determine
whether the requested operation can be approved. The authorization server is the single point at which the
operation is authorized; the client's possession of a key does not grant it authority to approve the operation
on its own behalf.

The approving party is the human user, resource owner, organizational authority, or policy authority whose
approval is required. The approving party interacts with the authorization server out of band, not with the
client. The approving party can be the same subject on whose behalf the client is acting, but this is not
required.

Some deployments bundle several distinct roles into a single "client". This mechanism keeps them separate, which
is what allows the approval experience to be hosted somewhere other than the acting component:

* the *key-holder*, which makes the request, proves possession of the client key, and uses the resulting access
  token. This is the actor identified by the challenge `cnf` claim.

* the *approval surface*, which presents the operation to the approving party and gathers a decision. In this
  mechanism the approval surface is reached through the authorization server's `authorization_uri`, which the
  authorization server MAY direct to a wallet, native application, or other user agent.

* the *approver*, which is the authorization server applying policy together with the approving party.

A deployment that requires the approval surface to be a component separate from the key-holder is supported: the
key-holder still proves possession of the client key, while the human-facing interaction occurs at the
authorization server or a user agent it directs to. A deployment that instead requires a separate OAuth client,
distinct from the key-holder, to drive the authorization server exchange is better served by relaying a challenge without a `cnf` claim; see {{assurance-profiles}}.

The cryptographic binding chain is the core of this architecture:

1. The protected resource observes the client key on the initial request and records its confirmation method
   in the `cnf` claim of the challenge.

1. The authorization server enforces that the client presenting the challenge proves possession of the
   key identified by that `cnf` claim.

1. The authorization server sender-constrains the issued access token to that same key by setting the access
   token's `cnf` confirmation method to the same value.

1. The protected resource enforces proof of possession of that key when the access token is presented for the
   challenged operation.

Because the same key is bound at every step, an access token obtained for an operation is useless to any party
other than the client that obtained it.

The following figure shows the basic protocol flow:

~~~aasvg
+----------------------+        +--------+        +--------------------+
| Authorization Server |        | Client |<------>| Protected Resource |
+----------------------+        +--------+        +--------------------+
            ^                        |
            |   challenge           |
            |<-----------------------|
            |
            | approval interaction
            v
+-----------------+
| Approving Party |
+-----------------+
~~~
{: #fig-architecture title="Overall architecture of transaction authorization with challenges"}

The flow is as follows:

1. The client sends a request to the protected resource using its existing authorization context and proving
   possession of the client key.

1. The protected resource determines that the request requires transaction-specific authorization.

1. The protected resource issues a challenge, naming the client key, and returns it to the client.

1. The client presents the challenge to the authorization server, proving possession of the client key.

1. The authorization server validates the challenge, determines the approving party, and obtains any required approval.

1. The authorization server issues a sender-constrained access token to the client.

1. The client retries or continues the request and presents the access token to the protected resource, again proving possession of the client key.

1. The protected resource validates the access token and processes the request if the token authorizes the
   challenged operation for that client.

# Client Key and Proof of Possession {#client-key-pop}

Throughout this flow the client proves possession of the client key, and the access token issued to the client
is sender-constrained to that key. This document does not define a new proof-of-possession mechanism; it uses
existing OAuth sender-constraining mechanisms. A conforming client and authorization server MUST support
DPoP {{DPOP}}. A deployment MAY use mutual-TLS {{MTLS}} instead where all participants support it.

The mechanism in use determines how the client key is identified in a `cnf` claim ({{POP-KEY}}), how the client
demonstrates possession on a request, and how the access token is presented:

DPoP {{DPOP}}:
: The client key is identified by its JWK Thumbprint ({{!JWK-THUMBPRINT=RFC7638}}) in the `jkt` confirmation
  method. The client demonstrates possession by including a DPoP proof in the `DPoP` header field. The access
  token has `token_type` `DPoP` and is presented using the `DPoP` authentication scheme. This is the default and
  the mandatory-to-implement mechanism.

mutual-TLS {{MTLS}}:
: The client key is the key of a client certificate, identified by the certificate SHA-256 thumbprint in the
  `x5t#S256` confirmation method. The client demonstrates possession by establishing a mutual-TLS connection
  using that certificate. The access token has `token_type` `Bearer` and is presented using the `Bearer`
  authentication scheme over the mutual-TLS connection.

A protected resource and authorization server MUST use the same mechanism, and therefore the same form of `cnf`
confirmation, for a given transaction: the confirmation method the protected resource records in the challenge determines the proof the authorization server and protected resource subsequently require. The remainder
of this document is written in terms of the client key, the `cnf` claim, and proof of possession; it uses DPoP
in examples.

The client key SHOULD be the same key to which the client's existing access token is bound, so that a single key
identifies the client across the original request, the transaction authorization flow, and the retry.

# Assurance Profiles {#assurance-profiles}

Not every requesting component can hold a key and prove possession of it, and not every deployment uses the
keyed-principal topology. This document therefore defines two profiles. A protected resource selects the profile
per operation, according to the sensitivity of the operation and not according to what a particular request
happens to present.

Key-bound profile:
: The challenge contains a `cnf` claim, the authorization server issues a sender-constrained access token,
  and the entire flow is bound to the client key as described in this document. This is the default profile and
  the one the remainder of this document describes.

Relay profile:
: The challenge does not contain a `cnf` claim, and the flow does not require the client to hold or prove
  possession of a key. This profile supports requesters that cannot prove possession of a key, and the topology in
  which an untrusted agent relays the challenge to a separate client that drives the authorization server
  exchange. The authorization server issues a transaction token
  ({{!TXN-TOKENS=I-D.ietf-oauth-transaction-tokens}}), using the transaction token response format of
  {{TXN-TOKENS}}, rather than a sender-constrained access token. The client presents the transaction token using
  the `Txn-Token` header field defined by {{TXN-TOKENS}}, alongside the access token it already uses for the
  protected resource. Because the transaction token is a bearer credential, the protected resource MUST treat it
  as single-use and maintain sufficient state to detect replay. Because the relay profile has no `cnf` claim,
  redemption is bound to the intended client by the challenge `client_id` claim, when present, together with
  client authentication at the authorization server.

Except where a profile is named explicitly, the normative requirements in the remainder of this document describe
the key-bound profile. In the relay profile, the requirements concerning the client key, proof of possession, the
`cnf` claim, and sender-constraint of the issued token do not apply and are replaced by the behavior described in
this section; all other requirements -- including challenge signing and validation, the determination of the
approving party, the transaction authorization endpoint, and the polling flow -- apply unchanged. In the relay
profile the authorization server binds the pending request and any `transaction_authorization_id` to the
authenticated client rather than to a proof-of-possession key.

The selected profile is integrity protected: it is determined by the presence or absence of the `cnf` claim in
the protected-resource-signed challenge. A relaying agent therefore cannot downgrade a key-bound operation
to the relay profile by stripping the `cnf` claim, because doing so invalidates the signature, nor upgrade or
retarget the binding by inserting one. An authorization server MUST honor the profile indicated by the validated
challenge and MUST NOT issue an unbound transaction token for a challenge that carries a `cnf` claim.

A protected resource SHOULD require the key-bound profile for high-impact or non-idempotent operations, and
SHOULD restrict use of the relay profile to deployments where the weaker, single-use bearer protection is
acceptable for the operation in question.

# Transaction Authorization Challenge {#challenge}

A transaction authorization challenge is a signed JWT ({{!JWT=RFC7519}}) generated by a protected resource to request
transaction-specific authorization for a particular operation. The challenge describes the operation to be
authorized, identifies the authorization server expected to authorize it, names the client key to which the
resulting authorization is to be bound, and contains freshness and integrity protection.

The challenge is consumed by the authorization server and can also be validated by the client before the
client presents it to the authorization server. The client cannot modify the challenge or substitute its own
description of the requested operation.

A challenge is a request for authorization. It is not a bearer credential and does not itself grant access
to the protected resource.

## Challenge Capability Signal

A client indicates that it supports the challenge mechanism by sending the `Accept-Txn-Challenge` header
field with a true value in requests to a protected resource. To obtain a key-bound challenge
({{assurance-profiles}}), a client that sends this header MUST also prove possession of its key on the request,
using one of the mechanisms in {{client-key-pop}}, so that the protected resource can observe the client key. A
client that cannot prove possession of a key can obtain only a relay-profile challenge.

In the key-bound profile, this specification assumes that the client's existing access token is itself
sender-constrained to the client key, so that a single key identifies the client across the original request, the
transaction authorization flow, and the retry. When DPoP is used, the DPoP proof on the initial request is bound
to the presented access token using the `ath` claim as described in {{DPOP}}; when mutual-TLS is used, the access
token is bound to the client certificate of the mutual-TLS connection. If a client holds only a bearer access
token that is not sender-constrained, it MUST establish a sender-constrained access token, for example by
obtaining one from its authorization server, before it can use the key-bound profile; otherwise it is limited to
the relay profile.

The `Accept-Txn-Challenge` header field is an Item Structured Field; see {{Section 3.3 of !STRUCTURED-FIELDS=RFC9651}}.
Its value MUST be a Boolean. Any other value type MUST be handled by recipients as if the field were not present.
For example, if this field is included multiple times, its type will become a List and the field will be ignored.

This document does not define any parameters for the `Accept-Txn-Challenge` header field value, but future
documents might define parameters. Receivers MUST ignore unknown parameters.

The following example indicates support for challenges and proves possession of the client key:

~~~
GET /accounts/123 HTTP/1.1
Host: resource.example.com
Authorization: DPoP mF_9.B5f-4.1JqM
Accept-Txn-Challenge: ?1
DPoP: eyJ0eXAiOiJkcG9wK2p3dCIsImFsZyI6IkVTMjU2IiwiandrIjp7Li4ufX0...
~~~
{: #fig-challenge-capability title="Client indicating support and proving possession of the client key"}

An `Accept-Txn-Challenge` header field with a false value has the same semantics as when the header field is not present.

A client MUST NOT include the `Accept-Txn-Challenge` header field unless it can complete one of the profiles in
{{assurance-profiles}}: either it controls a client key, can prove possession of that key to both the protected
resource and the authorization server, and can present the resulting access token to the protected resource
(key-bound profile); or it can relay the challenge to a client that drives the authorization server exchange
(relay profile).

A protected resource MUST NOT return a challenge unless the request includes an `Accept-Txn-Challenge`
header field with a true value, or the protected resource has out-of-band knowledge that the client supports this
specification. To return a key-bound challenge, the protected resource additionally requires a verifiable
proof of possession of the client key on the request, or another means to determine the client key; absent that,
it can return only a relay-profile challenge.

## Challenge Response

When a protected resource requires transaction-specific authorization, it returns an HTTP error response
indicating that transaction authorization is required. This document defines the OAuth error code
`transaction_authorization_required` for this purpose.

In the key-bound profile, before issuing the challenge, the protected resource MUST validate the proof of
possession on the request as described in {{client-key-pop}} and determine the confirmation method of the
confirmed key. The protected resource sets the `cnf` claim of the challenge to the corresponding
confirmation: a `jkt` member for DPoP, or an `x5t#S256` member for mutual-TLS. The protected resource SHOULD use
the authentication scheme of the proof-of-possession mechanism in use (`DPoP` or `Bearer`) in the
`WWW-Authenticate` header field. In the relay profile, the protected resource omits the `cnf` claim (see
{{assurance-profiles}}).

For example, using DPoP:

~~~
HTTP/1.1 401 Unauthorized
WWW-Authenticate: DPoP error="transaction_authorization_required",
  error_description="Transaction-specific authorization is required",
  transaction_challenge="eyJhbGciOiJFUzI1NiIsInR5cCI6InR4bi1hdXRoei1jaGFsbGVuZ2Urand0In0..."
~~~
{: #fig-challenge-response title="Protected resource requesting transaction authorization"}

The `transaction_challenge` parameter contains a challenge encoded as a JWS Compact Serialization object as
specified in {{!JWS=RFC7515}}. The challenge MUST be signed by the protected resource.

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

The challenge claims identify the protected resource, the authorization server expected to authorize the
operation, the client key to which the resulting authorization is to be bound, and the operation being authorized.

### Challenge Claims

A challenge MUST contain the following claims:

`iss`:
: Issuer claim defined in {{JWT}}. Identifier of the protected resource that generated the challenge. The
  access token issued in response to the challenge MUST use this value as its audience unless an
  application profile defines a different audience binding.

`aud`:
: Audience claim defined in {{JWT}}. Identifier of the authorization server expected to validate the challenge and issue an access token for the challenged operation. The value is selected by the protected resource
  and need not be the issuer of the access token presented with the original request. The protected resource
  MUST select an authorization server that it trusts to evaluate the challenge and issue access tokens
  for the requested operation.

`cnf`:
: Confirmation claim defined in {{POP-KEY}}, identifying the client key the protected resource confirmed on the
  request that triggered the challenge. This claim is REQUIRED in the key-bound profile and absent in the
  relay profile; its presence selects the profile (see {{assurance-profiles}}). The member is `jkt` for DPoP or
  `x5t#S256` for mutual-TLS, as described in {{client-key-pop}}. The `cnf` claim has a dual role here: the client
  that presents the challenge to the authorization server MUST prove possession of this key, and the access
  token the authorization server issues in response MUST be sender-constrained to the same key. This claim thus
  binds the authorization request, and the resulting access token, to a single client.

`iat`:
: Issued At claim defined in {{JWT}}. Time at which the challenge was issued.

`exp`:
: Expiration Time claim defined in {{JWT}}. Time at which the challenge expires. Because the challenge
  describes a single pending operation, protected resources SHOULD use short expiration times.

`jti`:
: JWT ID claim defined in {{JWT}}. Unique identifier for the challenge.

`txn`:
: Transaction Identifier claim defined in {{!SET=RFC8417}}. Transaction identifier for the challenged operation.
  The access token issued in response to the challenge MUST be associated with the same transaction
  identifier. The transaction identifier MUST be unique within the context of the protected resource for the
  lifetime of the challenge and any access token issued in response to it. The `txn` value is the stable
  identifier that correlates the challenge, the pending transaction authorization request, the issued token, and
  any subsequent re-evaluation of the operation; clients and protected resources MAY use it as a handle to track
  the operation across these steps.

`authorization_details`:
: Claim containing Authorization Details as defined in {{!OAUTH-RAR=RFC9396}}. Structured description of the operation
  for which transaction-specific authorization is requested.

`reason`:
: Human-readable explanation of why transaction-specific authorization is required. This value is intended for
  display to the client, the user, or the approving party. The value is integrity protected as part of the
  challenge.

A challenge MAY contain the following claims:

`client_id`:
: Identifier of the OAuth client that the protected resource expects to redeem the challenge, as observed
  from the authorization context of the request that triggered the challenge. When this claim is present, the
  authorization server MUST verify that the authenticated client (see {{Section 3.2.1 of OAUTH-FRAMEWORK}}) matches
  this value before issuing a token. This binds redemption of the challenge to a specific authenticated
  client. In the key-bound profile it supplements the proof-of-possession binding established by the `cnf` claim,
  and is most useful for confidential clients whose `client_id` is accompanied by client authentication. In the
  relay profile, where there is no `cnf` claim, this claim is the principal means of binding redemption to a
  specific client, and a protected resource that requires such binding SHOULD include it.

`act`:
: Actor claim defined in {{!OAUTH-TOKEN-EXCHANGE=RFC8693}}. Identity of the party acting in the request that
  triggered the challenge, for cases where the authorization server's policy or the approving party needs to
  know who is acting and not only that the requester holds the confirmed key. The `cnf` claim binds the request to
  a key and defeats confused-deputy attacks; the `act` claim conveys actor identity for policy and display. When
  present, the authorization server MUST consider this claim when determining whether the challenged operation can
  be approved. This is particularly relevant where the approving party is a different resource owner or an
  organizational authority that authorizes based on who is acting.

`reason_uri`:
: URI identifying additional information about the transaction authorization request. The URI MUST be controlled by
  the protected resource or by a party trusted by the protected resource. The client and authorization server MAY
  dereference this URI to obtain additional information for display or policy evaluation.

The challenge MAY contain additional claims. The authorization server MUST ignore claims it does not understand
unless local policy requires otherwise.

Information obtained from `reason_uri` MUST NOT override the security-relevant contents of the signed challenge
unless an application profile defines how that information is authenticated and bound to the challenge.

Application profiles MAY define additional claims that bind the challenge to the original request or to selected
security-relevant request components.

The following example shows the claims of a challenge:

~~~
{
  "iss": "https://resource.example.com",
  "aud": "https://as.example.com",
  "client_id": "s6BhdRkqt3",
  "cnf": {
    "jkt": "0ZcOCORZNYy-DWpqq30jZyJGHTN0d2HglBV3uiguA4I"
  },
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
  "reason_uri": "https://resource.example.com/transactions/97053963-771d-49cc-a4e3-20aad399c312"
}
~~~
{: #fig-challenge-example title="Example transaction authorization challenge"}

## Challenge Signing and Key Discovery

A protected resource that issues challenges MUST sign each challenge using an
asymmetric signing key. The client and authorization server MUST validate the challenge signature before using
any claim from the challenge for an authorization decision or for display to the user.

A protected resource that issues challenges MUST make the public keys needed to
validate those challenges available to clients and authorization servers. This document defines protected
resource metadata parameters, using the OAuth 2.0 Protected Resource Metadata mechanism described in
{{!OAUTH-PROT-METADATA=RFC9728}}. Deployments MAY also use preconfigured trust relationships or other
mechanisms to establish the same keying information.

This document defines the following protected resource metadata parameters:

`txn_challenge_jwks_uri`:
: URL of the protected resource's JSON Web Key Set containing public keys used to validate challenges.

`txn_challenge_signing_alg_values_supported`:
: JSON array containing the JWS `alg` values supported by the protected resource for challenges.

If a protected resource uses the same JWK Set for challenges and for other protected
resource signing keys, the value of `txn_challenge_jwks_uri` MAY be the same as another JWK Set URI published by
the protected resource.

When the challenge-signing key is published in the JWK Set identified by `txn_challenge_jwks_uri`, the `kid` value in the JOSE header of a
challenge MUST identify the signing key in that JWK Set.

If the challenge cannot be validated, the client or authorization server MUST NOT treat the challenge as authentic.

## Client Processing

After receiving a challenge from a protected resource, the client presents it to the authorization server
identified by the `aud` claim, as described in {{transaction-authorization-flow}}.

Before presenting the challenge to the authorization server, the client MUST validate the challenge
signature, issuer, audience, and expiration. The client MUST verify that the `aud` claim identifies the
authorization server to which the client intends to present the challenge. In the key-bound profile, the
client MUST verify that the `cnf` claim identifies the client key it controls; if it does not, the client MUST NOT
present the challenge, because the resulting access token would be bound to a key the client cannot prove
possession of. If the challenge contains a `client_id` claim, the client MUST verify that it matches its own
client identifier.

The client MAY display the challenge contents to the user before sending it to the authorization server.
This allows the user to decide whether to continue and whether to disclose the challenged operation to the
authorization server. When displaying information about the operation, the client MUST use information obtained
from the validated challenge or from protected resource state identified by the validated challenge.

If the user declines to continue, or if the client cannot validate the challenge, the client MUST NOT present the challenge
to the authorization server.

In the key-bound profile, where work is delegated across multiple components, the component that presents the challenge to the
authorization server and ultimately uses the access token MUST be the one whose key is named by the `cnf` claim.
A client MUST NOT present a challenge bound to a key it does not control. Delegating an operation to a
different component therefore requires that component to trigger its own challenge, bound to its own key, or to use
the delegation mechanism in {{delegation}}.

## Authorization Server Processing {#authorization-server-processing}

The authorization server MUST validate the challenge before accepting it for processing or
issuing an access token.

At a minimum, the authorization server MUST verify that:

* the challenge signature is valid;

* the challenge was issued by a protected resource recognized by the authorization server;

* the challenge has not expired;

* the `aud` claim identifies the authorization server;

* in the key-bound profile, the client presenting the challenge proves possession of the key identified by the `cnf` claim, as described in {{client-key-pop}} and {{transaction-authorization-flow}};

* if the challenge contains a `client_id` claim, the authenticated client matches that value;

* the authorization server is willing to issue access tokens for the protected resource identified by the `iss`
  claim and for the requested operation;

* the access token issued in response to the challenge will use the protected resource identified by the `iss` claim
  as its audience, unless an application profile defines a different audience binding;

* the `txn` claim is present, is a string, and is acceptable;

* the requested operation is sufficiently described;

* the client is permitted to request transaction authorization for the challenged operation.

The `cnf` claim provides verifiable requester context. The authorization server MUST consider this context
when determining whether the challenged operation can be approved, and MUST sender-constrain the issued access
token to the same key. This prevents an approval obtained for one client from being used by another.

The authorization server determines the approving party according to local policy. The approving party can be the subject
associated with the original request, a different resource owner, an administrator, an organizational approval workflow,
or another policy authority.

The authorization server MUST obtain any required approval before issuing an access token. The approval interaction
takes place between the authorization server and the approving party, out of band from the client. The authorization
server SHOULD present the approving party with sufficient information to understand the operation being authorized. The
authorization server MUST NOT rely on a description supplied by the client as the basis for the authorization decision;
it MUST use the operation description in the validated challenge, or protected resource state identified by it.
The client's possession of the client key authenticates the client and binds the resulting access token; it does not
authorize the operation.

If the authorization server accepts the challenge for processing, the client obtains the result using the transaction
authorization flow described in {{transaction-authorization-flow}}. If the authorization server rejects the challenge,
cannot validate the challenge, or cannot obtain the required approval, it MUST NOT issue an access token for the
challenged operation.

# Transaction Authorization Flow {#transaction-authorization-flow}

The approval required to satisfy a challenge can require interaction with a human user,
resource owner, organizational workflow, or policy authority. Such interaction can take longer than a single HTTP
request-response exchange. Therefore, this document defines an asynchronous polling flow based on the style of
the OAuth 2.0 Device Authorization Grant defined in {{!OAUTH-DEVICE=RFC8628}}.

Unlike the Device Authorization Grant, this flow does not use a device code, user code, or verification URI. Instead,
the client submits a challenge to the authorization server. If the authorization server
accepts the challenge for processing, it returns a transaction authorization identifier. The client then polls the transaction
authorization endpoint with that identifier until the authorization server returns an access token or an error.

In the key-bound profile, every request the client makes to the transaction authorization endpoint MUST prove
possession of the client key, using the mechanism in {{client-key-pop}} that corresponds to the `cnf` claim of the
submitted challenge: a DPoP proof in the `DPoP` header field, or a mutual-TLS connection using the client
certificate. The authorization server MUST verify that the key the client demonstrates possession of equals the
key identified by the `cnf` claim of the submitted challenge. This binds the entire flow to the client
identified by the protected resource.

A successful transaction authorization response does not indicate that the challenged operation has been approved.
It only indicates that the authorization server has accepted the challenge for processing.
The challenged operation is authorized only when the authorization server issues an access token and the protected
resource accepts that token for the challenged operation.

The following figure shows the transaction authorization flow:

~~~aasvg
+--------+                         +----------------------+    +-----------------+
| Client |                         | Authorization Server |    | Approving Party |
+--------+                         +----------------------+    +-----------------+
    |                                         |                         |
    | Transaction Authorization Request       |                         |
    | transaction_challenge (+ DPoP proof)    |                         |
    |---------------------------------------->|                         |
    |                                         |                         |
    |      Transaction Authorization Response |                         |
    |  transaction_authorization_id, interval |                         |
    |<----------------------------------------|                         |
    |                                         |                         |
    |                                         | Approval Request        |
    |                                         |------------------------>|
    |                                         |                         |
    | Transaction Authorization Poll          |                         |
    | transaction_authorization_id (+ DPoP)   |                         |
    |---------------------------------------->|                         |
    |                                         |                         |
    |       authorization_pending / slow_down |                         |
    |<----------------------------------------|                         |
    |                                         |                         |
    |                                         |         Approval Result |
    |                                         |<------------------------|
    |                                         |                         |
    | Transaction Authorization Poll          |                         |
    | transaction_authorization_id (+ DPoP)   |                         |
    |---------------------------------------->|                         |
    |                                         |                         |
    |      Transaction Authorization Response |                         |
    |     DPoP-bound access token             |                         |
    |<----------------------------------------|                         |
    |                                         |                         |
~~~
{: #fig-transaction-authorization-flow title="Transaction authorization flow"}

## Transaction Authorization Request

This specification defines a new OAuth endpoint: the transaction authorization endpoint. The authorization server MUST
publish the location of the transaction authorization endpoint using the `transaction_authorization_endpoint` authorization
server metadata parameter defined by this document.

The client makes a transaction authorization request to the transaction authorization endpoint by sending a POST request
with the following parameters using the `application/x-www-form-urlencoded` format with a character encoding of
UTF-8 in the HTTP request entity-body:

`client_id`:
: REQUIRED if the client is not authenticating with the authorization server as described in {{Section 3.2.1 of OAUTH-FRAMEWORK}}.
  The client identifier issued to the client during the registration process.

`transaction_challenge`:
: REQUIRED. The challenge received from the protected resource.

In the key-bound profile, the request MUST prove possession of the client key as described in {{client-key-pop}},
corresponding to the `cnf` claim of the challenge: a DPoP proof in the `DPoP` header field when DPoP is used,
or a mutual-TLS connection using the client certificate when mutual-TLS is used.

For example, the client makes the following HTTPS request using DPoP:

~~~
POST /txn-authorization HTTP/1.1
Host: as.example.com
Content-Type: application/x-www-form-urlencoded
DPoP: eyJ0eXAiOiJkcG9wK2p3dCIsImFsZyI6IkVTMjU2IiwiandrIjp7Li4ufX0...

client_id=s6BhdRkqt3
&transaction_challenge=eyJhbGciOiJFUzI1NiIsInR5cCI6InR4bi1hdXRoei1jaGFsbGVuZ2Urand0In0...
~~~
{: #fig-transaction-authorization-request title="Transaction authorization request"}

Requests to the transaction authorization endpoint MUST use the Transport Layer Security (TLS) protocol {{?TLS=I-D.ietf-tls-rfc8446bis}}
and implement the best practices of {{!BCP-195=RFC7525}}.

The client authentication requirements of {{Section 3.2.1 of OAUTH-FRAMEWORK}} apply to requests on this endpoint, which means
that confidential clients (those that have established client credentials) authenticate in the same manner as when making requests
to the token endpoint, and public clients provide the `client_id` parameter to identify themselves. Client authentication is
distinct from, and in addition to, proof of possession of the client key: the former authenticates the client as a registered
OAuth client, while the latter binds the request to the key named in the challenge.

The authorization server MUST validate the challenge as described in {{authorization-server-processing}} and, in the
key-bound profile, MUST verify that the client proves possession of the key identified by the `cnf` claim before accepting the
request for processing.

## Transaction Authorization Response

After receiving a transaction authorization request, the authorization server validates the challenge as
described in {{authorization-server-processing}}. The authorization server then either issues an access token, indicates that
the transaction authorization request is pending, or returns an error response.

If the authorization server approves the challenged operation without additional interaction, it returns an access token
response as described in {{successful-access-token-response}}.

If additional interaction or policy evaluation is required, the authorization server returns an HTTP 200 response with an
`application/json` body containing the following parameters:

`transaction_authorization_id`:
: REQUIRED. A server-generated identifier used by the client to continue or poll the transaction authorization request.

`expires_in`:
: REQUIRED. Lifetime in seconds of the pending transaction authorization request maintained by the authorization server.

`interval`:
: OPTIONAL. Minimum amount of time in seconds that the client SHOULD wait between polling requests. If omitted, the client SHOULD use 5 seconds.

`authorization_uri`:
: OPTIONAL. URI that the authorization server can use to drive an interactive approval or authentication step with the
  approving party. The client MAY present this URI to the user or open it in a user agent. This URI is used when the
  authorization server requires an interactive approval or authentication step.

The authorization server MUST bind the `transaction_authorization_id` to the client that initiated the transaction authorization request,
including its proof-of-possession key.

The authorization server MAY include additional members in this response, and application profiles MAY define
additional members, for example to convey the status of outstanding prerequisites for the operation. Clients MUST
ignore members they do not recognize.

A successful response containing `transaction_authorization_id` does not indicate that the challenged operation has been approved.
It only indicates that the authorization server has accepted the transaction authorization request for processing.

For example:

~~~
HTTP/1.1 200 OK
Content-Type: application/json
Cache-Control: no-store

{
  "transaction_authorization_id": "txn-authz-abc123",
  "expires_in": 300,
  "interval": 5
}
~~~
{: #fig-transaction-authorization-response title="Transaction authorization pending response"}

The following example includes an `authorization_uri` for an authorization interaction:

~~~
HTTP/1.1 200 OK
Content-Type: application/json
Cache-Control: no-store

{
  "transaction_authorization_id": "txn-authz-abc123",
  "authorization_uri": "https://as.example.com/txn-authorization/txn-authz-abc123",
  "expires_in": 300,
  "interval": 5
}
~~~
{: #fig-transaction-authorization-interaction-response title="Transaction authorization interaction response"}

## Pending and Polling

If the authorization server returns a `transaction_authorization_id`, the client continues the transaction authorization
request by polling the transaction authorization endpoint.

The client polls by sending a POST request with the following parameters using the `application/x-www-form-urlencoded`
format with a character encoding of UTF-8 in the HTTP request entity-body:

`client_id`:
: REQUIRED if the client is not authenticating with the authorization server as described in {{Section 3.2.1 of OAUTH-FRAMEWORK}}.
  The client identifier issued to the client during the registration process.

`transaction_authorization_id`:
: REQUIRED. The transaction authorization identifier returned by the authorization server.

In the key-bound profile, each polling request MUST prove possession of the client key, using the same mechanism
as the initial transaction authorization request.

For example, using DPoP:

~~~
POST /txn-authorization HTTP/1.1
Host: as.example.com
Content-Type: application/x-www-form-urlencoded
DPoP: eyJ0eXAiOiJkcG9wK2p3dCIsImFsZyI6IkVTMjU2IiwiandrIjp7Li4ufX0...

client_id=s6BhdRkqt3
&transaction_authorization_id=txn-authz-abc123
~~~
{: #fig-transaction-authorization-poll title="Polling a transaction authorization request"}

The client MUST wait at least the number of seconds specified by the `interval` parameter before polling again.
If no `interval` value was provided, the client MUST wait at least 5 seconds between polling requests.

The authorization server MUST ensure that the client polling the transaction authorization endpoint is the
same client that initiated the transaction authorization request, by verifying the client authentication and, in
the key-bound profile, that the proof of possession on the polling request uses the key bound to the
`transaction_authorization_id`.

If the transaction authorization request is still pending, the authorization server returns an error response with the
`authorization_pending` error code, as defined in {{Section 3.5 of OAUTH-DEVICE}}.

For example:

~~~
HTTP/1.1 400 Bad Request
Content-Type: application/json
Cache-Control: no-store

{
  "error": "authorization_pending"
}
~~~
{: #fig-transaction-authorization-pending title="Transaction authorization pending response"}

The authorization server MAY return an error response with the `slow_down` error code, as defined in {{Section 3.5 of OAUTH-DEVICE}},
to instruct the client to increase the polling interval. After receiving `slow_down`, the client MUST increase the polling interval by at
least 5 seconds.

On encountering a connection timeout, clients MUST unilaterally reduce their polling frequency before retrying. The use of an exponential
backoff algorithm to achieve this, such as doubling the polling interval on each such connection timeout, is RECOMMENDED.

If the transaction authorization request is approved, the authorization server returns an access token response as described in
{{successful-access-token-response}}.

If the approving party denies the request, the authorization server returns an error response with the `access_denied` error code.
If the transaction authorization request has expired, the authorization server returns an error response with the `expired_token`
error code, as defined in {{Section 3.5 of OAUTH-DEVICE}}.

Error responses use the OAuth error response format. The authorization server MAY include additional members, and
application profiles MAY define additional members, for example a structured reason for denial. Clients MUST
ignore members they do not recognize.

## Successful Access Token Response {#successful-access-token-response}

If the transaction authorization request is approved, the authorization server returns an access token response as defined in
{{Section 5.1 of OAUTH-FRAMEWORK}}. This section describes the key-bound profile; in the relay profile the authorization server
instead returns a transaction token using the response format of {{TXN-TOKENS}} (see {{assurance-profiles}}).

The access token MUST be sender-constrained to the client key: the authorization server MUST associate the access token with a
`cnf` confirmation equal to the `cnf` claim of the challenge. When DPoP is used the confirmation is the `jkt` member and the
`token_type` is `DPoP`; when mutual-TLS is used the confirmation is the `x5t#S256` member and the `token_type` is `Bearer`, as
described in {{client-key-pop}}.

The access token MUST be narrowed to the authorized operation. The authorization server MUST include the `authorization_details`
from the challenge, or an equivalent or narrower representation, and MUST associate the access token with the `txn` value
from the challenge. The access token MUST use the `iss` value from the challenge as its audience unless an application
profile defines a different audience binding. The authorization server SHOULD issue access tokens with short expiration times
because they represent authorization for a specific operation.

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

# Transaction Access Token {#transaction-access-token}

An access token issued in response to a challenge represents authorization for the challenged operation by a specific
client. This document profiles the sender-constrained access token of {{DPOP}} or {{MTLS}} for use as evidence of
transaction-specific authorization.

The access token is evidence that the authorization required for the challenged operation has been obtained. A
protected resource MAY treat it as terminal authorization for the operation. In deployments that separate the
authorization decision from its enforcement, a protected resource MAY instead treat the access token as one input
to a subsequent authorization decision, for example by a policy decision point, rather than as the decision
itself. This document does not constrain that choice; the validation requirements of this document apply in
either case.

If the protected resource is expected to enforce obligations associated with the authorization, those obligations
MAY be conveyed within the `authorization_details` of the access token or in a claim defined by an application
profile.

The access token is issued by the authorization server identified by the `aud` claim of the challenge and is
presented to the protected resource that issued the challenge.

The access token MUST contain sufficient information for the protected resource to determine that the token authorizes the challenged
operation. This information can include the `authorization_details` claim, the `txn` value, a reference to protected resource state,
or other information agreed between the protected resource and authorization server. The access token MUST contain a `cnf` claim
equal to the `cnf` claim of the challenge.

## Token Presentation

The client presents the access token to the protected resource together with the request for the challenged operation, proving
possession of the client key as described in {{client-key-pop}}. When DPoP is used, the client presents the access token using the
`DPoP` authentication scheme in the `Authorization` request header field and includes a DPoP proof in the `DPoP` header field. When
mutual-TLS is used, the client presents the access token using the `Bearer` authentication scheme over a mutual-TLS connection
established with the client certificate.

For the challenged operation, this access token is the authorization the protected resource evaluates: the client presents it in
the `Authorization` header in place of the access token it presented on the original request, and does not present that original
access token for the challenged operation. The access token issued in response to a challenge is therefore narrowed by the
`authorization_details` and `txn` of the challenge so that it conveys, on its own, both that the client may perform the
operation and that the operation was authorized. Because the protected resource selected the issuing authorization server in the
challenge `aud` claim and trusts it to authorize the operation (see {{authorization-server-processing}}), this holds even when
that authorization server differs from the issuer of the access token presented on the original request. A protected resource that
requires additional baseline context beyond what the access token carries MUST obtain it through application-specific means and MUST
NOT expect a second token in the request.

For example, using DPoP:

~~~
POST /payments HTTP/1.1
Host: resource.example.com
Authorization: DPoP eyJhbGciOiJFUzI1NiIsInR5cCI6ImF0K2p3dCJ9...
DPoP: eyJ0eXAiOiJkcG9wK2p3dCIsImFsZyI6IkVTMjU2IiwiandrIjp7Li4ufX0...
Content-Type: application/json

{
  "amount": "5000.00",
  "currency": "GBP",
  "recipient": "Example Ltd"
}
~~~
{: #fig-access-token-presentation title="Presenting a transaction access token to a protected resource"}

Because the access token is sender-constrained to the client key, only the client that obtained the token can present it. A client
MUST NOT use the access token for a different transaction, different protected resource, or different operation than the one
described in the challenge.

In deployments where work is delegated across multiple components, the access token cannot be used by a different component, because
the recipient cannot prove possession of the bound key. Delegating the challenged operation to a different component requires that
component to obtain its own access token via its own challenge, or to use the delegation mechanism in {{delegation}}.

## Protected Resource Validation

Before accepting an access token as authorization for a challenged operation, the protected resource MUST validate the
access token according to {{OAUTH-FRAMEWORK}}, the sender-constraining requirements of {{DPOP}} or {{MTLS}} as applicable,
and the requirements of this document.

At a minimum, the protected resource MUST verify that:

* the access token was issued by the authorization server identified by the `aud` claim of the challenge;

* the access token audience identifies the protected resource, unless an application profile defines a different audience binding;

* the access token has not expired;

* the client proves possession of the bound key on the request, and the access token's `cnf` confirmation equals the key
  demonstrated (the JWK Thumbprint of the DPoP proof key, or the SHA-256 thumbprint of the mutual-TLS client certificate), and
  equals the `cnf` value the protected resource recorded for this operation;

* the access token is associated with the same `txn` value as the challenge;

* the access token authorizes the requested operation, for example by matching the `authorization_details`;

* the access token has not previously been used, if the protected resource requires single-use access tokens.

A protected resource MUST reject an access token that does not correspond to the challenge for the requested
operation, or that is presented by a party that cannot prove possession of the bound key.

A protected resource SHOULD treat these access tokens as single-use when the challenged operation is non-idempotent or high impact.
If single-use semantics are required, the protected resource MUST maintain sufficient state to detect replay of the access
token or transaction identifier.

The protected resource MUST NOT accept the access token as general authorization for operations other than the challenged operation.

## Propagation Within the Protected Resource Trust Domain {#propagation}

The access token issued in response to a challenge is sender-constrained to the client key and is audience-restricted to
the protected resource that issued the challenge. It is consumed at that protected resource and is not intended to be
forwarded to other components as a bearer credential.

When a protected resource must propagate the authorized-transaction context to downstream services as it fans the operation out
within its own trust domain, it does so after validating the access token, using a transaction token as defined by
{{TXN-TOKENS}}: the protected resource's trust domain issues a transaction token through its Token Service and propagates it on
the internal call chain. This keeps the external leg sender-constrained and theft-resistant while reusing the transaction-token
mechanism for the internal leg for which it was designed. The transaction token carries the same `txn` value as the challenge so that the internal context remains correlated with the authorized operation.

# Delegation Across Components {#delegation}

In some deployments the component that obtains transaction authorization is not the component that performs the operation, or the
operation is carried out by a chain of components. Because the access token is sender-constrained to a single client key, it
cannot simply be handed to another component: the recipient cannot prove possession of the bound key. Rather than weaken the
binding, this mechanism re-establishes it at each hop using OAuth 2.0 Token Exchange {{OAUTH-TOKEN-EXCHANGE}}.

A downstream component that is to continue the operation presents the access token it received to the authorization server's
token endpoint as the subject token of a token exchange request {{OAUTH-TOKEN-EXCHANGE}}, together with proof of possession of
its own client key (a DPoP proof or a mutual-TLS connection, per {{client-key-pop}}). The authorization server, applying its
delegation policy, issues a new access token that is:

* sender-constrained to the downstream component's key, by setting the new access token's `cnf` confirmation to that key;

* associated with the same `txn` value and an equivalent or narrower `authorization_details`, so the authorization remains scoped
  to the same operation; and

* extended with an `act` claim ({{OAUTH-TOKEN-EXCHANGE}}) that records the delegating component as an actor, preserving the
  delegation chain.

The subject token presented by the downstream component is sender-constrained to the upstream component's key, which the
downstream component cannot demonstrate. The authorization server therefore accepts the subject token on the basis of its
delegation policy -- not proof of possession of the upstream key -- after validating the subject token's issuer, audience,
`txn`, and expiration; the downstream component proves possession only of its own key, to which the newly issued token is bound.

Each hop is therefore independently sender-constrained, and the authorization server retains a verifiable record of the
delegation chain. A component MUST NOT present an access token bound to a key it does not control as evidence of its own
authorization, and the authorization server MUST apply policy to determine whether the requested delegation is permitted before
issuing a re-bound access token. The
protected resource validates the final access token exactly as in {{transaction-access-token}}; the `act` chain conveys the
delegation context for any policy the protected resource applies.

# Security Considerations

Challenges and the access tokens issued in response to them are security-sensitive artifacts. A challenge
requests authorization for a specific operation, and the resulting access token represents evidence that the challenged
operation was authorized for a specific client. Implementations need to ensure that these artifacts cannot be modified,
replayed, substituted, or used for a different operation or by a different client.

## Transaction Authorization Challenge Integrity

A protected resource MUST sign each challenge using an asymmetric signing key. Clients and
authorization servers MUST validate the challenge signature before using any claim from the challenge for display,
policy evaluation, or authorization decisions. If a challenge cannot be validated, it MUST NOT be treated as
authentic.

The authorization server MUST verify that the `aud` claim identifies the authorization server and that the protected
resource identified by the `iss` claim is trusted to request transaction authorization for the requested operation. An
authorization server MUST NOT issue an access token for a challenge issued by an unrecognized or unauthorized
protected resource.

## Client Key Binding {#client-key-binding}

The defining security property of this mechanism is that the entire flow is bound to a single client key. The protected
resource records the confirmation of the client key in the `cnf` claim of the challenge; the authorization
server verifies proof of possession of that key and sender-constrains the access token to it; and the protected resource
verifies proof of possession again when the access token is presented. As a result, an access token obtained for an
operation is useless to any party other than the client that obtained it. This defeats theft, replay, and confused-deputy
attacks in which an approval obtained for one client, user, or delegation context is used to authorize a transaction
initiated by another requester.

This sender-constraint is what distinguishes this mechanism from one that binds only by `client_id`. Client authentication
binds the redemption of the challenge at the authorization server, but the access token is later presented to the
protected resource, a party to which the client does not authenticate as an OAuth client. Without a key binding, the access
token would be a bearer token on that leg and could be replayed by anyone who obtained it, for example over the relaying path
between the client and the protected resource. Proof of possession closes that gap, and unlike `client_id` it also protects
public clients, whose `client_id` is not a secret. In the key-bound profile the `client_id` claim, when present, is
therefore complementary defense-in-depth on the authorization-server leg and not a substitute for the `cnf`
binding. In the relay profile there is no `cnf` binding, so `client_id` and the single-use bearer transaction token
are the only constraints on redemption and replay; this is why the relay profile is restricted to operations for
which that weaker protection is acceptable (see {{assurance-profiles}}).

The `cnf` claim provides verifiable requester context. It is stronger than an asserted requester identifier, because
the authorization server and protected resource verify possession of the corresponding private key rather than relying on
a claimed value.

Implementations MUST ensure that the key recorded in the challenge `cnf` claim is the key actually demonstrated on the
request that triggered the challenge. A protected resource MUST NOT issue a challenge bound to a key whose
possession has not been demonstrated on the triggering request, or otherwise established out of band.

## Possession Is Not Authority

Proof of possession of the client key authenticates the client and binds the request and the resulting access token; it does
not authorize the operation. The description of the operation is signed by the protected resource and MUST NOT be replaced
by a client-supplied description. The decision to approve the operation is made by the authorization server and the approving
party. The approval interaction takes place between the authorization server and the approving party, out of band from the
client. Implementations MUST NOT treat a keyed client as authorized to approve operations on its own behalf, and MUST NOT
move the approval decision to the client.

Authorization servers and protected resources MUST NOT rely on an unprotected description supplied by the client as the basis
for user display, policy evaluation, or authorization decisions. Information presented to the user or approving party SHOULD be
derived from the validated challenge, from protected resource state identified by the challenge, or from information
otherwise authenticated and bound to the challenge.

## Operation Binding and Replay

The challenge and the resulting access token MUST be bound to the same transaction
identifier. The protected resource MUST verify that the `txn` value associated with the access token matches the `txn` value
from the challenge, and that the operation described by the access token matches the requested operation. An access
token MUST NOT be accepted as authorization for any operation other than the challenged operation.

Access tokens can be replayed by the bound client if they are not sufficiently constrained. Authorization servers MUST issue
these access tokens with short lifetimes. Protected resources SHOULD treat them as single-use for
non-idempotent or high-impact operations, and maintain sufficient state to detect replay where single-use semantics
are required.

## Privacy

The client can inspect the challenge before presenting it to the authorization server. This is important because the
challenge can contain sensitive information about the requested operation, user intent, protected resources, or
organizational policy. Clients SHOULD allow the user to decline before disclosing the challenge to the authorization
server when it contains privacy-sensitive transaction details.

Challenges and access tokens can reveal sensitive information if logged or exposed to unintended parties.
Implementations SHOULD minimize the information included in challenges and access tokens, avoid logging them
unless necessary, and protect them in transit and at rest.

Deployments that wish to limit what the authorization server learns about the operation MAY introduce a privacy-preserving
intermediary, in the manner of the privacy service of {{AAUTH}}, between the client and the authorization server. Such an
extension is out of scope for this document and MUST preserve the challenge integrity and client key binding properties
described here.

## Extensions

Application profiles that define additional challenge claims, request binding mechanisms, or alternative audience
bindings MUST describe how those extensions preserve challenge integrity, the client key binding, and prevent replay,
substitution, and confused-deputy attacks.


# IANA Considerations

This document registers the `Accept-Txn-Challenge` HTTP field name, one
OAuth error code, two OAuth parameters, two OAuth Protected Resource
Metadata parameters, one OAuth Authorization Server Metadata Parameter, two JWT claims.

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
: rs-client response, transaction authorization request

Change controller:
: IETF

Reference:
: this document

Name:
: transaction_authorization_id

Parameter Usage Location:
: transaction authorization request, transaction authorization response

Change controller:
: IETF

Reference:
: this document


## OAuth Protected Resource Metadata Registration

IANA is requested to register the following values in the "OAuth Protected Resource Metadata" registry {{IANA.OAuth.Parameters}}.

Metadata name:
: txn_challenge_jwks_uri

Metadata description:
: URL of the protected resource's JSON Web Key Set containing public keys used to validate challenges.

Change controller:
: IETF

Reference:
: this document

Metadata name:
: txn_challenge_signing_alg_values_supported

Metadata description:
: JSON array containing the JWS `alg` values supported by the protected resource for challenges.

Change controller:
: IETF

Reference:
: this document

## OAuth Authorization Server Metadata Registration

IANA is requested to register the following value in the "OAuth Authorization Server Metadata" registry {{IANA.OAuth.Parameters}}.

Metadata name:
: transaction_authorization_endpoint

Metadata description:
: URL of the authorization server endpoint to which clients submit challenges.

Change controller:
: IETF

Reference:
: this document

## JSON Web Token Claims Registration

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

The client-key binding in this mechanism adapts the resource token concept from the AAuth protocol {{AAUTH}}.

TODO acknowledge.
