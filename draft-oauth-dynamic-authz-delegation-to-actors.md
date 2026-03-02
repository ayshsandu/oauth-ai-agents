---
title: "OAuth 2.0 Extension: Dynamic Authorization delegation to Delegated Actors"
abbrev: "OAuth 2.0 Dynamic Delegation to Delegated Actors"
category: info

docname: draft-oauth-dynamic-authz-delegation-to-actors-00
submissiontype: IETF  # also: "independent", "editorial", "IAB", or "IRTF"
number:
date:
consensus: true
v: 3
area: "Security"
workgroup: "Web Authorization Protocol"
keyword:
 - secondary-actor
 - dynamic-delegation
 - authorization
 - actor
 - obo
 - autonomous-entity
venue:
  group: "Web Authorization Protocol"
  type: "Working Group"
  mail: "oauth@ietf.org"
  arch: "https://mailarchive.ietf.org/arch/browse/oauth/"
  github: "shashimalcse/oauth-ai-agents"

author:
 -
    fullname: Thilina Shashimal Senarath
    organization: WSO2
    email: thilinasenarath97@gmail.com
 -  
    fullname: Ayesha Dissanayaka
    organization: WSO2
    email: ayshsandu@gmail.com

normative:
  RFC2119:
  RFC6749:
  RFC7519:
  RFC7636:
  RFC8174:
  RFC8693:
  RFC9068:
  RFC9126:

informative:
  RFC8628:
  OpenID.CIBA:
    title: "OpenID Connect Client-Initiated Backchannel Authentication Flow - Core 1.0"
    target: https://openid.net/specs/openid-client-initiated-backchannel-authentication-core-1_0.html
    date: 2021-09
    author:
      - org: OpenID Foundation

--- abstract

This specification extends the OAuth 2.0 Authorization Framework {{RFC6749}} to enable the dynamic delegation of authority from a human user (the Granting User) to an autonomous entity (the Delegated Actor) through an OAuth 2.0-mediated flow. A Delegated Actor is any software entity -- such as an AI agent, an automated script, a background service, or a robotic process -- that performs actions on behalf of a user at protected resources.

The specification introduces the **requested_actor** parameter at the authorization endpoint to identify the specific Delegated Actor requiring delegation, and the **actor_token** parameter at the token endpoint to cryptographically authenticate the Delegated Actor during the exchange of an authorization code for an access token. The authorization flow supports Dynamic Consent, wherein the flow may be initiated or re-initiated by a resource server challenge, ensuring that Granting User consent is obtained or refreshed in real time when access is attempted. This extension ensures secure delegation with explicit, informed user consent; streamlines the authorization flow for autonomous entities; and enhances auditability through structured access token claims that document the full delegation chain from the Granting User to the Delegated Actor via the client application.

The specification supports three authorization initiation modes -- Pushed Authorization Requests (PAR) {{RFC9126}} for interactive sessions with enhanced request integrity, Client-Initiated Backchannel Authentication (CIBA) {{OpenID.CIBA}} for background or offline Delegated Actor scenarios, and direct front-channel authorization as a baseline for primary deployments -- enabling flexible deployment and upfront Delegated Actor validation before user authorization is solicited.

--- middle

# Introduction

Modern systems increasingly rely on autonomous software entities -- AI agents, automated scripts, background services, robotic process automation (RPA) bots, and other programmatic actors -- that perform tasks on behalf of human users. These entities, referred to in this specification as "Delegated Actors," frequently require access to protected resources governed by OAuth 2.0 to act on-behalf-of human users. The core challenge is the dynamic delegation of authority: enabling a human user (the "Granting User") to securely authorize a Delegated Actor just-in-time to act on their behalf, with fine-grained, auditable, and revocable control.

Standard OAuth 2.0 flows, such as the Authorization Code Grant and the Client Credentials Grant {{RFC6749}}, do not fully address the complexities of delegating authority to an autonomous entity that is distinct from a traditional client application itself. They lack specific mechanisms to (a) obtain explicit, informed user consent targeted at a particular Delegated Actor, (b) authenticate the Delegated Actor as a distinct identity during the token exchange, or (c) produce access tokens that unambiguously document the delegation chain.

The OAuth 2.0 Token Exchange specification {{RFC8693}} provides a framework for exchanging tokens but is primarily designed for server-to-server communication. It does not natively support obtaining explicit user consent for a Delegated Actor through a human-in-the-loop channel (such as the authorization endpoint or CIBA endpoint), nor does it define how to acquire the initial subject token -- adding complexity to the delegation process. This specification builds upon the delegation semantics of the Token Exchange specification {{RFC8693}} while addressing these gaps.


To address these limitations, this specification leverages the OAuth 2.0 Authorization Code Grant {{RFC6749}}, Pushed Authorization Requests (PAR) {{RFC9126}}, and Client-Initiated Backchannel Authentication (CIBA) {{OpenID.CIBA}} flows to enable dynamic, user-consented, and auditable authorization delegation to Delegated Actors. It introduces the following key enhancements:


1. The **requested_actor** parameter at the initiation endpoint, allowing the client to specify the Delegated Actor for which delegation is requested. This parameter may convey intent-based scoping information.

2. The **actor_token** parameter at the token endpoint, enabling the Delegated Actor to cryptographically identify (who the actor is) and authenticate (proving actor is present) itself when exchanging a user-approved authorization code for an access token.

3. Three **authorization initiation modes** providing deployment flexibility:
   * **Pushed Authorization Requests (PAR)** {{RFC9126}} (RECOMMENDED) for standard interactive sessions, providing request integrity and enabling upfront actor validation.
   * **Client-Initiated Backchannel Authentication (CIBA)** {{OpenID.CIBA}} for scenarios where the Delegated Actor operates as a background process and the Granting User is not currently interacting with the Client.
   * **Direct front-channel authorization** as a baseline for low-complexity or legacy clients where PAR is not supported.

4. **Upfront Actor Validation** via PAR and CIBA, allowing the Client to submit the `actor_token` during initiation so the Authorization Server can validate the Delegated Actor's identity before prompting the Granting User for consent.

5. A **Dynamic Consent** mechanism, wherein the authorization flow can be initiated or re-initiated by a resource server challenge, enabling real-time, "human-in-the-loop" consent acquisition when the Delegated Actor encounters an access barrier.

6. Structured claims in the resulting access token (following {{RFC9068}}), capturing the identities of the Granting User, the Delegated Actor, and the client application for transparency, auditability, and downstream policy enforcement.

7. Defined **error codes** for delegation-specific failure modes, including scenarios where consent is pending, denied, or where the Delegated Actor's identity does not match the consented delegation.

This approach builds on existing OAuth 2.0 infrastructure and is designed to be technology-agnostic: while AI agents are a prominent use case, the protocol applies equally to any automated system, script, or sub-process that must act on behalf of a human user.

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "NOT RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in BCP 14 {{RFC2119}} {{RFC8174}} when, and only when, they appear in all capitals, as shown here.

# Terminology

This specification uses the following terms:

Delegated Actor:
: An autonomous software entity that acts on behalf of a Granting User at protected resources. A Delegated Actor is distinct from the client application; it is the downstream entity to which authority is ultimately delegated. Examples include AI agents, automated scripts, robotic process automation (RPA) bots, background data-processing services, and financial rebalancing algorithms. A Delegated Actor MUST possess a unique, stable identifier and MUST be capable of authenticating itself to the Authorization Server via an Actor Token.

Granting User:
: The human resource owner who explicitly authorizes a Delegated Actor to access their protected resources. The Granting User provides informed consent through the Authorization Server's consent interface, and this consent is bound to a specific Delegated Actor, client, and set of permissions. This is the "resource owner" as defined in OAuth 2.0 {{RFC6749}}.

Client:
: An application registered as an OAuth 2.0 client that initiates the authorization flow and facilitates the overall delegation mechanics between the Granting User, the Delegated Actor, and the Authorization Server. The Client acts as an intermediary vessel for the flow: it constructs authorization requests on behalf of the Delegated Actor and relays authorization codes and tokens. Because this specification REQUIRES the Delegated Actor to authenticate itself via the `actor_token` at the token endpoint, the Client MAY be a public client (as defined in Section 2.1 of {{RFC6749}}); the Delegated Actor's authentication provides the cryptographic proof of identity that would otherwise be expected from a confidential client, making the Client primarily a conduit for the delegation flow rather than the sole bearer of security credentials. A Client MAY be a standalone application, a sub-component of the Delegated Actor (e.g., a "Tool" or "Plugin" invoked by an AI Orchestrator), or a platform hosting the Delegated Actor.

