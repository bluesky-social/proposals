# 0004 OAuth 2.0 for the AT Protocol

_This proposal written by Matthieu Sieben, Devin Ivy, and Daniel Holmgren, on behalf of Bluesky. First posted in February 2024._

## Introduction

[ATPROTO] describes how decentralized entities can communicate in order to provide a world class social networking technology. Users' messages are exchanged from their own/shared Personal Data Servers (PDS) using a federated networking model.

As of writing of this document, there are basically two ways to obtain credentials to control a user's PDS:

- Using the user's credentials (user handle + user password)
- Using an "app password"

Only the latter method is safe to use with a third party application or service (client) that would like to act on the user's behalf. However, this method of obtaining credentials is not very user friendly and does not provide the UX end users are used to.

OAuth2 is a well known, well documented framework specifically made for granting user credentials to third party clients. However, it was not initially designed to work in a fully decentralized environment, such as the [ATPROTO] network. The main issue blocking its direct adoption is that, in OAuth2, clients need to be pre-registered and known by the Authorization Server (AS) before credentials can be granted. The [OAuth 2.0 Dynamic Client Registration Protocol][RFC7591] describes how clients can dynamically register into an AS. However, this method has several disadvantages:

- Clients need to keep a state for all the AS they registered to, making them harder to implement and maintain.
- The client credentials obtained from the AS during the registration can get lost, without any way for the clients recover them autonomously.
- The protocol does not provide a good protection against "theft of identity" (a non-legitimate client registering with the same name, logo, etc. as another, legitimate, client).

This proposal describes an alternative way of performing client registration. This method is relies on being able derive the client metadata document from its client id, allowing clients to be registered on the fly.

This proposal also describes the minimal OAuth requirements that clients and [ATPROTO] servers must implement in order to be able to interact with each other. The choices in this document are based on state of the art security practices and were largely influenced by [DRAFT-OAUTH-BROWSER-BASED-APPS].

### Terminology

