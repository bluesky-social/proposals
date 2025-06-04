# Proposal: Client assertion backend for browser-based applications

> This proposal was written by Devin Ivy and edited by Matthieu Sieben, on behalf of Bluesky. First posted in June 2025.

In its current form, ATProto's OAuth flavor requires app developers to make a choice: use a “confidential” or “public” client. This choice will impact the security properties of the OAuth session like its lifetime, the ability to use silent sign-in, etc.

Single-page web apps and native apps are applications that run and hold credentials on user devices. They typically don’t have a server component (besides a trivial one for serving static assets), making them de facto “public”. In order to become “confidential”, clients must be able to securely maintain and use private keys. As a result, they also must rely on a more complex server component, or "backend".

Native and browser-based apps often achieve this through a [Token-Mediating Backend](https://datatracker.ietf.org/doc/html/draft-ietf-oauth-browser-based-apps#section-6.2.1) (TMB) architecture. But implementing such a server can be quite complex, as the server will have to deal with session management, token storage, etc. This is especially true when combined with DPoP.

The purpose of this proposal is to outline a low-footprint method of making confidential browser-based clients. It does so by extending the [JWT Profile for OAuth 2.0 Client Authentication and Authorization Grants](https://datatracker.ietf.org/doc/html/rfc7523) – when used in combination with [Demonstrating Proof of Possession](https://datatracker.ietf.org/doc/html/rfc9449), which is already a part of the ATProto OAuth profile.

## Specification

When validating requests that perform client authentication using [JWT Profile for OAuth 2.0](https://datatracker.ietf.org/doc/html/rfc7523), such as the token or PAR endpoints, if the `client_assertion` JWT contains a `cnf` claim using the `jkt` method (JWK Thumbprint Confirmation Method) then it MUST be validated by the AS against the public key of the request’s DPoP proof.

In other words, client assertions may be bound to a public DPoP key.

## Upgrading a browser-based client

To upgrade a browser-based client from public to confidential, we’ll take three steps:

1. The client runs a web service backend (to be described) which contains private key material used for client authentication consistent with [RFC 7523](https://datatracker.ietf.org/doc/html/rfc7523), and reflects the public keys through the `jwks` or `jwks_uri` claims in their client metadata document.
2. When the client wants to make a request to the authorization server’s PAR or token endpoint during the authorization flow, it first obtains a client assertion by making a request to its backend containing a DPoP proof. The backend returns back a client assertion JWT consistent with [JWT Profile for OAuth 2.0 Client Authentication and Authorization Grants (RFC 7523)](https://datatracker.ietf.org/doc/html/rfc7523), bound to the DPoP public key using the `cnf` claim and `jkt` method.
3. The client sends this client assertion to the authorization server token endpoint, along with a DPoP proof using its same session key. Per the [specification](#specification) above, the authorization server will validate that the same DPoP key was used for both the client assertion and the token endpoint.

## Client assertion backend implementation details

Implementers of a client assertion backend and their consumers should consider the following:

- The client assertion backend should have an private endpoint to generate a DPoP-bound client assertion JWT consistent with [RFC 7523](https://datatracker.ietf.org/doc/html/rfc7523).
- The request should use the `POST` HTTP method and must contain a DPoP proof via the `DPoP` header which MUST be validated by the backend.
- The backend may require a DPoP nonce.
- The client assertion will contain the `cnf` claim containing the DPoP public key’s fingerprint as `jkt`.
- The endpoint should enforce CORS to ensure browser-originating requests only come from the origin(s) associated with the client.
- The endpoint should track usage of each end device via their DPoP public keys (creation date, last updated date, app version, etc.).
- The endpoint may refuse to respond with a client assertion, e.g. in case of suspected abuse of a given end device or compromised end client.
- The endpoint may require additional authentication.
- The endpoint’s response must not be cached.
- The client assertion backend may also be used to host the client metadata document.

### Example

Request

```
POST /oauth/client-assertion
Host: my.client.com
Origin: https://my.client.com
Content-Length: 0
DPoP: eyJhbGciOiJFUzI1NiIsInR5cCI6ImRwb3Arand0IiwiandrIjp7Imt0eSI6IkVDIiwiY3J2IjoiUC0yNTYiLCJ4IjoiOHhnUU55YmwxZUl0NFBGYzZXWXBDd0dBejdWQXM2OGVHNHpTZERQUVpTbyIsInkiOiJfRURBNVB2RUhPbDVVc05LWjE3cUd4WkhzQ3NnNTFWdTFOZFl1bVY0UWJNIn19.eyJpc3MiOiI8TXlEZXZpY2U-IiwiaWF0IjoxNzQ5MDIwNzM4LCJqdGkiOiJkaGhhcGN0Y3B5YiIsImh0bSI6IlBPU1QiLCJodHUiOiJodHRwczovL215LmNsaWVudC5jb20vb2F1dGgvY2xpZW50LWFzc2VydGlvbiJ9.7INkysVo70hZtMznmuqUSWS1kv4pyr2CAqPQqx8YdH-aK0AGoF8oSLa6cUI0V5FY_XmOlZWQK2qI0CpaS0kn-A
```

Response

```
Content-Length: 523
Content-Type: application/json; charset=utf-8
Cache-Control: no-store
Access-Control-Allow-Origin: https://my.client.com

{
  "client_id": "https://my.client.com/oauth-client-metadata.json",
  "client_assertion": "eyJhbGciOiJFUzI1NiJ9.eyJpc3MiOiJodHRwczovL215LmNsaWVudC5jb20vb2F1dGgtY2xpZW50LW1ldGFkYXRhLmpzb24iLCJzdWIiOiJodHRwczovL215LmNsaWVudC5jb20vb2F1dGgtY2xpZW50LW1ldGFkYXRhLmpzb24iLCJhdWQiOiJodHRwczovL2Jza3kuc29jaWFsIiwianRpIjoiZTBxOHp1YTJxdW4iLCJpYXQiOjE3NDkwMjE2MDksImNuZiI6eyJqa3QiOiIwN1FERUZRcVduV0ZCUmxCOFprN09idzFJMktabFhBb0ZqeGJicjd0Vm9BIn19.vgYiLDqYXeCGHxJdl_eYGpqeWfU3EPQh8Dv7BMBnh9OBGET0Sr2Wj_y7ViJsxMC_X2YeQZICIehQbHYG9wOMPw"
}
```

## Conclusion

This approach offers browser-based apps a clear path to becoming confidential clients, improving their security posture without sacrificing their overall app architecture or session management. Clients making use of DPoP-bound client assertions as outlined have the following properties:

- User sessions may be revoked en masse by rotating the keys advertised in the client’s metadata document.

- Individual sessions may be discontinued by the client assertion backend refusing to issue new assertions for a given DPoP key.

- Max session lifetimes or other arbitrary session lifetime rules can be enforced by the client assertion backend.

- By bounding the client authentication to the session DPoP key it greatly constrains the impact of having an open endpoint for providing client assertions (though the backend may also require other forms of authentication). You could not mint an assertion in one session then use it in another.

This essentially gives a client backend veto power over token issuance, without the token request actually having to pass through the backend service. Consider this in contrast to the token-mediating backend pattern, which acts more like a proxy with its own auth lifecycle. A simpler path to authoring confidential browser-based apps promotes a more secure network where developers have wide latitude to choose how and what they build.

## Resources

- [OAuth 2.0 for Browser-Based Applications](https://datatracker.ietf.org/doc/html/draft-ietf-oauth-browser-based-apps)
- [JWT Profile for OAuth 2.0 Client Authentication and Authorization Grants](https://datatracker.ietf.org/doc/html/rfc7523)
- [Demonstrating Proof of Possession](https://datatracker.ietf.org/doc/html/rfc9449)
- [OAuth 2.0 Attestation-Based Client Authentication](https://datatracker.ietf.org/doc/draft-ietf-oauth-attestation-based-client-auth/)