Dynamic Consent:
: A consent model in which the Granting User's authorization is obtained or refreshed in real time, at the moment a Delegated Actor encounters an access barrier. Unlike static, upfront consent, Dynamic Consent supports a "human-in-the-loop" pattern: the Delegated Actor's flow is interrupted, consent is solicited from the Granting User via the Authorization Server, and the flow resumes once consent is granted. Dynamic Consent MAY be triggered by a resource server challenge (HTTP 401 or 403), or proactively by the client when a new delegation scope is required.

Intent Scope:
: A scope value or structured scope parameter that conveys the Delegated Actor's intended action or purpose, rather than a static resource permission. Intent Scopes allow the Authorization Server to present the Granting User with a meaningful description of what the Delegated Actor intends to do (e.g., "schedule a meeting on your behalf" or "rebalance your portfolio using conservative strategy"), enabling more informed consent decisions. Intent Scopes are OPTIONAL and complement traditional OAuth 2.0 scope values.

Authorization Server:
: The server that authenticates the Granting User, obtains their consent for delegation to a specific Delegated Actor, and issues access tokens. The Authorization Server MUST be capable of recognizing Delegated Actor identifiers and binding authorization codes to the tuple of (Granting User, Client, Delegated Actor).

Resource Server:
: The server hosting the protected resources, capable of accepting and validating access tokens. A Resource Server MAY challenge a client or Delegated Actor with an HTTP 401 or 403 response to trigger the Dynamic Consent flow. Examples include API servers, data stores, tool-hosting services, and inter-service endpoints.

Authorization Code:
: A temporary, single-use credential issued by the Authorization Server to the Client's redirect URI after the Granting User has authenticated and granted consent for a specific Delegated Actor to act on their behalf. The Authorization Code is bound to the Granting User, Client, and the consented Delegated Actor. Authorization Codes are issued in PAR (Option A) and Direct (Option C) modes only. In CIBA (Option B), the Access Token is obtained through the CIBA token delivery modes (poll, ping, or push) per {{OpenID.CIBA}}.

Actor Token:
: A security token (e.g., a JWT {{RFC7519}}) used by a Delegated Actor to cryptographically authenticate itself to the Authorization Server during the token exchange. The `sub` claim of an Actor Token MUST identify the Delegated Actor. A Delegated Actor obtains an Actor Token through a separate authentication mechanism (e.g., client credentials grant, mutual TLS, or an identity provider flow), which is outside the scope of this specification.

Access Token:
: An access token issued by the Authorization Server, permitting the bearer to access protected resources on behalf of a specific Granting User. When issued via this flow, the Access Token explicitly documents the delegation path through structured claims identifying the Granting User, the Delegated Actor, and the Client.

Pushed Authorization Request (PAR):
: A mechanism defined in {{RFC9126}} that allows the Client to submit authorization request parameters directly to the Authorization Server via a back-channel POST request, receiving a `request_uri` in return. The Client then uses this `request_uri` in the front-channel redirect to the authorization endpoint. In this specification, PAR provides request integrity protection and enables upfront validation of the Delegated Actor before the Granting User is redirected for authorization.

Client-Initiated Backchannel Authentication (CIBA):
: A mechanism defined in {{OpenID.CIBA}} enabling the Client to request user authorization without the Granting User interacting with the Client's user-agent. The Authorization Server contacts the Granting User through an out-of-band channel to obtain consent. See {{OpenID.CIBA}} for the full specification.

Binding Message:
: A short, human-readable string included in a CIBA authentication request that the Authorization Server MUST display to the Granting User on their out-of-band device. The Binding Message MUST accurately reflect the `intent_scope` and the Delegated Actor's identity. See Section 7.1 of {{OpenID.CIBA}} for the base parameter definition.

Step-Up Authorization Challenge:
: A mechanism by which a Resource Server signals to a Client that the authorization associated with the presented access token does not meet the Resource Server's requirements.(e.g., HTTP 401 Unauthorized due to an invalid/missing token, or HTTP 403 Forbidden due to insufficient scope). In this specification, the Step-Up Authorization Challenge is the RECOMMENDED mechanism for Resource Servers to trigger the Dynamic Authorization flow.

# Protocol Overview

This extension defines a flow where a Client application facilitates Granting User consent for a Delegated Actor, and the Delegated Actor then uses this consent along with its own authentication to obtain an access token for accessing protected resources on behalf of the Granting User.

## High-Level Overview

1. The Delegated Actor initiates the flow by signaling to the Client that it needs to perform an action on the Granting User's behalf, providing its identifier (ActorID) and optionally an Intent Scope describing the desired action. The Client acts as a vessel, facilitating the delegation mechanics for the Delegated Actor.

2. The Client attempts the action by making a request to the Resource Server (with an existing access token, if available).

3. If access is unsuccessful (e.g., HTTP 401 Unauthorized due to an invalid/missing token, or HTTP 403 Forbidden due to insufficient scope), the Resource Server challenges the Client. This challenge triggers the Dynamic Consent mechanism.

4. One of the following authorization initiation mode is executed based on the Delegated Actor's operational context and the Granting User's availability: (a) Pushed Authorization Request (PAR) for interactive sessions (RECOMMENDED), (b) Client-Initiated Backchannel Authentication (CIBA) for background/offline scenarios, or (c) direct front-channel authorization as a baseline. The Client then initiates the authorization flow, including the `requested_actor` parameter and optionally Intent Scope values.
    * For PAR, the Client SHOULD include the `actor_token` for upfront validation. The Authorization Server validates the `actor_token` and binds the validated actor identity to the `request_uri` session before redirecting the User-Agent for consent.
    * For CIBA, the Client MUST provide the actor_token in the initial backchannel authentication request to validate and bind the Delegated Actor's identity to the transaction.
    * In CIBA, **ping** or **poll** modes, subsequent requests to the Token Endpoint SHOULD NOT include the actor_token unless the Authorization Server requires a fresh proof-of-presence at the moment of token issuance. The auth_req_id serves as the secure handle for the pending delegation.

5. The Authorization Server validates the request parameters, including:
  * Validating the `requested_actor` to confirm it corresponds to a recognized Delegated Actor identity.
  * If the `actor_token` is provided (via PAR or CIBA), validating the token's signature, expiration, and binding, and verifying that the `sub` claim of the `actor_token` matches the `requested_actor` parameter. If they do not match, the Authorization Server MUST reject the request with the error code `actor_mismatch`. 
  * For PAR, the validated actor identity is bound to the `request_uri` session. For CIBA, the validated actor identity MUST be bound to the complete transaction identified by the `auth_req_id`, ensuring continuity through the out-of-band consent and token polling phases.

6. The Authorization Server then authenticates the Granting User: Facilitates a suitable authentication flow (if not already authenticated) based on the initiation method, and presents a consent screen detailing the Client, the `requested_actor` (Delegated Actor identity), and the requested permissions (including any Intent Scopes). This is the "human-in-the-loop" decision point.

6. Upon Granting User consent, the flow proceeds based on the initiation mode:
  * **PAR and Direct modes:** The Authorization Server issues an Authorization Code bound to the tuple (Granting User, Client, Delegated Actor, granted scopes) and redirects the User-Agent back to the Client's redirect_uri.
  * **CIBA mode:** The Authorization Server records the Granting User's consent (obtained via the out-of-band channel) and prepares to issue an Access Token. 

7. **PAR and Direct modes:** The Client receives the Authorization Code via the User-Agent redirect.

8. The Client requests an Access Token from the Authorization Server's token endpoint:
  * **PAR and Direct modes:** The request uses the standard `authorization_code` grant type and MUST include the Authorization Code, the PKCE `code_verifier`, and the `actor_token` (the cryptographic authentication token of the Delegated Actor).
  * **CIBA mode:** The CIBA token delivery mode can be any of the modes specified in {{OpenID.CIBA}}: **poll**, **ping**, or **push**.

9. The Authorization Server validates the entire request:
  * **PAR and Direct modes:** Validates the Authorization Code, PKCE `code_verifier`, and the `actor_token`. It ensures the `actor_token` is valid, and its subject corresponds to the `requested_actor` for whom the Granting User granted consent (which is linked to the Authorization Code). If any validation fails, the Authorization Server MUST return an appropriate error response (e.g., `invalid_actor_token` or `actor_mismatch`).
  * **CIBA poll and ping modes:** Validates the `auth_req_id` and confirms that the Granting User has completed consent via the out-of-band channel. Because the Delegated Actor's identity was validated and bound to the `auth_req_id` during the initial CIBA authentication request, the `actor_token` SHOULD NOT be required in the token request. If the Authorization Server requires a fresh proof-of-presence at the moment of token issuance, it validates the `actor_token` if present. If the `actor_token` is required but missing or invalid at this stage, the Authorization Server MUST return an error response with the error code `invalid_actor_token`.
  * **CIBA push mode:** The Authorization Server delivers the Access Token directly to the Client's callback endpoint upon Granting User consent. No Client-initiated token request occurs; actor binding relies on the upfront validation from the CIBA authentication request.