Along with the terms defined in the various RFCs listed in the [references](#references) (OAuth 2.0 [RFC6749], JSON Web Token (JWT) [RFC7519], etc.), this draft uses the terms defined hereafter:

- Global client identifier: a globally unique identifier for a client. This differs from the "client identifier" defined by OAuth 2.0 [RFC6749] in the sense that, in the OAuth 2.0 framework, the client identifier is only unique for a given AS. In the current framework, we need clients to be uniquely identified across all AS' in a verifiable way. In this document, "Client ID" refers to the global client identifier of the client.
- Personal Data Server (PDS): the "resource server" that hold end-users' data, as defined by [ATPROTO].
- Entryway: In large PDS deployments, it is likely that users will be hosted on numerous individual PDSes. In this case, a special server, the entryway, is responsible for handling unauthenticated requests. This includes the OAuth2 authentication endpoints described in this document. The entryway will thus act as an "Authorization Server", as defined by OAuth 2.0 [RFC6749]. In smaller deployments (e.g. self-hosted PDS), the PDS and entryway may be the same entity/server.

### Goals

The goals we try to achieve through this framework are:

- Allow clients to obtain user credentials in order to interact with users' PDS, without having to be registered with those AS beforehand.
- Allow the Authorization Server (Entryway) to verify that authorization requests are coming from a legitimate client, and are properly formatted & scoped for that client.
- Ensure that clients never lose their ability to interact with the Authorization Server (avoid loss of credentials).
- Allow backend, browser based & native apps to obtain credentials using state of the art security practices.

## Framework

This framework builds on top of the [OAuth 2.0 Authorization Framework][RFC6749], the [OAuth2.1 draft][DRAFT-OAUTH-V2-1], the [OAuth 2.0 Security Best Current Practice][DRAFT-OAUTH-SECURITY-TOPICS] and the [OAuth 2.0 for Browser-Based Apps (BCP draft)][DRAFT-OAUTH-BROWSER-BASED-APPS].

When a client needs to obtain credentials to interact with a user's PDS, it must initiate an authorization flow with the PDS's Authorization Server (AS). In order to determine the AS's authorization metadata, the client must first resolve the PDS URI from the user's handle (see [ATPROTO]). Once the PDS url is known (e.g. `https://shimeji.us-east.host.bsky.network`), the [Authorization Server Metadata][RFC8414] endpoint (`<PDS_ORIGIN>/.well-known/oauth-authorization-server`) will allow the client to obtain all the information it needs to initiate an OAuth2 authorization flow (see the [server metadata](#server-metadata) section below).

Clients do not need to be pre-registered in the Authorization Server. Instead, the Authorization Server will dynamically load the client metadata from the client_id, during an authorization request, as described in the [client metadata](#client-metadata) section. The client metadata is a JSON file that contains the client's metadata (name, logo, allowed redirect URIs, expected scopes, JWKS, etc.). The content of that document is based on the [OAuth 2.0 Dynamic Client Registration Protocol][RFC7591] and [OAuth 2.0 Dynamic Client Registration Management Protocol][RFC7592] specifications.

Since clients do not register themselves with the Authorization Server, they will not exchange a client secret with the AS. Instead, they will use a public/private keypair to authenticate themselves, using the `urn:ietf:params:oauth:grant-type:jwt-bearer` grant type as described in [JWT for Assertion Framework protocol][RFC7523]. The client will expose its public keys through the `jwks` & `jwks_uri` client metadata. Since browser & native apps clients are unable to use private keys in order to authenticate themselves, these will act as public clients.

This framework requires the following specifications to be used during the authorization flow:

- PAR: Pushed Authorization Requests ([RFC9126]). Because of the reasons listed hereafter, and because of the added security it provides, the use of PAR is **mandatory** for all clients. Frontend client must also use (unauthenticated) PAR requests.

  > Note: In the future, we might relax this requirement for frontend clients, but for now, we want to keep things simple & consistent across all clients.

  - PAR allows clients to authenticate themselves before the authorization flow (using `client_assertion`), allowing the Authorization Server to propose a different UX to confidential clients (e.g. SSO, etc.).
  - PAR allows clients to send a very large authorization request payload (e.g. a request object containing a lot of claims) without having to worry about URL length limitations.
  - PAR improves security and privacy by allowing clients to send sensitive information (e.g. `code_verification`, etc.) directly to the Authorization Server, without having to send this data through the front channel.

- PKCE: Proof Key for Code Exchange ([RFC7636]), which is part of the [OAuth2.1 draft][DRAFT-OAUTH-V2-1]. It is used to prevent any entity, other than the one that initiated the authorization request, to exchange the authorization code for an access token. The main impact of this choice is that the implicit grant type **must not** allow clients to obtain access tokens. Instead, all clients **must** contact the "token endpoint" after they are redirected from the "authorization endpoint" in order to obtain an access token. The OAuth client performing the authorization flow must be able to store a session securely. Only the `S256` challenge method is allowed by this specification. In the future, we might allow `plain` challenge method, but only for authenticated clients.

- DPoP: Demonstrating Proof of Possession ([RFC9449]) is used to **bind** tokens (access tokens & refresh tokens) to a client instance that holds a specific private key. This is done by providing a "proof of possession" when requesting for new tokens. The Authorization token will then issue a "sender-constrained" Access Token (the public DPoP key of the client is included in the access token). The client instance will then be able to access the resource server (PDS) by providing **both** the access token and a recently generated DPoP proof. This mechanism protects against token theft and replay attacks.

The general framework is described in figure 1.

```none
┌───────┐                   ┌────────┐                       ┌─────┐ ┌──────────────────────┐
│ User  │                   │ Client │                       │ PDS │ │ Authorization Server │
└───┬───┘                   └───┬────┘                       └──┬──┘ └────────────┬─────────┘
    │ User enters @handle (1)   │                               │                 │
    ├──────────────────────────►│                               │                 │
    │                           ├──┐                            │                 │
    │                           │  │ Client resolves PDS URL (2)│                 │
    │                           │◄─┘                            │                 │
    │                           │                               │                 │
    │                           │ GET /.well-known         (3)  │                 │
    │                           │  /oauth-authorization-server  │                 │
    │                           ├──────────────────────────────►│                 │
    │                           │                               │                 │
    │                           ├──┐                            │                 │
    │                           │  │ Validate issuer (4)        │                 │
    │                           │◄─┘                            │                 │
    │                           │                               │                 │
    │                           │ Pushed Authorization Request (5)                │
    │                           ├───────────────────────────────┬────────────────►│
    │                           │                               │                 │
    │                           │ Client metadata discovery (6) │                 │
    │                           │◄──────────────────────────────┼─────────────────┤
    │ User redirected to        │                               │                 │
    │ authorize URL (7)         │                               │                 │
    │◄──────────────────────────┤                               │                 │
    │                           │                               │                 │
    │ redirected to authorize URL (8)                           │                 │
    ├───────────────────────────┬───────────────────────────────┼────────────────►│
    │                           │                               │                 ├──┐
    │                           │                               │                 │  │ Verification
    │ User authenticates themself on AS and approves the request (10)             │◄─┘ (9)
    │◄──────────────────────────┬───────────────────────────────┬────────────────►│
    │                           │                               │                 │
    │ User redirected to redirect_uri (11)                      │                 │
    ├──────────────────────────►┐                               │                 │
    │                           │                               │                 │
    │                           │ Token retrieval (12)          │                 │
    │                           ├───────────────────────────────┼────────────────►│
    │                           │                               │                 ├──┐
    │                           │                               │                 │  │ Session
    │                           │ Tokens are issued and returned to the client(14)│◄─┘ created (13)
    │                           │◄──────────────────────────────┬─────────────────┤
    │                           │                               │                 │
    │                           │ Client makes API requests (15)│                 │
    │                           ├──────────────────────────────►│                 │
    │                           │                               │                 │
```

Figure 1.: Authorization flow for a client. Note that neither the client nor the authorization server necessarily know about each other before the authorization flow is initiated.

First step (1) is to ask, through the client, the user for their [ATPROTO] `@handle`, or, should they have forgotten their handle, their PDS or Entryway's URL (e.g. `bsky.social`). The client will then load (3) and validate (4) the Authorization Server Metadata using the method described [below](#server-metadata).

The client will then build (5) the authorization URL using [PAR][RFC9126] & [PKCE][RFC7636]. This will cause the Authorization Server to load and validate the client metadata (6) using the method described [below](#client-metadata). Once the authorization request is successfully created, the end user will be redirected to the authorize endpoint (7, 8).

Whenever receiving a request for a particular client, the Authorization Server **must** perform the following steps (9):

- Verify that the client ID is in the right format (see the [global client identifier](#global-client-identifier) section)
- Load & verify the client metadata, deduced from the client ID (see the [client metadata](#client-metadata) section)
- Verify that the authorization request is compliant with the current spec (see the [authorization request](#authorization-request) section)

During the user authorization step (10), if the user never approved a request from a particular client, the [AUTH-UI] will contain any relevant information required for the user to be able to make an informed decision (see the [Impersonation](#impersonation) section below). Note that if the client was authenticated during PAR (step 5), the Authorization Server can decide to grant an increased level of trust to the client, and thus skip some of the authorization UI steps. For example, AS's **should not** allow silent sign on for unauthenticated clients. Similarly, the AS **should** always require user consent for unauthenticated clients (even if consent was already granted before).

During token retrieval (12), the Authorization Server **must** ensure the Token Request is compliant with the current spec (see the [token request](#token-request) section).

In addition to the requested tokens, the token response must also contain, in the `sub` claim, the user's [ATPROTO] distributed identifier (DID). This value will allow clients to resolve the PDS url using the [DID-PLC] resolution method. If the authorization request was initiated (in step 1) using the user's `@handle`, the client **must** verify that the token response's `sub` claim matches the DID that was resolved from this handle. If no `@handle` was provided when initiating the flow, the client **MUST** perform the [DID-PLC] + Issuer resolution mechanism using the token response's `sub`. This is done to verify that the `iss` of the token response matches the `issuer` from the Authorization Server Metadata resolved from the DID. It is critical that the client checks that the `sub` is indeed hosted and managed by the `iss`, and vice versa, whenever a token response is received.

In addition to being bound to the client's DPoP key, all tokens issued by the AS will also be bound to the public key used by the client to authenticate itself through the `urn:ietf:params:oauth:grant-type:jwt-bearer` grant (only for public client). If, at any point, a client stops advertising a public key that is used in active sessions, all those sessions will be invalidated. The AS will choose, at its discretion how it will implement this (proactively, whenever metadata are fetched, on the next token refresh, etc.).

Because frontend only apps are not able to provide a similar mechanism to invalidate credentials at scale, [DRAFT-OAUTH-BROWSER-BASED-APPS] requires that refresh tokens "MUST set a maximum lifetime [...] or expire if they are not used in some amount of time". As an additional restriction, unauthenticated clients are not allowed use silent sign on. This means that the user will have to give its consent again (but _not_ necessarily re-authenticate itself if he still has an active session on the AS) to the client every time a period of inactivity is reached. In practice, after some inactivity period, the user will be redirected to the authorization server to re-authorize the client with a message saying: "You have been inactive for 48 hours on `bsky.app`. You are still authenticated as John Doe. To continue, please re-authorize the `bsky.app` to access your account by clicking this button".

Authenticated clients **should** rotate their keys on a regular basis (e.g. on every new app release, or every month, whichever comes first). They can do so by adding a new key to their JWKS and removing the old ones from time to time. If a breach is detected, the client **must** immediately remove the compromised key from its JWKS. If the mitigation of the breach takes long, all the keys **must** be removed from the JWKS as the issue is being fixed, preventing any new tokens from being issued.

### Global client identifier

A global client identifier is a string that uniquely identifies a client. The client identified must allow the Authorization Server to retrieve the [client metadata](#client-metadata) document.

This framework relies on the domain name system (DNS) by identifying clients through their **fully qualified domain names** (FQDN). This choice is motivated by the fact that DNS is: globally available, allows Authorization Server to retrieve client metadata information (through a well-known HTTP endpoint) and is understood by users as a source of trust (e.g. users trust that `bsky.app` is owned by Bluesky, `google.com` by Google, etc.).

Domain names used as client IDs **must** have a suffix registered in the [Public Suffix List][PSL]. The only exception to this rule is `localhost`, which **must** be used for local development only.

The client ID **must** be the normalized version of the FQDN. This means that the client ID **must** be the FQDN, lowercased, with the trailing dot removed (if any). Segments containing non ASCII characters **must** be punycoded.

Authorization Server **must not** accept client IDs that are not compliant with this specification.

### Client metadata

Instead of relying on dynamic client registration, clients will be automatically/lazily registered by the AS when they first initiate an authorization flow. In order to do so, the AS will need to be able to resolve the the client metadata from the client's [Global identifier](#global-client-identifier).

In order to load the client metadata, the AS will:

1. Build the "client url" by prepending `https://` to the client ID.
2. Build the "client metadata url" by appending `/.well-known/oauth-client-client` to the client url.
3. Fetch the JSON document from the "client metadata url".
4. Load any related resources (such as the document reference by the `jwks_uri` metadata)
5. Verify that the retrieved metadata and JWKS are valid. Any client metadata that does not satisfy this specification **must** be rejected by the AS.

In addition to be conformant with the Client Metadata described in [RFC7591], the following rules also apply to the client metadata document. These rules **must** be enforced by the AS.

- the metadata **must** contain `client_id`, and this value **must** be strictly equal to the client id that was used to resolve this document.
- the metadata **must** contain `"dpop_bound_access_tokens": true`. All clients must use [DPoP][RFC9449] proof when requesting tokens.
- the metadata **must** contain at least one `redirect_uris` entry.
- the `application_type`, if present, **must** either be `"web"` or `"native"` (default is `web`).
- `"web"` clients must use HTTPS for all their `redirect_uris`.
- redirect uri using the `https:` scheme **must** be on the same origin as the client, regardless of the the `application_type`.
- redirect uri using the `http:` scheme are only allowed for `"native"` clients.
- redirect uri using the `http:` scheme **must** use either `127.0.0.1` or `[::1]` in their hostname component. Any other hostname (including `localhost`) is forbidden. The port **must not** be specified as the AS will allow any port when validating loopback redirect uris.
- `"native"` clients are allowed to specify redirect uris using custom schemes (e.g. `app.bsky:/callback`). The custom scheme **must** be the reverted domain name of the client (e.g. `com.example.app:` when the client id is `app.example.com`), and must contain at least one `.` character.
- When custom schemes are used in redirect uri, only a single slash (`/`) character is allowed after the scheme (e.g. `app.bsky:/callback` is allowed, but `app.bsky://callback` is not).
- the `client_uri` metadata, if present, **must** be on the same origin as the URL derived from the client ID.
- the `subject_type` metadata, if present, **must** be `"public"`.
- the `grant_types` metadata **must** contain `authorization_code`. It _may_ also contain `refresh_token`.
- the `response_types` metadata **must** contain `code`. Other response types can be added (e.g. `id_token`) but won't necessarily be supported by the AS.
- the `scope` metadata **must** contain `offline_access` if, and only if, `refresh_token` is present in `grant_types`.
- the `scope` metadata can be used to restrict which scopes are allowed during the authorization flow for that client. No scope are allowed by default.
- the `response_types` _may_ contain `"token"`. However, since [PKCE][RFC7636] is mandatory for all exchanges, AS **must** only allow `"token"` response type to be used when PKCE is irrelevant (such as during the `password` grant type).
- every `token_endpoint_auth_method` (where `<endpoint>` is `token`, `revocation`, `introspection`), if present, **must** be set to `private_key_jwt` or `none`.
- if any of the `token_endpoint_auth_method` is set to `private_key_jwt`, the client **must** provide a JSON web key set (JWKS), either through the `jwks` metadata or through the `jwks_uri` metadata, that contains at least one key.
- `jwks` and `jwks_uri` **must not** be used together.

> Note: The AS **must not** cache any data related to a client for a period of time longer than 1 minute.

#### Client metadata for local development

When using `localhost` as client ID, the AS will not be able to resolve the client metadata using the method described above. Instead, the Authorization Server will use the following client metadata:

```json
{
  "client_id": "localhost",
  "client_name": "Loopback atproto client",
  "client_uri": "http://localhost/",
  "response_types": ["code"],
  "grant_types": ["authorization_code"],
  "redirect_uris": ["http://127.0.0.1/", "http://[::1]/"],
  "token_endpoint_auth_method": "none",
  "application_type": "native",
  "dpop_bound_access_tokens": true
}
```

> Note: the client_name is up to the provider implementation

OpenID compliant server implementation are encouraged to add the `code id_token` response types as well:

```json
{
  // [props omitted for clarity]

  "scope": "profile openid",
  "response_types": ["code", "code id_token"],
}
```

> Note: OpenID support should be inferred by the presence of an `openid` scope in the server metadata.

### Server Metadata

In order to retrieve the server metadata, the client will build the metadata URL from the PDS URL (e.g. `https://bsky.social`) and fetch it (e.g. `https://bsky.social/.well-known/oauth-authorization-server`). The client **must** follow any redirection that occurs during this process.

If the user does not know the PDS URL, it can be resolved from the user's `@handle` and [DID] document. One way of doing this is to use the HTTP and DNS methods described in [ATPROTO]. Another, which is more convenient for browser based clients (as they are not able to make DNS requests), is to use a public API that resolves the user's handle to its PDS URL. Bluesky provides such an API:

```http
GET https://api.bsky.app/xrpc/com.atproto.identity.resolveHandle?handle=<handle_here>
```

The response is a JSON object containing user's DID

```json
{ "did": "did:plc:123" }
```

This DID can then be used to obtain the PDS's location using the [DID-PLC] or [DID-WEB] method:

```json
{
  // [omitted]
  "service": [
    {
      "id": "#atproto_pds",
      "type": "AtprotoPersonalDataServer",
      "serviceEndpoint": "https://shimeji.us-east.host.bsky.network"
    }
  ]
}
```

By building the metadata URL (`https://shimeji.us-east.host.bsky.network/.well-known/oauth-authorization-server`), and following any redirects, the client can obtain the PDS's [Authorization Server Metadata][RFC8414].

Once the client retrieved the Authorization Server Metadata, it **must** verify the following items. Any authorization flow with an AS not compliant with these rules **must** be rejected by the client.

- `issuer` **must** be a valid URL on the same origin as the URL that was used to fetch the document. If a redirection occurred, the URL of the last HTTP request **must** be used to check the metadata's `issuer` (see [DRAFT-OAUTH-SECURITY-TOPICS] section 4.4).
- `issuer` **must** be an HTTPS URL in the form `https://<domain>[:<port>]/`. The port **must** be omitted if it is the default port for the scheme (e.g. `443` for `https`). The `http:` scheme **must not** be used.
- `response_types_supported` **must** at least contain `code`.
- `grant_types_supported` **must** at least contain `authorization_code` and `refresh_token`.
- `code_challenge_methods_supported` **must** contain `S256`.
- `token_endpoint_auth_methods_supported` **must** contain both `private_key_jwt` and `none`.
- `token_endpoint_auth_signing_alg_values_supported` **must** contain `ES256`.
- `scopes_supported` **must** contain `refresh_token`, `email`, `profile`.
- `subject_types_supported`, if present, **must** contain `public`.
- `authorization_response_iss_parameter_supported` **must** be `true`. Both AS' and clients **must** be [RFC9207] compliant.
- `pushed_authorization_request_endpoint` **must** be set. Both AS' and clients **must** be [RFC9126] compliant.
- `require_pushed_authorization_requests` **must** be set to `true`.
- `dpop_signing_alg_values_supported` **must** be set and contain `ES256`.
- `require_request_uri_registration`, if present, **must** be `true`.

### Authorization Server requirements

In addition to the [OAuth 2.0 Security Best Current Practice][DRAFT-OAUTH-SECURITY-TOPICS], the AS **must** also comply with the following rules:

- The following scopes **must** be supported: `email`, `profile`, `offline_access`
- The `code` response type **must** be supported.
- The `ES256` JWT verification algorithm must be supported for any JWT verification (e.g. `client_assertion`, `dpop` proof, etc.)
- [Pushed Authorization Request][RFC9126] **must** be enforced (`require_pushed_authorization_requests` in the server metadata must be `true`).
- The [Pushed Authorization Request][RFC9126] endpoint **must** support the same authentication methods as the token endpoint (namely `private_key_jwt` and `none`).
- grant types: `authorization_code` & `refresh_token`
- Response modes: `fragment`, `query`, `form_post`
- The Token & PAR Endpoints **must** support the `none` & `private_key_jwt` Authentication Methods.
- They **must** require PKCE for all authentication requests.
- The `code_challenge_methods_supported` server metadata ([PKCE][RFC7636]) **must** contain `RS256`. The AS _should not_ allow the `plain` challenge method to be used.
- They **must** require DPoP for all tokens requests. They **must** support DPoP for authorization requests.
- Access tokens **must** have a maximum lifetime of 1 hour.
- The OAuth authorization server metadata must be expose through the `oauth-authorization-server` well-known endpoint (see the [server metadata](#server-metadata) section)
- Unauthenticated clients **must not** be issued refresh tokens with a total lifetime longer than 48 hours
- refresh tokens **must** be bound to the client's DPoP key
- refresh tokens **should** be rotated any time they are used. If a previous refresh token is replayed, the AS **must** revoke the currently active refresh token. See [DRAFT-OAUTH-SECURITY-TOPICS] (section 4.14.2).

### Authorization Request

The following rules must be enforced by the AS when receiving an authorization request.

- Ensure that [PKCE][RFC7636], with the `S256` challenge method, is used.
- The `token` response type **must not** be allowed as it is not compatible with PKCE.
- If a DPoP proof is provided during PAR, ensure that the DPoP key is the same as the one used during the token request.
- `redirect_uri` _may_ be omitted when initiating authorization requests if the client has **exactly** one `redirect_uris` defined.
- `redirect_uri`, when provided, **must** exactly match one of the `redirect_uris` defined in the client metadata. The only exception to this rule concerns loopback redirect uris (for `native` client only), where the PORT, and only he PORT, **must** be ignored when comparing the `redirect_uri` against the client's `redirect_uris`
- Ensure that the requested `scope` values is a subset of the list of scopes defined in the client metadata.
- Authorization response **must** contain an `iss` parameter ([RFC9207]). The client **must** verify that the `iss` as per [RFC9207].

### Token Request

The following rules must be enforced by the AS when receiving a token request.

- Ensure that [DPoP][RFC9449] is used and bind both the access token and the refresh token to the client's DPoP key. If a DPoP proof is provided during PAR, ensure that the DPoP key is the same as the one used during the token request.
- Public clients **must** generate a new keypair for each set of tokens they request. Authorization servers **should** reject initial tokens requests from public clients that use the same keypair as a previous request.
- Ensure that the client authentication method is the same as the one used during PAR (if `client_assertion` was used during PAR, the same authentication method **must** be used, with a JWT containing a distinct `jti` claim, but signed by the same key)
- DPoP proof and client assertion **must** be signed using a different keypair.
- The Token Response from the AS **must** contain a `sub` claim, which must be the end-user's ATPROTO identifier (`did:plc:123`)

## Supported architectures

### Backend for Frontend

In this architecture, all secrets are kept on the backend. This means that the access tokens are **bound** to the backend instance that requested them. This effectively prevents any frontend from accessing the resource server (PDS) directly. Instead, if a frontend app needs to perform a request to the PDS, it will have to use its backend as a proxy to do so. The backend will then load all the credentials linked to the user (using any session mechanism defined on that backend), and will perform the request on behalf of the user, generating the DPoP proof and refreshing the tokens if need be. This is the only architecture that allows the backend server to contact the PDS from a background worker since the DPoP key is stored on the backend server.

```none
                            +-------------+  +--------------+ +--------------+
                            |             |  |              | |              |
                            |Authorization|  |    Token     | |   Resource   |
                            |  Endpoint   |  |   Endpoint   | |    Server    |
                            |             |  |              | |              |
                            +-------------+  +--------------+ +--------------+
                                ^                        ^              ^
                                |                     (F)|           (K)|
                                |                        v              v
                                |         +-----------------------------------+
                                |         |                                   |
                                |         |    Backend for Frontend  (BFF)    |
                             (D)|         |                                   |
                                |         +-----------------------------------+
                                |           ^     ^     ^     +       ^  +
                                |      (B,I)|  (C)|  (E)|  (G)|    (J)|  |(L)
                                v           v     v     +     v       +  v
+-----------------+         +-------------------------------------------------+
|                 |  (A,H)  |                                                 |
| Static Web Host | +-----> |                    Browser                      |
|                 |         |                                                 |
+-----------------+         +-------------------------------------------------+
```

This figure was taken from the section 6.1.1 of [DRAFT-OAUTH-BROWSER-BASED-APPS]. It shows that the backend is the only entity that can contact the resource server (PDS). It can do so either on its own (from a background worker) or on behalf of a user (by proxying the requests).

In this architecture, the authorization flow works as follows:

- The backend app is configured with **two** keypairs (it is recommended to use different keys for different purposes):

  - One for signing DPoP proofs
  - One for signing JWTs (used by the backend client to authenticate itself through `client_assertion` in the token request)

- The backend receives either the user's `@handle` (and uses it to resolve the PDS url) or the Entryway URL directly.
- The backend loads the ISSUER's [Authorization Server Metadata](#server-metadata).
- The backend builds the authorization URL using [PKCE][RFC7636] & [PAR][RFC9126] by `POST`ing an HTTP request to the `<pushed_authorization_request_endpoint>` (retrieved from the Authorization Server Metadata).
- The backend redirects the user to the `<authorization_endpoint>` (retrieved from the Authorization Server Metadata), including the `request_uri` received during the previous step in the URL.
- Upon successful authorization, the authorization server redirects the user back to the backend's `<redirect_uri>` with an authorization `code` in the URL.
- The backend uses the `code`, along with a `client_assertion` (JWT) and a `dpop` proof, to request an access token from the `<token_endpoint>` (retrieved from the Authorization Server Metadata).
- The backend stores the access token & refresh token in its secure storage, linked to the internal user entry of the user who initiated the authorization flow.
- Any time a request must be made to a PDS, on behalf of a user, the backend loads the user's credentials (access token, refresh token) and uses them to generate a DPoP proof and perform the request.

### Token-Mediating Backend

In this architecture, a backend is used to obtain and manage tokens on behalf of a frontend app. The frontend app maintains a session with its backend. The backend exposes the access token to the frontend app through an API. The frontend app can then use the access token to access the resource server (PDS) directly through a cross-origin request.

Since the entity that is going to make the requests on the resource server (PDS) is the browser, it will also be the entity that will have to provide the DPoP proof. This means that the frontend app will have to generate a keypair and store the private key in the browser's indexedDB. The private key will then be used to generate the DPoP proof.

These proofs will be used both when requesting the new tokens through the backend API, and when accessing the resource server (PDS) directly.

```none
                            +-------------+  +--------------+ +--------------+
                            |             |  |              | |              |
                            |Authorization|  |    Token     | |   Resource   |
         +-----------------+|  Endpoint   |  |   Endpoint   | |    Server    |
         |                  |             |  |              | |              |
         |                  +-------------+  +--------------+ +--------------+
         |                      ^                   ^                 ^
         |                      |                (F)|                 |
         |(D')                  |                   v                 |
         |                      |   +-----------------------+         |
         |                      |   |                       |         |
         |                      |   |Token-Mediating Backend|         | (J)
         |                   (D)|   |                       |         |
         |                      |   +-----------------------+         |
         |                      |       ^     ^     ^     +           |
         |                      |  (B,I)|  (C)|  (E)|  (G)|           |
         v                      v       v     v     +     v           v
+-----------------+         +-------------------------------------------------+
|                 |  (A,H)  |                                                 |
| Static Web Host | +-----> |                    Browser                      |
|                 |         |                                                 |
+-----------------+         +-------------------------------------------------+
```

This figure was taken from the section 6.2.1 of [DRAFT-OAUTH-BROWSER-BASED-APPS]. In this scenario, the browser obtains tokens from the Token-Mediating Backend and uses them to access the Resource Server (J).

In this architecture, the authorization flow works as follows:

- The backend app is configured with one keypair for signing JWTs (used by the backend client to authenticate itself through `client_assertion` in the token request)
- The backend asks the user for their `@handle` and/or their PDS URL.
  - The PDS URL, if omitted, is resolved from the `@handle` using common HTTP or DNS resolution. The backend will be able to perform DNS resolution itself.
- The backend loads the ISSUER's [Authorization Server Metadata](#server-metadata).
- The backend builds the authorization URL using [PKCE][RFC7636] & [PAR][RFC9126] by POSTing an HTTP request to the `<pushed_authorization_request_endpoint>` (retrieved from the Authorization Server Metadata) (D).
- The AS loads & validates the client metadata (D')
- The backend redirects the user to the `<authorization_endpoint>` (retrieved from the Authorization Server Metadata).
- Upon successful authorization, the authorization server redirects the user back to the backend's `<redirect_uri>` with an authorization `code` in the URL.
- The backend stores that `code` in the user's session.
- The frontend app is loaded in the browser.
- The app creates a **non exportable** asymmetric key pair ([WEBCRYPTO]). The private key should be stored in the browser's indexedDB. This keypair will be used for generating DPoP proofs.
- The frontend app asks the backend for an access token by providing a `dpop` proof, along with its session id.
- The backend uses the `code` (loaded from the session data), along with a `client_assertion` (JWT) and the frontend app's `dpop` proof, to request an access token from the `<token_endpoint>` (retrieved from the Authorization Server Metadata).
- The backend stores the access token & refresh token in its secure storage, linked to the user's session.
- The backend returns the access token to the frontend app.
- The frontend app uses the access token to access the resource server (PDS) by providing both the access token and a DPoP proof, generated using its keypair.
- When the tokens need to be refreshed, the frontend app calls the backend again, providing the session-id cookie and a `dpop` proof. The backend then uses the refresh token to obtain a new access token, and returns it to the frontend app.

### Progressive Web App (PWA)

In this architecture, the client is a frontend only app. It can be a mobile app, an app hosted on github pages, or any other app that does not rely on a dedicated backend.

```none
                      +---------------+           +--------------+
                      |               |           |              |
       +------------+ | Authorization |           |   Resource   |
       |              |    Server     |           |    Server    |
       |              |               |           |              |
       |              +---------------+           +--------------+
       |                     ^     ^                 ^     +
       |(C)                  |     |                 |     |
       |                     |(B)  |(D)              |(E)  |(F)
       |                     |     |                 |     |
       |                     |     |                 |     |
       v                     +     v                 +     v
+-----------------+         +-------------------------------+
|                 |   (A)   |                               |
| Static Web Host | +-----> |           Browser             |
|                 |         |                               |
+-----------------+         +-------------------------------+
```

This figure was taken from the section 6.3.1 of [DRAFT-OAUTH-BROWSER-BASED-APPS]. We can see the Browser negotiating with the Authorization Server (B), and then interacting directly with the Resource Server (D,E).

The client metadata **must not** contain any public key (JWKS) as the client, being "serverless", has no way to securely store & use any private key.

In this architecture, the authorization flow works as follows:

- The app asks the user for their `@handle`. This will allow the client to resolve the user's PDS using the `com.atproto.identity.resolveHandle` lexicon form a public PDS host (DNS resolution cannot be performed by a browser app) + `https://plc.directory/<did>`
  - If the user is not able to provide its handle, or if the the handle does not resolve anymore, the app will need to ask the user for its entryway URL (="ISSUER").
- The app loads the Authorization Server Metadata (`https://<ISSUER>/.well-known/oauth-authorization-server`).
- The app builds the authorization URL using [PKCE][RFC7636] & [PAR][RFC9126] by POSTing an HTTP request to the `<pushed_authorization_request_endpoint>` (retrieved from the Authorization Server Metadata) (B).
- The AS loads & validates the client metadata (C)
- The app redirects the user to the `<authorization_endpoint>` (retrieved from the Authorization Server Metadata), either in the same tab or in a popup/new tab.
- Upon successful authorization, the authorization server redirects the user back to the app's `<redirect_uri>` with an authorization `code` in the URL.
- The app creates a **non exportable** asymmetric key pair ([WEBCRYPTO]). The private key should be stored in the browser's indexedDB so it can be re-used after a reload of the browser. This keypair will be used for generating DPoP proofs.
- The app uses the `code`, along with a `dpop` proof, to request an access token from the `<token_endpoint>` (retrieved from the Authorization Server Metadata).
- The app stores the tokens in the browser's storage (typically, indexedDB).
- The app resolved the PDS url by using `aud` obtained by querying the `<introspection_endpoint>` (retrieved from the Authorization Server Metadata).
- The app can now make authenticated requests to the resource server (PDS) by providing both the access token and a freshly generated DPoP Proof.

## Examples

### Authorization flow from a backend client

The client starts by resolving Authorization Server Metadata as described [before](#server-metadata).

In order to build the authorization URL, the client performs a [PAR][RFC9126] + [PKCE][RFC7636] request towards the AS, using the `pushed_authorization_request_endpoint` obtained from the PDS's [Authorization Server Metadata][RFC8414]. Here is an example of such a request:

```http
POST https://bsky.social/oauth/par
Content-Type: application/x-www-form-urlencoded

response_type=code
&code_challenge=K2-ltc83acc4h0c9w6ESC_rEMTJ3bww-uCHaoeK1t8U
&code_challenge_method=S256
&client_id=bsky.app
&state=duk681S8n00GsJpe7n9boxdzen
&redirect_uri=https://bsky.app/my-app/oauth-callback
&scope=scope_a scope_b scope_c
&login_hint=did:plc:123
&client_assertion_type=urn%3Aietf%3Aparams%3Aoauth%3Aclient-assertion-type%3Ajwt-bearer
&client_assertion=<SELF_SIGNED_JWT>
```

Upon reception of this request, the AS will load the [client metadata](#client-metadata) and verify the [authorization request](#authorization-request). The AS will respond with a request URI:

```http-response
HTTP/1.1 201 Created
Cache-Control: no-cache, no-store
Content-Type: application/json

{
  "request_uri": "urn:ietf:params:oauth:request_uri:bwc4JK-ESC0w8acc191e-Y1LTC2",
  "expires_in": 90
}
```

The client will then redirect the user to the authorize endpoint using the `request_uri` obtained from the previous step. The AS will verify that the `request_uri` is still valid. Here is what the authorization URL will look like:

```url
https://bsky.social/oauth/authorize?client_id=bsky.app&request_uri=urn%3Aietf%3Aparams%3Aoauth%3Arequest_uri%3Abwc4JK-ESC0w8acc191e-Y1LTC2
```

The AS will then authenticate the user and ask them to approve the request. Silent sign-on will only be used if the user already has a session on the AS with the same DID as the one defined in the `login_hint` parameter of the PAR request.

Upon successful authorization by the user, the AS will issue an authorization code and redirect the user back to the client's `redirect_uri`.

The client will use that code (along with [PKCE][RFC7636], DPoP ([RFC9449]) & JWT for Assertion Framework protocol ([RFC7523]) tokens), to contact the `/token` endpoint on the AS. The AS will make all necessary checks (JWT, PKCE, DPoP key <> client assertion key, request expiration, etc.) to ensure that the request is valid. This will be enforced by the AS. Here is an example of such a request:

```http
POST https://bsky.social/oauth/token
Content-Type: application/x-www-form-urlencoded
DPoP: <DPOP_PROOF_JWT>

grant_type=authorization_code
&code=<AUTHORIZATION_CODE>
&code_verifier=dBjftJeZ4CVP-mB92K27uhbUJU1p1r_wW1gFWFOEjXk
&client_id=bsky.app
&redirect_uri=https://bsky.app/my-app/oauth-callback
&client_assertion_type=urn%3Aietf%3Aparams%3Aoauth%3Aclient-assertion-type%3Ajwt-bearer
&client_assertion=<SELF_SIGNED_JWT>
```

```http-response
HTTP/1.1 200 OK
Content-Type: application/json
Cache-Control: no-store

{
 "access_token": "Kz~8mXK1EalYznwH-LC-1fBAo.4Ljp~zsPE_NeO.gxU",
 "token_type": "DPoP",
 "expires_in": 2677,
 "refresh_token": "Q..Zkm29lexi8VnWg2zPW1x-tgGad0Ibc3s3EwM_Ni4-g"
}
```

The AS will bind those tokens to the `client_id` and the public key used to authenticate the client. The validity of the access token will be limited to a short period of time (e.g. 1 hour). The validity of the refresh token **must** be limited to a short period of time (e.g. 24 hours) or expire if they are not used in some amount of time (e.g. 24 hours). The AS **can** use longer pariods for authenticated clients but **must not** allow refresh tokens without a limited lifetime.

Refresh tokens will later be used in order to obtain new access tokens. These requests on the `token_endpoint` must be authenticated using the same method (JWT for Assertion Framework protocol ([RFC7523])). the same DPoP key must be presented in the `DPoP` header of the request.

### Authorization flow from a serverless browser app

In this mode, the browser will act as a public client. This is essentially the flow described in section 6.3 of [DRAFT-OAUTH-BROWSER-BASED-APPS]. The reason why we want to support the least secure option from the book is because we want to allow client developers to create simple serverless apps for the Atproto ecosystem.

The client starts by resolving Authorization Server Metadata as described [before](#server-metadata).

In order to build the authorization URL, the client performs a [PAR][RFC9126] + [PKCE][RFC7636] request towards the AS, using the `pushed_authorization_request_endpoint` obtained from the PDS's [Authorization Server Metadata][RFC8414]. Note that browser apps not being able to securely store a shared private key, they will not be able to use the `client_assertion` parameter. Here is an example of such a request:

```http
POST https://bsky.social/oauth/par
Content-Type: application/x-www-form-urlencoded

response_type=code
&code_challenge=K2-ltc83acc4h0c9w6ESC_rEMTJ3bww-uCHaoeK1t8U
&code_challenge_method=S256
&client_id=bsky.app
&state=duk681S8n00GsJpe7n9boxdzen
&redirect_uri=https://bsky.app/my-app/oauth-callback
&scope=scope_a scope_b scope_c
&login_hint=did:plc:123
```

Note that the client **must** bind the `state` parameter to the issuer of the authorization request. This is required to properly verify the `iss` parameter in the authorization response (see [RFC9207]).

```http-response
HTTP/1.1 201 Created
Cache-Control: no-cache, no-store
Content-Type: application/json

{
  "request_uri": "urn:ietf:params:oauth:request_uri:bwc4JK-ESC0w8acc191e-Y1LTC2",
  "expires_in": 90
}
```

The client will then build an authorization URL using the `authorize_endpoint` obtained from the PDS's Authorization Server Metadata. Here is an example of a URL generated by the client:

```url
https://bar.xzy/oauth2/authorize
  ?client_id=bsky.app
  &request_uri=urn%3Aietf%3Aparams%3Aoauth%3Arequest_uri%3Abwc4JK-ESC0w8acc191e-Y1LTC2
```

The browser will open that URL and the user will be redirected to the AS. The AS will flag this authorization request as "non-confidential". This will cause the following limitations to be applied:

- No silent sign-on will be allowed in this case (any `prompt=none` request will result in `error=login_required` errors)
- The user will be shown a confirmation [AUTH-UI] whether he already approved this client or not.
- The total lifetime of the tokens will be limited.
- The DPoP key must be a key never encountered before. This is done to prevent a malicious actor who managed to steal a DPoP key from a client to be able to use it during future sessions, at which point any vulnerability might have been fixed.

Upon successful authorization by the user, the AS will issue an authorization code and redirect the user back to the client's `redirect_uri` with a `code`.

The client will use that code (along with [PKCE][RFC7636]), to contact the `/token` endpoint on the AS.

```http
POST https://bsky.social/oauth/token
Content-Type: application/x-www-form-urlencoded
DPoP: <DPOP_PROOF_JWT>

grant_type=authorization_code
&code=<AUTHORIZATION_CODE>
&code_verifier=dBjftJeZ4CVP-mB92K27uhbUJU1p1r_wW1gFWFOEjXk
&client_id=bsky.app
&redirect_uri=https://bsky.app/my-app/oauth-callback
```

```http-response
HTTP/1.1 200 OK
Content-Type: application/json
Cache-Control: no-store

{
 "access_token": "Kz~8mXK1EalYznwH-LC-1fBAo.4Ljp~zsPE_NeO.gxU",
 "token_type": "DPoP",
 "expires_in": 2677,
 "refresh_token": "Q..Zkm29lexi8VnWg2zPW1x-tgGad0Ibc3s3EwM_Ni4-g"
}
```

The AS will make all necessary checks (PKCE, request expiration, etc.) to ensure that the request is valid and issue a new access token and a refresh token.

See [DRAFT-OAUTH-BROWSER-BASED-APPS] (section 6.3.2.5) and [DRAFT-OAUTH-SECURITY-TOPICS] (section 4.14.2), for more details on the restriction that **must** be applied to the refresh tokens.

## Security

### Refreshing tokens

When refreshing tokens, the AS will ensure that the public key used when initiating the session is still active within the client. If so, it will issue a new access token and rotated refresh token and return them to the client. If not, it will invalidate the refresh token and respond with an error.

### Session invalidation

If, at any point, a public key that was used by a client to authenticate itself is not exposed by that client anymore, all sessions linked to the missing key(s) must be invalidated. This allows clients to proactively invalidate all sessions linked to a compromised key.

### Impersonation

Since client names and logo can easily be spoofed, they will **not** be used in the [AUTH-UI] to avoid any misdirection while the user make their choice. The best (an only) way users nowadays know how to identify an internet actor is by its domain name. Their handle could also be used, but these are not as easy to identify as being authentic as domain names are.

### Server Side Request Forgery (SSRF)

The AS **must** ensure that any of the data it loads from the client (e.g. client metadata, `jwks_uri`, etc.) is not pointing to a private IP address or a local network. This is to prevent the client from being able to perform SSRF attacks on the AS.

## Frequently Asked Questions

### How can we experiment locally with OAuth without having to deploy client metadata ?

See the [Client metadata for local development](#client-metadata-for-local-development) section.

### How would a "headless" client (e.g. a CLI) authenticate?

In a CLI, the client can use the `localhost` authentication method described above, and trigger the authorization flow by opening a browser window. The CLI will need to run a local web server to receive the authorization code and exchange it for tokens. Since clients will not be able to authenticate themselves in this scenario, the token lifetime will be limited to 2 month (with at least one refresh every 48 hours).

For clients that require long lived tokens, the AS would ideally have to implement device flow. This is not covered yet by this spec. Instead, a provider might decide to rely on the ability to create "API tokens".

### Can I use a PDS 's Authorization Server as OIDC Identity Provider ?

Yes. There is nothing in this spec that prevents the Authorization Server to be compatible with OpenID. Clients can even rely on this capability and dynamically customize the authorization request to contain the `openid` scope & `id_token` response type.

### `application_type` is part of OIDC, why is it part of this spec ?

The `application_type` client metadata claim is part of [OIDC-CLIENT-REGISTRATION] and not part of [Dynamic Client Registration Protocol][RFC7591]. While this spec does not require OIDC compatibility, that particular claim was added for the following reasons:

- [DRAFT-OAUTH-SECURITY-TOPICS] distinguishes security practices for native & web apps.
- [DRAFT-OAUTH-BROWSER-BASED-APPS] requires exact matching of the `redirect_uri` for web apps. This rules out the use of loopback redirect uris for web apps.
- Since AS' can be implemented to be OIDC compliant, it is important that all clients, including those that are not OIDC compliant, stay compatible with every AS. Since the default `application_type` is `web`, non OIDC compliant clients could be rejected by OIDC compliant AS, if they are using `redirect_uris` that are not allowed for `web` clients. For this reason, using Loopback or custom scheme redirect uris requires to specify `application_type` as `native`.

### Should websites allow users to login with ATPROTO ?

While technically possible it's not recommended at the current time. Clients attempting to implement such a scheme **MUST** take into account that the `sub` in the token response **cannot** be trusted without the did -> pds -> issuer resolution described in this document. Not doing this verification would allow a malicious actor to impersonate any user in the client.

## References

- [OAuth research](https://blueskyweb.notion.site/oauth2-research-762c333111ec4713b6727d3f1603d61c)
- [OAuth resources](https://blueskyweb.notion.site/OAuth-resources-87e8cd12da4f4c07896a7887a8e0b70c)
- [OAuth Support in Bluesky and AT Protocol](https://aaronparecki.com/2023/03/09/5/bluesky-and-oauth)
- [IETF Working group: Web Authorization Protocol](https://datatracker.ietf.org/wg/oauth/documents/)
- [Description of a Project](https://github.com/ewilderj/doap/wiki)
- [INDIE-AUTH] IndieAuth
- [JSON for Linking Data](https://json-ld.org/)
- [The Open Graph protocol](https://ogp.me/)
- [DID] Decentralized Identifiers (DIDs) v1.0
- [DID-WEB] `did:web` Method Specification
- [DID-PLC] `did:plc` Method Specification
- [OIDC-CORE] OpenID Connect Core 1.0
- [OIDC-CLIENT-REGISTRATION] OpenID Connect Dynamic Client Registration 1.0
- [RFC6749] OAuth 2.0 Authorization Framework
- [RFC7517] JSON Web Key (JWK)
- [RFC7519] JSON Web Token (JWT)
- [RFC7521] Assertion Framework
- [RFC7523] JWT for Assertion Framework
- [RFC7591] Dynamic Client Registration Protocol
- [RFC7592] Dynamic Client Registration Management Protocol
- [RFC7636] Proof Key for Code Exchange (PKCE)
- [RFC8414] Authorization Server Metadata (`/.well-known/oauth-authorization-server`)
- [RFC9101] JWT-Secured Authorization Request (JAR)
- [RFC9126] Pushed Authorization Requests (PAR)
- [RFC9207] OAuth 2.0 Authorization Server Issuer Identification
- [RFC9449] Demonstrating Proof of Possession (DPoP)
- [DRAFT-AUTHORIZATION-SERVER-DISCOVERY] Draft: Authorization Server Discovery
- [DRAFT-OAUTH-ATTESTATION-BASED-CLIENT-AUTH] Draft: Attestation-Based Client Authentication
- [DRAFT-OAUTH-BROWSER-BASED-APPS] Draft: Oauth browser based apps
- [DRAFT-OAUTH-FIRST-PARTY-APPS] Draft: OAuth 2.0 for First-Party Applications
- [DRAFT-OAUTH-SECURITY-TOPICS] Draft: Security Best Current Practice
- [DRAFT-OAUTH-V2-1] Draft: OAuth 2.1
- [W3C-WEBMANIFEST] Draft: Web Application Manifest
- [WEBCRYPTO] Web Crypto API

[ATPROTO]: https://atproto.com/ 'AT Protocol'
[AUTH-UI]: https://www.oauth.com/oauth2-servers/authorization/the-authorization-interface/ 'The Authorization Interface'
[DID]: https://www.w3.org/TR/did-core 'Decentralized Identifiers (DIDs) v1.0'
[DID-WEB]: https://w3c-ccg.github.io/did-method-web/ 'did:web Method Specification'
[DID-PLC]: https://web.plc.directory/spec/v0.1/did-plc 'did:plc Method Specification'
[INDIE-AUTH]: https://indieauth.spec.indieweb.org 'IndieAuth'
[DRAFT-AUTHORIZATION-SERVER-DISCOVERY]: https://datatracker.ietf.org/doc/html/draft-parecki-authorization-server-discovery-00 'Draft: Authorization Server Discovery'
[DRAFT-OAUTH-ATTESTATION-BASED-CLIENT-AUTH]: https://datatracker.ietf.org/doc/html/draft-ietf-oauth-attestation-based-client-auth-01 'Draft: Attestation-Based Client Authentication'
[DRAFT-OAUTH-BROWSER-BASED-APPS]: https://datatracker.ietf.org/doc/html/draft-ietf-oauth-browser-based-apps 'Draft: Oauth browser based apps'
[DRAFT-OAUTH-FIRST-PARTY-APPS]: https://datatracker.ietf.org/doc/html/draft-parecki-oauth-first-party-apps-00 'Draft: OAuth 2.0 for First-Party Applications'
[DRAFT-OAUTH-SECURITY-TOPICS]: https://datatracker.ietf.org/doc/html/draft-ietf-oauth-security-topics 'Draft: Security Best Current Practice'
[DRAFT-OAUTH-V2-1]: https://datatracker.ietf.org/doc/html/draft-ietf-oauth-v2-1-09 'Draft: OAuth 2.1'
[OIDC-CORE]: https://openid.net/specs/openid-connect-core-1_0.html 'OpenID Connect Core 1.0'
[OIDC-CLIENT-REGISTRATION]: https://openid.net/specs/openid-connect-registration-1_0.html 'OpenID Connect Dynamic Client Registration 1.0'
[PSL]: https://publicsuffix.org/ 'Public Suffix List'
[RFC7591]: https://datatracker.ietf.org/doc/html/rfc7591 'RFC7591: Dynamic Client Registration Protocol'
[RFC6749]: https://datatracker.ietf.org/doc/html/rfc6749 'RFC6749: OAuth 2.0 Authorization Framework'
[RFC7519]: https://datatracker.ietf.org/doc/html/rfc7519 'RFC7519: JSON Web Token'
[RFC7517]: https://datatracker.ietf.org/doc/html/rfc7517 'RFC7517: JSON Web Key'
[RFC7521]: https://datatracker.ietf.org/doc/html/rfc7521 'RFC7521: Assertion Framework'
[RFC7523]: https://datatracker.ietf.org/doc/html/rfc7523 'RFC7523: JWT for Assertion Framework'
[RFC7592]: https://datatracker.ietf.org/doc/html/rfc7592 'RFC7592: Dynamic Client Registration Management Protocol'
[RFC7636]: https://datatracker.ietf.org/doc/html/rfc7636 'RFC7636: Proof Key for Code Exchange'
[RFC8414]: https://datatracker.ietf.org/doc/html/rfc8414 'RFC8414: Authorization Server Metadata'
[RFC9126]: https://datatracker.ietf.org/doc/html/rfc9126 'RFC9126: Pushed Authorization Requests'
[RFC9101]: https://datatracker.ietf.org/doc/html/rfc9101 'The OAuth 2.0 Authorization Framework: JWT-Secured Authorization Request (JAR)'
[RFC9207]: https://datatracker.ietf.org/doc/html/rfc9207 'RFC9207: OAuth 2.0 Authorization Server Issuer Identification'
[RFC9449]: https://datatracker.ietf.org/doc/html/rfc9449 'RFC9449: Demonstrating Proof of Possession'
[W3C-WEBMANIFEST]: https://w3c.github.io/manifest/ 'Draft: Web Application Manifest'
[WEBCRYPTO]: https://developer.mozilla.org/en-US/docs/Web/API/Web_Crypto_API 'Web Crypto API'