10. Upon successful validation, the Authorization Server issues an Access Token to the Client. This token is a JWT (per {{RFC9068}}) containing claims identifying the Granting User (e.g., `sub`), the Client (e.g., `azp` or `client_id`), and the Delegated Actor (e.g., `act` claim).

11. The Client retries the action or performs a new action on the Resource Server using this newly obtained Access Token.

12. If access is successful: The Resource Server validates the Access Token (including the delegation claims like `act`) and processes the request, returning the resource or confirming the action.

The Client may then relay the result to the Delegated Actor.
## Sequence Diagram

~~~ ascii-art
    +-----------+   +--------+    +-----------------+   +---------------------+    +---------------+
    | User-Agent|   | Client |   | Delegated Actor  |  | Authorization Server|   | Resource Server |
    +-----------+   +--------+    +-----------------+   +---------------------+    +---------------+
          |             |              |                   |                       |
          |     (1) Signals need to act on Granting User's behalf                 |                       
          |             |  by passing ActorID (+ optional Intent Scope)            |                       |
          |             |<-------------|                   |
          |             |              |                   |                       |
          |             |              (2) Client attempts action                  |
          |             | -------------------------------------------------------> |
          |             |              |                   |                       |
    /--------------------------------- Access Unsuccessful ----------------------------------\
    |     |             |              |                   |                       |         |
    |------------------------------------[If Unauthorized]-----------------------------------|
    |     |             |              |                   |                       |         |
    |     |             |              |             +---------------------------------+     |              
    |     |             |              |             | Token validation is failed      |     |              
    |     |             |              |             +---------------------------------+     |             
    |     |             |              |                   |                       |         |
    |     |             |  (3) CHALLENGE: HTTP 401, WWW-Authenticate:              |         |
    |     |             |           Bearer error="invalid_token"                   |         |
    |     |             | <--------------------------------------------------------|         |
    |     |             |              |                   |                       |         |
    |  (4) Redirect to AS (for User Authentication and Consent)                    |         |
    |              with requested_actor request parameter                          |         |
    |     |<------------|              |                   |                       |         |
    |     |             |              |                   |                       |         |
    |     |         (5) Authorization Request              |                       |         |
    |     |----------------------------------------------->|                       |         |
    |     |             |              |                   |                       |         |
    |     |     (6) User Authenticates & Consents          |                       |         |
    |     |<---------------------------------------------->|                       |         |
    |     |             |              |                   |                       |         |
    |------------------------------------[If Forbidden]--------------------------------------|
    |     |             |              |                   |                       |         |
    |     |             |              |            +---------------------------------+      |             
    |     |             |              |            |   Insufficient Authorization    |      |             
    |     |             |              |            +---------------------------------+      |           
    |     |             |              |                   |                       |         |
    |     |               (7) CHALLENGE: HTTP 403, WWW-Authenticate:               |         |
    |     |       Bearer error="insufficient_scope" required_scope="scope1 scope2" |         |
    |     |             | <--------------------------------------------------------|         |
    |     |             |              |                   |                       |         |
    |  (8) Redirect to AS (for User Consent)               |                       |         |
    |    with requested_actor request parameter            |                       |         |
    |     |<------------|              |                   |                       |         |
    |     |             |              |                   |                       |         |
    |     |         (9) Authorization Request              |                       |         |
    |     |----------------------------------------------->|                       |         |
    |     |             |              |                   |                       |         |
    |     |            (10) User Consents                  |                       |         |
    |     |<---------------------------------------------->|                       |         |
    |----------------------------------------------------------------------------------------|
    |     |             |              |                   |                       |         |
    |     | (11) Redirect with Authorization Code          |                       |         |
    |     | <----------------------------------------------|                       |         |
    |     |             |              |                   |                       |         |
    |     | (12) Authorization Code    |                   |                       |         |
    |     |------------>|              |                   |                       |         |
    |     |             |              |                   |                       |         |
    |     |             | (13) Token Request with actor_token request parameter    |         |
    |     |             | ---------------------------------|                       |         |
    |     |             |              |                   |                       |         |
    |     |             |  (14) Access Token (JWT)         |                       |         |
    |     |             | <--------------------------------|                       |         |
    |     |             |              |                   |                       |         |
    |     |             |  (15) Client Retries Action with Access Token            |         |
    |     |             |--------------------------------------------------------->|         |
    \----------------------------------------------------------------------------------------/
    /------------------------------------ Access Successful ---------------------------------\
    |     |             |              |                   |                       |         |
    |     |             |  (16) Protected Resource / Action Succeeded              |         |
    |     |             |<---------------------------------------------------------|         |
    \----------------------------------------------------------------------------------------/
~~~


The steps in the sequence diagram are as follows:

1. Delegated Actor signals its intent to the Client, providing its identifier (ActorID) and optionally an Intent Scope.

2. Client attempts to access protected resources on the Resource Server.

3. If access is unsuccessful with token validation, the Resource Server challenges the Client with a WWW-Authenticate header indicating the invalid token error. This triggers the Dynamic Consent flow.

4. Client redirects the Granting User's User-Agent to the Authorization Server's authorization endpoint, including `requested_actor` and PKCE challenge.

5. User-Agent makes the Authorization Request to the Authorization Server.

6. Granting User authenticates and grants consent (human-in-the-loop decision point).

7. If access is unsuccessful with insufficient scopes, the Resource Server challenges the Client with a WWW-Authenticate header indicating the insufficient scope error. This also triggers the Dynamic Consent flow.

8. Client redirects the Granting User's User-Agent to the Authorization Server's authorization endpoint, including `requested_actor` and PKCE challenge.

9. User-Agent makes the Authorization Request to the Authorization Server.

10. Granting User grants consent.

11. Authorization Server redirects the User-Agent back to the Client's redirect_uri with an Authorization Code bound to the (Granting User, Client, Delegated Actor) tuple.

12. Client receives the Authorization Code via the User-Agent redirect.

13. Client requests an Access Token from the Authorization Server's token endpoint, including the Authorization Code, PKCE `code_verifier`, and `actor_token`.

14. Authorization Server validates the request (including `actor_token` binding) and issues an Access Token (JWT) to the Client.

15. Client retries the action on the Resource Server using the newly obtained Access Token.

16. Resource Server validates the Access Token (including delegation claims) and processes the request, returning the protected resource or confirming the action.

# Detailed Protocol Steps

## Authorization Initiation

This specification defines three options for how the Client initiates the authorization flow. The appropriate mode depends on the Delegated Actor's operational context, the required security level, and whether the Granting User is actively interacting with the Client.

### Initiation Mode Selection

The Client MUST choose one of the following initiation modes:

Option A -- Pushed Authorization Request (PAR) {{RFC9126}} (RECOMMENDED):
: Use this for standard interactive sessions where the Granting User is actively present. PAR transmits all authorization parameters (including the `requested_actor` and the `actor_token`) server-to-server over TLS, preventing parameter tampering in the user-agent and enabling upfront actor validation.

Option B -- Client-Initiated Backchannel Authentication (CIBA) {{OpenID.CIBA}}:
: Use this when the Delegated Actor is running as a background process and the Granting User is NOT currently interacting with the Client (e.g., scheduled batch jobs, background monitoring agents, after-hours automation). The Authorization Server contacts the Granting User through an out-of-band channel (push notification, SMS, authentication app) to obtain consent.

Option C -- Direct Authorization Request:
: Retained as a baseline for low-complexity or legacy clients/ authorization servers where PAR is not supported. The authorization parameters are transmitted through the user-agent via query parameters. This option does NOT support upfront `actor_token` validation; the `actor_token` is only presented at the token endpoint.

### Upfront Actor Validation (PAR and CIBA)

To ensure the integrity of the delegation process before involving the Granting User, the Client SHOULD include the `actor_token` and `actor_token_type` parameters in the PAR or CIBA initiation request. If provided, the Authorization Server MUST validate the `actor_token` before prompting the Granting User for consent. This upfront validation:

* Confirms the Delegated Actor's identity and token validity before the Granting User is asked to make a consent decision.
* Prevents wasted user interactions if the Delegated Actor's credentials are invalid, expired, or revoked.
* Binds the validated actor identity to the authorization session (the PAR `request_uri` or the CIBA `auth_req_id`), enabling the Authorization Server to verify continuity at the token endpoint.

If the `actor_token` fails upfront validation, the Authorization Server MUST reject the request with the error code `invalid_actor_token` and MUST NOT continue to authorization flow, leaving the Granting User uninvolved.

### Option A: Pushed Authorization Request (PAR) (RECOMMENDED)

For standard interactive sessions where the Granting User is actively interacting with the Client, the Client SHOULD use Pushed Authorization Requests {{RFC9126}} to submit the authorization parameters directly to the Authorization Server's PAR endpoint over a back-channel connection.

#### PAR Request

~~~
POST /par HTTP/1.1
Host: authorization-server.example.com
Content-Type: application/x-www-form-urlencoded
Authorization: Basic <client_credentials>

response_type=code&
client_id=<client_id>&
redirect_uri=<redirect_uri>&
scope=<scope>&
intent_scope=<intent_scope>&
code_challenge=<code_challenge>&
code_challenge_method=S256&
requested_actor=<actor_id>&
actor_token=<actor_token>&
actor_token_type=urn:ietf:params:oauth:token-type:jwt
~~~

#### PAR Request Parameters

All parameters from the Direct Authorization Request (Option C) are valid in the PAR request, with the following additions:

actor_token:
: RECOMMENDED. The security token used to authenticate the Delegated Actor, submitted for upfront validation. This token MUST be a valid token (e.g., a JWT {{RFC7519}}) issued to the Delegated Actor and MUST include the `sub` claim identifying the Delegated Actor. If provided, the Authorization Server MUST validate this token and bind the verified actor identity to the resulting `request_uri`.

actor_token_type:
: REQUIRED if `actor_token` is present. An identifier for the type of the `actor_token`, per Section 3 of {{RFC8693}}. For JWT-based actor tokens, the value MUST be `urn:ietf:params:oauth:token-type:jwt`.

#### Authorization Server Processing (PAR)

Upon receiving the PAR request, the Authorization Server MUST perform the following steps:

1. Authenticate the Client if applicable.

2. Validate the request parameters according to {{RFC9126}} and the parameter rules defined in Option C.

3. Validate the `requested_actor`. The Authorization Server MUST verify that the provided `requested_actor` corresponds to a recognized Delegated Actor identity. If the `requested_actor` is unknown, the Authorization Server MUST return an error response with the error code `invalid_actor`.

4. If `actor_token` is present:
   a. Validate the token's signature, issuer, audience (if applicable), and expiration. The token MUST NOT be expired or revoked.
   b. Extract the Delegated Actor identity from the `actor_token`'s `sub` claim.
   c. Verify that the `sub` claim matches the `requested_actor` parameter. If they do not match, the Authorization Server MUST return an error with the error code `actor_mismatch`.
   d. Bind the validated actor identity to the `request_uri` session.

5. If all validations pass, return the `request_uri`.

#### PAR Response

If the request is valid, the Authorization Server returns a `request_uri`:

~~~
HTTP/1.1 201 Created
Content-Type: application/json
Cache-Control: no-store

{
  "request_uri": "urn:ietf:params:oauth:request_uri:6esc_11ACC5bwc014ltc14eY22c",
  "expires_in": 60
}
~~~

If validation fails, the Authorization Server returns an error response per {{RFC9126}} Section 2.3, using the error codes defined in the Error Response section below.

#### Front-Channel Redirect (PAR)

The Client then redirects the Granting User's user-agent using the `request_uri`:

~~~
GET /authorize?client_id=<client_id>&
request_uri=urn:ietf:params:oauth:request_uri:6esc_11ACC5bwc014ltc14eY22c HTTP/1.1
Host: authorization-server.example.com
~~~

The Authorization Server resolves the `request_uri`, presents the consent screen to the Granting User, and upon consent, issues an Authorization Code as defined in the Authorization Code Response section below.

### Option B: Client-Initiated Backchannel Authentication (CIBA)

When the Delegated Actor is operating as a background process and the Granting User is NOT currently interacting with the Client, the Client SHOULD use CIBA {{OpenID.CIBA}} to obtain authorization through an out-of-band channel (e.g., push notification, SMS). See Section 4 of {{OpenID.CIBA}} for the base authentication request specification.

#### CIBA Authentication Request

~~~
POST /bc-authorize HTTP/1.1
Host: authorization-server.example.com
Content-Type: application/x-www-form-urlencoded
Authorization: Basic <client_credentials>

scope=<scope>&
intent_scope=<intent_scope>&
client_id=<client_id>&
login_hint=<granting_user_identifier>&
binding_message=<human_readable_intent_description>&
requested_actor=<actor_id>&
actor_token=<actor_token>&
actor_token_type=urn:ietf:params:oauth:token-type:jwt
~~~

#### CIBA-Specific Parameters

In addition to the parameters introduced by this specification (`scope`, `intent_scope`, `client_id`, `requested_actor`, `actor_token`, `actor_token_type`), the CIBA request includes the standard CIBA parameters defined in Section 7.1 of {{OpenID.CIBA}}. Of particular note:

login_hint:
: REQUIRED. Identifies the Granting User per Section 7.1 of {{OpenID.CIBA}}.

binding_message:
: REQUIRED for this specification. A human-readable string displayed to the Granting User on their authentication device. This message MUST accurately reflect the `intent_scope` and the Delegated Actor's identity (e.g., `"Allow rebalancer-v1 to execute trades on your portfolio"`).

See Section 7.1 of {{OpenID.CIBA}} for additional standard parameters (`id_token_hint`, `login_hint_token`, `requested_expiry`, `client_notification_token`, etc.).

#### Binding Message Integrity

The `binding_message` MUST be semantically consistent with the `intent_scope` and `scope` parameters. If a mismatch is detected, the Authorization Server MUST reject the request with `invalid_binding_message`. See the Binding Message Integrity subsection under Security Considerations for detailed requirements.

#### Authorization Server Processing (CIBA)

Upon receiving the CIBA authentication request, the Authorization Server MUST:

1. Authenticate the Client and validate standard CIBA parameters per Section 7.3 of {{OpenID.CIBA}}.
2. Validate the `requested_actor` and, the `actor_token` per the upfront validation rules defined above.
3. Validate `binding_message` integrity (see above). If a mismatch is detected, reject with `invalid_binding_message`.
4. Resolve the `login_hint` to identify the Granting User. If the Granting User cannot be identified, reject with `invalid_request`.
5. Initiate the out-of-band consent flow, displaying the `binding_message`, Delegated Actor identity, and requested permissions.

#### CIBA Authentication Response

If valid, the Authorization Server returns an `auth_req_id` per Section 7.3 of {{OpenID.CIBA}}. The `auth_req_id` binds the validated Delegated Actor identity to the pending transaction.

#### CIBA Token Delivery Modes

CIBA defines three token delivery modes -- **poll**, **ping**, and **push** -- as specified in Sections 5, 6, and 7 of {{OpenID.CIBA}} respectively. In all three modes, the `auth_req_id` serves as the secure handle for the pending delegation. Because the Delegated Actor's identity was already validated and bound to the `auth_req_id` during the initial CIBA authentication request, subsequent requests to the Token Endpoint SHOULD NOT include the `actor_token` unless the Authorization Server requires a fresh proof-of-presence at the moment of token issuance. If fresh proof-of-presence is required, the Authorization Server MUST advertise this via its discovery metadata (e.g., `ciba_actor_token_at_token_endpoint_required: true`).

##### Poll Mode

The Client polls the token endpoint per Section 10 of {{OpenID.CIBA}}:

~~~
POST /token HTTP/1.1
Host: authorization-server.example.com
Content-Type: application/x-www-form-urlencoded
Authorization: Basic <client_credentials>

grant_type=urn:openid:params:grant-type:ciba&
auth_req_id=1c266114-a1be-4252-8ad1-04986c5b9ac1&
client_id=<client_id>
~~~

The Authorization Server returns `authorization_pending` while the Granting User has not responded. The Client MUST respect the `interval` value and MUST NOT poll more frequently. Upon consent, the Authorization Server returns an Access Token with the delegation claims defined in the Access Token Structure and Claims section. If consent is denied, the Authorization Server returns `error=consent_denied`.

##### Ping Mode

In ping mode (Section 11 of {{OpenID.CIBA}}), the Authorization Server notifies the Client's pre-registered callback endpoint with the `auth_req_id` when the Granting User responds. The Client then retrieves the token from the token endpoint using the same request format as poll mode. The Client MUST validate the `client_notification_token` on every callback invocation.

##### Push Mode

In push mode (Section 12 of {{OpenID.CIBA}}), the Authorization Server delivers the Access Token directly to the Client's callback endpoint upon Granting User consent. No Client-initiated token request occurs. The pushed Access Token MUST contain the same delegation claims (`sub`, `azp`, `act`) as tokens issued through poll or ping modes. Because there is no Client-initiated token request in push mode, there is no opportunity to present an `actor_token` at issuance time; the actor binding relies entirely on the upfront validation during the CIBA authentication request.

#### CIBA Error Codes

In addition to standard CIBA errors, the following error codes apply to CIBA requests in this specification:

consent_denied:
: The Granting User explicitly denied consent via the out-of-band channel.

invalid_binding_message:
: The `binding_message` does not accurately reflect the `intent_scope` or requested permissions.

invalid_actor_token:
: The `actor_token` is invalid, expired, or does not match the `requested_actor`.

expired_token:
: The `auth_req_id` has expired before the Granting User responded.

### Option C: Direct Authorization Request (Baseline)

For low-complexity or legacy Clients where PAR is not supported, the Client MAY use a direct front-channel authorization request. This is the standard OAuth 2.0 Authorization Code Grant extended with the `requested_actor` parameter.

Note: This option does NOT support upfront `actor_token` validation. The authorization parameters are transmitted through the user-agent, and the `actor_token` MUST NOT be included in the front-channel request (to avoid exposing it in the user-agent). The `actor_token` is only presented at the token endpoint.

#### Direct Authorization Request

~~~
GET /authorize?response_type=code&
client_id=<client_id>&
redirect_uri=<redirect_uri>&
scope=<scope>&
intent_scope=<intent_scope>&
state=<state>&
code_challenge=<code_challenge>&
code_challenge_method=S256&
requested_actor=<actor_id> HTTP/1.1
Host: authorization-server.example.com
~~~

#### Parameters

response_type:
: REQUIRED. Value MUST be set to `code`, per OAuth 2.0 (Section 4.1.1 of {{RFC6749}}).

client_id:
: REQUIRED. The identifier of the Client as registered with the Authorization Server.

redirect_uri:
: REQUIRED. The Client's redirection endpoint as registered with the Authorization Server.

scope:
: RECOMMENDED. A space-delimited list of OAuth 2.0 scope values representing the permissions the Delegated Actor requires (e.g., `read:email write:calendar`).

intent_scope:
: OPTIONAL. A structured scope parameter or URI describing the Delegated Actor's intended action or purpose in human-readable or machine-parseable form (e.g., `intent:schedule_meeting` or `urn:example:rebalance_portfolio:conservative`). The Authorization Server SHOULD use this value to enhance the consent screen with a meaningful description of what the Delegated Actor intends to do. If the Authorization Server does not recognize the `intent_scope`, it MUST ignore it and fall back to the standard `scope` parameter.

state:
: RECOMMENDED. An opaque value used by the Client to maintain state between the request and callback, per Section 4.1.1 of {{RFC6749}}.

code_challenge:
: REQUIRED. The PKCE code challenge, per {{RFC7636}}.

code_challenge_method:
: REQUIRED. Value MUST be set to `S256`, per {{RFC7636}}.

requested_actor:
: REQUIRED. The unique identifier of the Delegated Actor for which the Client is requesting delegated access on behalf of the Granting User. This identifier MUST uniquely identify the Delegated Actor within the Authorization Server's domain and MUST be understood by the Authorization Server.

#### Authorization Server Processing (Direct)

Upon receiving the authorization request, the Authorization Server MUST perform the following steps:

1. Validate the request parameters according to the OAuth 2.0 Authorization Code Grant (Section 4.1.1 of {{RFC6749}}).

2. Validate the `requested_actor`. The Authorization Server MUST verify that the provided `requested_actor` corresponds to a recognized Delegated Actor identity. If the `requested_actor` is unknown, the Authorization Server MUST return an error response with the error code `invalid_actor`.

3. If an `intent_scope` parameter is present, the Authorization Server SHOULD resolve it to a human-readable description of the intended action for display on the consent screen.

4. The Authorization Server MUST present a consent screen to the Granting User (the "human-in-the-loop" decision point). This screen MUST clearly indicate:
   * The name or identity of the Client application initiating the request.
   * The identity and description of the Delegated Actor (`requested_actor`) for which delegation is being requested.
   * The specific scopes of access being requested, and if available, the intent description derived from `intent_scope`.
   * A clear indication that the Granting User is authorizing an autonomous entity to act on their behalf.

If the request is valid and the Granting User grants consent, the Authorization Server proceeds to issue an Authorization Code. If the Granting User denies consent, the Authorization Server MUST return an error response with the error code `consent_denied`. If the request is invalid, the Authorization Server returns an appropriate Error Response.

### Authorization Code Response (PAR and Direct)

Regardless of whether the authorization was initiated via PAR or Direct mode, if the Granting User grants consent, the Authorization Server issues an Authorization Code and redirects the user-agent back to the Client's `redirect_uri`.

Note: For CIBA mode, there is no Authorization Code; the Access Token is obtained directly through the CIBA token polling mechanism described in Option B.

~~~
HTTP/1.1 302 Found
Location: <redirect_uri>?code=<authorization_code>&state=<state>
~~~

#### Parameters

code:
: REQUIRED. The Authorization Code issued by the Authorization Server. This code is bound to the tuple (Granting User, Client, Delegated Actor, granted scopes). If the authorization was initiated via PAR with an upfront-validated `actor_token`, the validated actor identity is also bound to this code.

state:
: REQUIRED if the `state` parameter was present in the authorization request. The exact value received from the Client.

### Error Response (Authorization Endpoint)

If the request fails or the Granting User denies consent, the Authorization Server redirects the user-agent back to the Client's `redirect_uri` with error parameters (for PAR and Direct modes). For CIBA mode, errors are returned as JSON responses at the backchannel endpoint or during token polling.

~~~
HTTP/1.1 302 Found
Location: <redirect_uri>?error=<error_code>&error_description=<description>&state=<state>
~~~

The following error codes are defined for the authorization and PAR endpoints in addition to those in Section 4.1.2.1 of {{RFC6749}}:

consent_denied:
: The Granting User explicitly denied consent for the Delegated Actor delegation.

invalid_actor:
: The `requested_actor` parameter contains an identifier that the Authorization Server does not recognize or that is not eligible for delegation.

invalid_actor_token:
: (PAR and CIBA only) The `actor_token` provided for upfront validation is invalid, expired, revoked, or its signature cannot be verified.

actor_mismatch:
: (PAR and CIBA only) The identity in the `actor_token` (its `sub` claim) does not match the `requested_actor` parameter.

invalid_binding_message:
: (CIBA only) The `binding_message` does not accurately reflect the `intent_scope` or requested permissions.

## Access Token Request (PAR and Direct Modes)

For authorization flows initiated via PAR (Option A) or Direct (Option C) modes, the Client exchanges the Authorization Code for an Access Token at the Authorization Server's token endpoint using the `authorization_code` grant type. The Client MUST include the `actor_token` parameter to authenticate the Delegated Actor at the point of token issuance.

Note: For CIBA (Option B), the Access Token is obtained through the CIBA token delivery modes (poll, ping, or push) described in the Option B section. Because the Delegated Actor's identity is validated and bound to the `auth_req_id` during the initial CIBA authentication request, subsequent token endpoint requests in CIBA poll and ping modes SHOULD NOT include the `actor_token` unless the Authorization Server requires a fresh proof-of-presence at the moment of token issuance. In CIBA push mode, the Access Token is delivered directly to the Client's callback endpoint and no separate token request is made.

~~~
POST /token HTTP/1.1
Host: authorization-server.example.com
Content-Type: application/x-www-form-urlencoded
Authorization: Basic <client_credentials>

grant_type=authorization_code&
client_id=<client_id>&
code=<authorization_code>&
code_verifier=<code_verifier>&
redirect_uri=<redirect_uri>&
actor_token=<actor_token>&
actor_token_type=urn:ietf:params:oauth:token-type:jwt
~~~

### Parameters

grant_type:
: REQUIRED. Value MUST be set to `authorization_code`.

client_id:
: REQUIRED. The identifier of the Client, per Section 4.1.3 of {{RFC6749}}. Confidential clients MUST also authenticate using their registered authentication method (e.g., `client_secret_basic`, `client_secret_post`, or `private_key_jwt`).

code:
: REQUIRED. The Authorization Code received from the Authorization Server.

code_verifier:
: REQUIRED. The PKCE code verifier corresponding to the `code_challenge` sent in the authorization request, per {{RFC7636}}.

redirect_uri:
: REQUIRED. The same `redirect_uri` value that was included in the authorization request.

actor_token:
: REQUIRED. The security token used to authenticate the Delegated Actor at the moment of token issuance. This token MUST be a valid token (e.g., a JWT {{RFC7519}}) issued to the Delegated Actor and MUST include the `sub` claim identifying the Delegated Actor. The Client MUST send the `actor_token` regardless of whether it was previously submitted during PAR initiation; this ensures the Delegated Actor is still present and authorized at the time of issuance.

actor_token_type:
: REQUIRED. An identifier for the type of the `actor_token`, per Section 3 of {{RFC8693}}. For JWT-based actor tokens, the value MUST be `urn:ietf:params:oauth:token-type:jwt`.

### Authorization Server Processing

Upon receiving the token request, the Authorization Server MUST perform the following steps:

1. Validate the request parameters according to the OAuth 2.0 Token Endpoint (Section 4.1.3 of {{RFC6749}}), including client authentication.

2. Validate the `actor_token`: the Authorization Server MUST verify the token's signature, issuer, audience (if applicable), and expiration. The token MUST NOT be expired or revoked.

3. Extract the Delegated Actor identity from the `actor_token`'s `sub` claim.

4. Verify that the authenticated Delegated Actor identity matches the `requested_actor` value that the Granting User consented to during the initial Authorization Request, which is bound to the Authorization Code.

5. **Actor Continuity Check (PAR-initiated flows):** If the authorization was initiated via PAR and an `actor_token` was validated and bound during the PAR request, the Authorization Server MUST verify that the `actor_token` presented at the token endpoint identifies the same Delegated Actor that was bound to the PAR session (`request_uri`). This ensures continuity between the upfront validation and the token issuance, preventing a different (potentially malicious) Delegated Actor from substituting itself between the PAR request and the code exchange.

6. Validate the PKCE `code_verifier` against the `code_challenge` associated with the Authorization Code.

If all validations pass, the Authorization Server issues an Access Token. If any validation fails, the Authorization Server MUST return an appropriate Error Response.

### Access Token Response

If the Token Request is valid, the Authorization Server issues an Access Token. This token SHOULD be a JSON Web Token (JWT) {{RFC7519}} conforming to {{RFC9068}}, to include claims that document the delegation chain.

~~~
HTTP/1.1 200 OK
Content-Type: application/json;charset=UTF-8
Cache-Control: no-store
Pragma: no-cache

{
  "access_token": "<delegated_access_token>",
  "token_type": "Bearer",
  "expires_in": 3600,
  "scope": "<granted_scope>"
}
~~~

#### Parameters

access_token:
: REQUIRED. The access token issued by the Authorization Server.

token_type:
: REQUIRED. The type of the token issued, typically `Bearer`.

expires_in:
: RECOMMENDED. The lifetime in seconds of the access token.

scope:
: OPTIONAL if identical to the scope requested; otherwise REQUIRED. The scopes granted by the Authorization Server.

### Error Response

If the request is invalid, the Authorization Server returns an error response with an HTTP 400 (Bad Request) status code.

~~~
HTTP/1.1 400 Bad Request
Content-Type: application/json;charset=UTF-8
Cache-Control: no-store
Pragma: no-cache

{
  "error": "<error_code>",
  "error_description": "<human_readable_description>"
}
~~~

The following error codes are defined for the token endpoint in addition to those in Section 5.2 of {{RFC6749}}:

actor_mismatch:
: The identity in the `actor_token` (its `sub` claim) does not match the `requested_actor` value bound to the Authorization Code.

invalid_actor_token:
: The `actor_token` is invalid, expired, revoked, or its signature cannot be verified.

authorization_pending:
: The Authorization Code is valid, but the Granting User has not yet completed the consent interaction. The Client SHOULD retry after a reasonable delay. This code is analogous to the `authorization_pending` error in the Device Authorization Grant (RFC 8628).

## Access Token Structure and Claims

The Access Token SHOULD be a JWT Profile for OAuth 2.0 Access Tokens {{RFC9068}}. It MUST carry claims that explicitly document the delegation chain from the Granting User, through the Client, to the Delegated Actor.

In addition to standard JWT claims (e.g., `iss`, `aud`, `exp`, `iat`, `jti`), an Access Token issued via this flow MUST contain the following claims:

sub:
: REQUIRED. The unique identifier of the Granting User (the resource owner who consented to the delegation).

azp:
: REQUIRED. The `client_id` of the Client application that facilitated the delegation flow.

scope:
: REQUIRED. The granted scope values, representing the permissions the Delegated Actor is authorized to exercise.

act:
: REQUIRED. The actor claim, representing the Delegated Actor that is acting on behalf of the Granting User (Section 4.1 of {{RFC8693}}). This claim MUST contain a JSON object with at least the following member:
  * sub: REQUIRED. The unique identifier of the Delegated Actor.
  * Additional members (e.g., a human-readable name or type) MAY be included in the `act` claim.

Example Decoded JWT Payload:

~~~json
{
  "iss": "https://authorization-server.example.com/oauth2/token",
  "aud": "https://resource-server.example.com",
  "sub": "user-456",
  "azp": "s6BhdRkqt3",
  "scope": "read:email write:calendar",
  "exp": 1746009896,
  "iat": 1746006296,
  "jti": "unique-token-id-7f3a",
  "act": {
    "sub": "secondary-actor-assistant-v2"
  }
}
~~~

Resource Servers consuming this token can inspect the `sub` claim to identify the Granting User and the `act.sub` claim to identify the specific Delegated Actor that is performing the action. This provides a clear, auditable delegation path and enables Resource Servers to apply actor-specific access policies.

## Resource Server Challenge and Dynamic Consent Trigger

The authorization flow is typically triggered by a Resource Server challenge, such as an HTTP 401 (Unauthorized) or 403 (Forbidden) response, indicating a missing, invalid, or insufficient access token. This challenge is the primary trigger for the Dynamic Consent mechanism: upon receiving it, the Client interrupts the Delegated Actor's ongoing operation and redirects the Granting User's user-agent to the Authorization Server to obtain real-time consent.

### The Hand-Off Process (Human-in-the-Loop)

The "human-in-the-loop" hand-off works as follows:

1. **Interruption:** The Delegated Actor requests the Client to perform an action at the Resource Server. The Resource Server responds with a challenge (401 or 403).

2. **User-Agent Activation:** The Client activates the Granting User's user-agent (e.g., opens a browser window, sends a push notification with a deep link, or surfaces an in-app authorization prompt) and redirects it to the Authorization Server's authorization endpoint with the `requested_actor` and scope parameters.

3. **Consent Decision:** The Authorization Server presents the consent screen. The Granting User reviews the requested delegation and either grants or denies consent.

4. **Flow Resumption:** Upon granting consent, the Authorization Server redirects the user-agent back to the Client with an Authorization Code. The Client completes the token exchange (including `actor_token` validation) and retries the action at the Resource Server. The Delegated Actor's operation resumes transparently.

5. **Denial Handling:** If the Granting User denies consent, the Authorization Server redirects back with `error=consent_denied`. The Client MUST communicate the denial to the Delegated Actor so it can gracefully handle the refusal (e.g., abandon the action or attempt a fallback).

### Resource Server Request

When the Client, prompted by a Delegated Actor, requests a protected resource, it includes an access token in the Authorization header if available.

~~~
GET /some/protected/resource HTTP/1.1
Host: resource-server.example.com
Authorization: Bearer <existing_access_token_if_any>
~~~

### Resource Server Processing

Upon receiving the request, the Resource Server MUST validate the Access Token (if provided). This validation includes:

1. Ensuring the token is present if required for the resource.

2. Verifying the token's signature, issuer (`iss`), and audience (`aud`).

3. Checking if the token is expired or revoked.

4. Confirming the token contains the necessary scopes for the requested action.

5. If the resource requires action on behalf of a specific Delegated Actor, verifying the token contains the appropriate delegation claims (e.g., an `act` claim) and that the `act.sub` value identifies an authorized Delegated Actor.

If the Access Token is missing, invalid, or insufficient for the requested action, the Resource Server MUST return an error response, typically an HTTP 401 (Unauthorized) or HTTP 403 (Forbidden), including a `WWW-Authenticate` header.

### The WWW-Authenticate Response Header Field

The Resource Server MUST include a `WWW-Authenticate` header field in the response to indicate the reason for the challenge. This header field MUST include the error code and other relevant information to enable the Client to determine the appropriate remediation action.

Example (insufficient scope):

~~~
HTTP/1.1 403 Forbidden
WWW-Authenticate: Bearer error="insufficient_scope",
  error_description="The access token does not have the required scope(s)",
  required_scope="scope1 scope2"
Content-Type: application/json;charset=UTF-8

{
  "error": "insufficient_scope",
  "error_description": "The access token does not have the required scope(s)",
  "required_scope": "scope1 scope2"
}
~~~

Example (missing or invalid delegation):

~~~
HTTP/1.1 401 Unauthorized
WWW-Authenticate: Bearer error="invalid_token",
  error_description="Access token missing required delegation claims"
Content-Type: application/json;charset=UTF-8

{
  "error": "invalid_token",
  "error_description": "Access token missing required delegation claims (act)"
}
~~~

Upon receiving such a challenge, the Client SHOULD initiate the Dynamic Consent flow by redirecting the Granting User's user-agent to the Authorization Server.

# Security Considerations

## Delegated Actor Authentication

The security of this flow relies heavily on the Authorization Server's ability to securely authenticate the Delegated Actor during the Token Request using the Actor Token. The method by which Delegated Actors obtain and secure their Actor Tokens is critical and outside the scope of this specification, but MUST be implemented securely. Authorization Servers SHOULD require strong authentication methods for Delegated Actors, such as asymmetric key-based JWT assertion (e.g., `private_key_jwt`) rather than shared secrets.

## Pushed Authorization Request (PAR) Security Benefits

When PAR {{RFC9126}} is used as the initiation mode, the authorization parameters (including `requested_actor`, `actor_token`, `scope`, and `intent_scope`) are transmitted server-to-server over TLS, rather than through the Granting User's user-agent. This provides:

* **Request Integrity:** Authorization parameters cannot be tampered with by malicious browser extensions, compromised user-agents, or man-in-the-browser attacks.
* **Upfront Actor Validation:** The Authorization Server validates the `actor_token` before the Granting User is prompted, ensuring invalid or expired actors are rejected early.
* **Reduced Attack Surface:** The `actor_token` is never exposed in the user-agent's URL bar, history, or referrer headers.

Clients capable of using PAR SHOULD prefer it over Direct Authorization Requests for all delegation flows defined in this specification.

## CIBA Security Considerations

When CIBA {{OpenID.CIBA}} is used as the initiation mode, the security considerations in Section 13 of {{OpenID.CIBA}} apply. In addition, the following delegation-specific considerations apply:

* **Out-of-Band Channel Security:** The out-of-band channel MUST be secured against interception and replay per Section 13.1 of {{OpenID.CIBA}}.
* **Binding Message Authenticity:** The `binding_message` is the Granting User's primary means of understanding the delegation. Compromise of this message could lead to unauthorized delegation.
* **Polling Rate Limiting:** In poll mode, the Authorization Server MUST enforce the `interval` parameter per Section 10 of {{OpenID.CIBA}}.
* **Callback Endpoint Security (Ping and Push):** Callback endpoints MUST be pre-registered, TLS-protected, and authenticated via `client_notification_token` per Sections 11 and 12 of {{OpenID.CIBA}}. In push mode, compromise of the callback would expose delegation tokens.
* **Push Mode Actor Binding:** In push mode, fresh `actor_token` proof-of-presence cannot be enforced at issuance time. Push mode SHOULD NOT be used for delegation scenarios requiring actor proof-of-presence at token issuance.

## Binding Message Integrity

The `binding_message` in CIBA requests MUST accurately reflect the `intent_scope` and the requested permissions. The Authorization Server MUST reject any CIBA request where a mismatch is detected between the technical scopes and the human-readable `binding_message`, using the error code `invalid_binding_message`. This requirement prevents:

* **Social Engineering via Misleading Messages:** An attacker displaying a benign message (e.g., "Check your account balance") while requesting elevated permissions (e.g., `execute:trades`).
* **Scope/Intent Drift:** A misconfigured Client inadvertently presenting an outdated or incorrect description of the Delegated Actor's intended actions.

The Authorization Server SHOULD implement automated validation when `intent_scope` is registered with a known description. For unregistered `intent_scope` values, the Authorization Server MAY rely on policy-based heuristics or manual review.

## Actor Token Continuity

When the `actor_token` is submitted for upfront validation during PAR or CIBA initiation, the Authorization Server binds the validated actor identity to the authorization session.

**PAR and Direct modes:** At the token endpoint (for authorization code exchange), the Client MUST re-submit the `actor_token`, and the Authorization Server MUST verify that:

1. The re-submitted `actor_token` is still valid (not expired or revoked since the upfront validation).
2. The `sub` claim in the re-submitted `actor_token` identifies the same Delegated Actor that was bound during the initiation phase.

This two-phase validation ensures that the Delegated Actor remains authorized between the time the Granting User consents and the time the token is actually issued, mitigating time-of-check-to-time-of-use (TOCTOU) attacks.

**CIBA mode:** The `auth_req_id` returned by the CIBA authentication response serves as the secure handle that carries the validated actor binding through the entire transaction lifecycle. Because the Delegated Actor's identity is bound to the `auth_req_id` at initiation time, subsequent token endpoint requests (poll and ping modes) or server-initiated token delivery (push mode) SHOULD NOT require the `actor_token` to be re-submitted. The `auth_req_id` itself provides the continuity guarantee.

If the Authorization Server's policy requires a fresh proof-of-presence at the moment of token issuance (e.g., for high-assurance delegation scenarios), the Authorization Server MUST advertise this requirement and the Client MUST include the `actor_token` in the token request. In push mode, where no Client-initiated token request occurs, fresh proof-of-presence cannot be enforced at issuance time; therefore, push mode SHOULD NOT be used for delegation scenarios requiring token-issuance-time actor proof-of-presence.

## Proof Key for Code Exchange (PKCE)

PKCE {{RFC7636}} is REQUIRED for all authorization requests in this flow, to prevent authorization code interception attacks. This is especially important given that the authorization code ultimately results in a delegated token with an `act` claim, magnifying the impact of any code interception. The `code_challenge_method` MUST be `S256`.

## Single-Use and Short-Lived Authorization Codes

Authorization Codes MUST be single-use and have a short expiration time (RECOMMENDED no more than 10 minutes) to minimize the window for compromise. The Authorization Server MUST reject any attempt to reuse an Authorization Code and SHOULD revoke all tokens issued based on that code if reuse is detected.

## Binding Code to Delegated Actor and Client

The Authorization Server MUST bind the Authorization Code to the specific Granting User, Client (`client_id`), and Delegated Actor (`requested_actor`) during issuance and MUST verify this binding during the Token Request. If the `actor_token`'s `sub` claim does not match the `requested_actor` bound to the code, the Token Request MUST be rejected with the `actor_mismatch` error.

## Clear and Informed User Consent

The consent screen presented to the Granting User MUST clearly identify the Delegated Actor and the requested scopes to ensure the Granting User understands exactly what authority they are delegating and to whom. When an `intent_scope` is provided, the Authorization Server SHOULD display the intended action in human-readable form. The consent screen SHOULD explicitly indicate that an autonomous entity (not a human) will be exercising the delegated permissions.

## Privilege Escalation Prevention

Because Delegated Actors are autonomous and potentially dynamic, preventing privilege escalation is critical:

Scope Confinement:
: The Authorization Server MUST NOT issue an Access Token with scopes exceeding those explicitly consented to by the Granting User. The `act` claim in the Access Token signals to Resource Servers that the bearer is acting through delegation, and Resource Servers SHOULD apply additional scrutiny or restrictive policies to requests bearing an `act` claim.

Token Confinement:
: Access Tokens issued via this flow SHOULD have a short lifetime (RECOMMENDED no more than one hour) and SHOULD NOT be issued with refresh tokens unless the Authorization Server has established a high level of trust with the Delegated Actor. If refresh tokens are issued, they MUST be bound to the specific Delegated Actor and MUST be rotated on each use.

Compromised Actor Logic:
: If a Delegated Actor's logic is compromised (e.g., prompt injection in an AI agent, or code injection in an automated script), the attacker could attempt to use the delegated token for unintended purposes. To mitigate this risk:

  * Resource Servers SHOULD enforce fine-grained authorization policies based on the `act` claim, limiting the actions a specific Delegated Actor type can perform regardless of the scopes in the token.
  * Authorization Servers SHOULD support token revocation endpoints and provide mechanisms for Granting Users to revoke all active delegations to a specific Delegated Actor.
  * Clients SHOULD implement anomaly detection and rate limiting on actions requested by Delegated Actors.

Actor Impersonation:
: The Authorization Server MUST ensure that the `actor_token` presented at the token endpoint was issued to the same entity identified by `requested_actor`. This binding prevents a malicious Delegated Actor from presenting its own `actor_token` while using a code intended for a different (more privileged) Delegated Actor.

## Dynamic Consent Replay Prevention

Because the Dynamic Consent flow can be triggered repeatedly (e.g., by a compromised Resource Server sending spurious 401/403 challenges), the following mitigations apply:

* The Authorization Server SHOULD implement consent caching: if the Granting User has recently consented to the same (Client, Delegated Actor, scope) tuple, the Authorization Server MAY skip the interactive consent screen and issue a new code without re-prompting, subject to a configurable consent validity period.
* Clients MUST validate that the resource server challenge is legitimate before initiating the Dynamic Consent flow. Clients SHOULD NOT blindly redirect the Granting User to the Authorization Server on every 401/403 response.

## Auditability

The structured claims in the Access Token (`sub`, `azp`, `act`) provide essential information for auditing actions performed using the token, clearly showing who (Granting User) authorized the action, which application (Client) facilitated it, and which entity (Delegated Actor) performed it. Authorization Servers and Resource Servers SHOULD log delegation events for forensic and compliance purposes.

## Token Storage and Transmission

Access Tokens and Actor Tokens MUST be transmitted only over TLS-protected channels. Clients and Delegated Actors MUST store tokens securely and MUST NOT expose them in URLs, logs, or error messages.

# Use Case Examples

This section provides two complete examples demonstrating the generality of this specification: one involving an AI-driven personal assistant and one involving a traditional automated financial rebalancing script.

## Example 1: AI-Driven Personal Assistant

**Scenario:** A user ("Alice") uses a productivity application ("WorkHub") that integrates with an AI personal assistant ("AssistBot"). Alice asks AssistBot to schedule a meeting with a colleague by checking her calendar and sending an invitation on her behalf.

### Step 1: Delegated Actor Signals Intent

AssistBot (the Delegated Actor, identifier: `assistbot-v2`) signals to WorkHub (the Client) that it needs to read Alice's calendar and create an event.

### Step 2: Client Attempts Action

WorkHub makes a request to Alice's calendar service (the Resource Server):

~~~
GET /api/calendar/events HTTP/1.1
Host: calendar.example.com
Authorization: Bearer <existing_token_without_act_claim>
~~~

### Step 3: Resource Server Challenge

The calendar service determines the token lacks the required delegation claims and responds:

~~~
HTTP/1.1 401 Unauthorized
WWW-Authenticate: Bearer error="invalid_token",
  error_description="Access token missing required delegation claims"
~~~

### Step 4: Dynamic Consent via PAR (Human-in-the-Loop)

WorkHub submits the authorization parameters to the Authorization Server's PAR endpoint, including AssistBot's Actor Token for upfront validation:

~~~
POST /par HTTP/1.1
Host: auth.example.com
Content-Type: application/x-www-form-urlencoded

response_type=code&
client_id=workhub-app&
redirect_uri=https://workhub.example.com/callback&
scope=read:calendar write:calendar&
intent_scope=intent:schedule_meeting_with_colleague&
state=xy9z3k&
code_challenge=E9Melhoa2OwvFrEMTJguCHaoeK1t8URWbuGJSstw-cM&
code_challenge_method=S256&
requested_actor=assistbot-v2&
actor_token=eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCJ9...&
actor_token_type=urn:ietf:params:oauth:token-type:jwt
~~~

The Authorization Server validates the `actor_token` upfront, confirms that `assistbot-v2` is a recognized Delegated Actor, and returns a `request_uri`:

~~~
HTTP/1.1 201 Created
Content-Type: application/json
Cache-Control: no-store

{
  "request_uri": "urn:ietf:params:oauth:request_uri:workhub_cal_7f3a",
  "expires_in": 60
}
~~~

WorkHub then redirects Alice's browser using the `request_uri`:

~~~
GET /authorize?client_id=workhub-app&
  request_uri=urn:ietf:params:oauth:request_uri:workhub_cal_7f3a HTTP/1.1
Host: auth.example.com
~~~

### Step 5: Granting User Consent

The Authorization Server displays a consent screen to Alice:

> **WorkHub** is requesting that **AssistBot (v2)** be allowed to act on your behalf:
>
> - **Read** your calendar events
> - **Create** calendar events
>
> **Purpose:** Schedule a meeting with a colleague
>
> [Allow]  [Deny]

Alice clicks "Allow."

### Step 6: Authorization Code Issued

~~~
HTTP/1.1 302 Found
Location: https://workhub.example.com/callback?
  code=SplxlOBeZQQYbYS6WxSbIA&state=xy9z3k
~~~

### Step 7: Token Exchange with Actor Authentication

WorkHub exchanges the code at the token endpoint, including AssistBot's Actor Token. The Authorization Server verifies that this `actor_token` identifies the same Delegated Actor (`assistbot-v2`) that was validated during the PAR request (actor continuity check):

~~~
POST /token HTTP/1.1
Host: auth.example.com
Content-Type: application/x-www-form-urlencoded

grant_type=authorization_code&
client_id=workhub-app&
code=SplxlOBeZQQYbYS6WxSbIA&
code_verifier=dBjftJeZ4CVP-mB92K27uhbUJU1p1r_wW1gFWFOEjXk&
redirect_uri=https://workhub.example.com/callback&
actor_token=eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCJ9...&
actor_token_type=urn:ietf:params:oauth:token-type:jwt
~~~

### Step 8: Delegated Access Token Issued

~~~json
{
  "access_token": "eyJhbGciOiJSUzI1NiJ9...",
  "token_type": "Bearer",
  "expires_in": 3600,
  "scope": "read:calendar write:calendar"
}
~~~

Decoded JWT payload of the access token:

~~~json
{
  "iss": "https://auth.example.com",
  "aud": "https://calendar.example.com",
  "sub": "alice@example.com",
  "azp": "workhub-app",
  "scope": "read:calendar write:calendar",
  "exp": 1746009896,
  "iat": 1746006296,
  "jti": "tok-7f3a-1b2c",
  "act": {
    "sub": "assistbot-v2"
  }
}
~~~

### Step 9: Action Completed

WorkHub retries the calendar request with the new token. The calendar service validates the `act` claim and processes the request. AssistBot successfully schedules the meeting.

## Example 2: Automated Financial Portfolio Rebalancing Script

**Scenario:** An investor ("Bob") uses a brokerage platform ("InvestCo") that offers an automated portfolio rebalancing feature. A server-side script ("rebalancer-conservative-v1") periodically analyzes Bob's portfolio and executes trades to maintain his target allocation. This script is not an AI agent -- it is a traditional deterministic algorithm.

### Step 1: Delegated Actor Signals Intent

The rebalancing script (the Delegated Actor, identifier: `rebalancer-conservative-v1`) signals to the InvestCo platform (the Client) that it needs to read Bob's portfolio and execute trades using a conservative rebalancing strategy.

### Step 2: Client Attempts Action

InvestCo makes a request to the brokerage API (the Resource Server):

~~~
GET /api/portfolio/positions HTTP/1.1
Host: brokerage-api.example.com
Authorization: Bearer <existing_token>
~~~

### Step 3: Resource Server Challenge

The brokerage API determines the token lacks the required delegation claims for the rebalancing actor and responds:

~~~
HTTP/1.1 403 Forbidden
WWW-Authenticate: Bearer error="insufficient_scope",
  error_description="Token does not authorize automated trading",
  required_scope="read:portfolio execute:trades"
~~~

### Step 4: Dynamic Consent via CIBA (Out-of-Band)

Because the rebalancer operates as a background process and Bob is not actively using InvestCo, InvestCo initiates a CIBA authentication request with the rebalancer's `actor_token` for upfront validation:

~~~
POST /bc-authorize HTTP/1.1
Host: auth.investco.example.com
Content-Type: application/x-www-form-urlencoded

scope=read:portfolio execute:trades&
intent_scope=urn:investco:rebalance:conservative&
client_id=investco-platform&
login_hint=bob@example.com&
binding_message=Allow rebalancer-v1 to rebalance your portfolio (conservative)&
requested_actor=rebalancer-conservative-v1&
actor_token=eyJhbGciOiJFUzI1NiIsInR5cCI6IkpXVCJ9...&
actor_token_type=urn:ietf:params:oauth:token-type:jwt
~~~

The Authorization Server validates the `actor_token`, confirms `rebalancer-conservative-v1` is recognized, and returns:

~~~json
{
  "auth_req_id": "1c266114-a1be-4252-8ad1-04986c5b9ac1",
  "expires_in": 120,
  "interval": 5
}
~~~

### Step 5: Out-of-Band User Consent

Bob receives a push notification displaying the `binding_message`, the Delegated Actor identity, and the requested permissions. Bob reviews and taps "Authorize."

### Step 6: Token Polling

InvestCo polls the token endpoint per the CIBA poll mode. The `auth_req_id` carries the validated actor binding, so the `actor_token` is not re-submitted:

~~~
POST /token HTTP/1.1
Host: auth.investco.example.com
Content-Type: application/x-www-form-urlencoded

grant_type=urn:openid:params:grant-type:ciba&
auth_req_id=1c266114-a1be-4252-8ad1-04986c5b9ac1&
client_id=investco-platform
~~~

The Authorization Server returns `authorization_pending` until Bob responds.

### Step 7: Delegated Access Token Issued

Once Bob approves, the Authorization Server resolves the `auth_req_id` and issues the delegated Access Token.

Decoded JWT payload of the access token:

~~~json
{
  "iss": "https://auth.investco.example.com",
  "aud": "https://brokerage-api.example.com",
  "sub": "bob@example.com",
  "azp": "investco-platform",
  "scope": "read:portfolio execute:trades",
  "exp": 1746009896,
  "iat": 1746006296,
  "jti": "tok-fin-9d4e",
  "act": {
    "sub": "rebalancer-conservative-v1"
  }
}
~~~

### Step 8: Action Completed

InvestCo retries the portfolio request with the new token. The brokerage API validates the `act` claim, confirms that `rebalancer-conservative-v1` is an authorized trading actor, and processes the portfolio read. The rebalancer script analyzes the positions, computes the necessary trades, and submits them through InvestCo using the same delegated token.

 
--- back
