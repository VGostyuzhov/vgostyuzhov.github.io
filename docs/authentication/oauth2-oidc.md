# OAuth 2.0 & OpenID Connect (OIDC)

OAuth 2.0 delegates access to resources. OpenID Connect adds authentication statements for a client. A resource server must not treat an arbitrary OAuth access token as proof of a human login, or accept an ID token in place of its own API access token.

## Roles and Token Purposes

| Role or artifact | Purpose |
|---|---|
| Resource owner | Entity granting access; often a user, absent from a client-credentials interaction |
| Client | Application requesting access; public clients cannot keep a client secret confidential |
| Authorization server / OpenID Provider | Issues access credentials; acts as an identity provider when OIDC is used |
| Resource server | API that validates access credentials and enforces authorization |
| Access token | Intended for a resource server; may be opaque or a JWT |
| ID token | Authentication statement intended for the OIDC client; identifies issuer and subject |
| Refresh token | Credential used at the token endpoint to obtain new access tokens; never an API credential |

OIDC requests include the `openid` scope. Additional scopes such as `profile` and `email` request claims; an ID token need not contain a full user profile. Map local identities by the issuer and subject pair, not an email address alone.

!!! info "External Resources"
    - [OAuth 2.0 Roles and Grants](https://www.rfc-editor.org/rfc/rfc6749.html#section-1) (IETF)
    - [OpenID Connect ID Tokens](https://openid.net/specs/openid-connect-core-1_0.html#IDToken) (OpenID Foundation)

## Authorization Code Flow with PKCE

This flow assumes HTTPS and a pre-registered client and redirect URI. Use an established OAuth/OIDC library for request binding and validation.

1. The client creates transaction-specific `state`, a PKCE verifier, and an OIDC `nonce` when used. Bind them to the initiating browser interaction; derive the S256 challenge from the verifier.
2. The client redirects the browser to the authorization endpoint with client ID, exact redirect URI, scopes, state, challenge, and nonce when used. This **front channel** is visible to the browser.
3. The provider authenticates the user and obtains any required authorization. It redirects back with a short-lived authorization code and state, not an API token in the URL.
4. The client validates the returned transaction binding and issuer context. It exchanges the code and verifier at the token endpoint, using client authentication if it is a confidential client.
5. The provider checks the code, client, redirect URI, and PKCE proof, then issues tokens. This exchange is a **back-channel** server request for a backend client; a SPA performs it from the browser.
6. The OIDC client validates its ID token, then creates an application session if appropriate. The API independently validates the access token and authorizes operations.

**Why PKCE works:** The client retains a fresh, unpredictable verifier and sends its S256 challenge, `BASE64URL(SHA256(ASCII(verifier)))`, in the authorization request. The authorization server binds that challenge to the issued code. At redemption, the client supplies the verifier; the server recomputes the challenge and compares it with the stored value. A thief who intercepts only the code and public challenge cannot redeem the code without the verifier; SHA-256 does not make recovery of a suitably random verifier practical. This does not protect a client whose verifier or execution environment is also compromised.

**PKCE policy:** RFC 9700 requires public clients to use PKCE and recommends it for confidential clients. Use S256. PKCE binds the code exchange to the initiating client transaction; it is not client authentication.

**Binding controls:** `state` correlates the response with the browser transaction. `nonce`, when sent, must match the ID token claim. Do not reuse either value across transactions. Libraries can use supported PKCE/nonce-based CSRF defenses under the specification's conditions; do not omit request binding based on an incomplete custom implementation.

**Failure example:** A callback code obtained for another browser transaction must not establish a session. Reject incorrect state/issuer binding, failed PKCE proof, or an incorrect ID token nonce rather than trying a different flow.

!!! info "External Resources"
    - [OAuth Security Best Current Practice](https://www.rfc-editor.org/rfc/rfc9700.html#section-2.1.1) (IETF)
    - [PKCE S256](https://www.rfc-editor.org/rfc/rfc7636.html#section-4) (IETF)

## Validation and Token Lifecycle

- **Redirect URI:** Apply exact matching to registered values, with only the specification's narrowly defined exceptions; prohibit open redirects at the client.
- **ID token:** Validate expected issuer, audience/client ID, signature and algorithm under the library's profile, expiry, and nonce when sent. Apply additional `azp` checks when required. See [JWT validation](jwt.md).
- **Access token:** The API checks intended audience, scope and policy. Use trusted local JWT validation or authorization-server introspection according to the token format.
- **Refresh token:** For public clients, use rotation with reuse detection or sender constraints. Revoke affected token grants when compromise is detected.
- **Transport and logs:** Use HTTPS; avoid credentials in URLs and redact codes and tokens from logs.
- **Provider outages:** New logins and refreshes can fail. Existing application sessions or locally verified access tokens may continue until policy, expiry, or key availability prevents it.

!!! info "External Resources"
    - [OIDC ID Token Validation](https://openid.net/specs/openid-connect-core-1_0.html#IDTokenValidation) (OpenID Foundation)
    - [Refresh Token Protection](https://www.rfc-editor.org/rfc/rfc9700.html#section-4.14) (IETF)

## Other Grants and Deployment Choices

**Client credentials** authenticates a confidential client for machine-to-machine access without a user. Restrict its audiences and scopes; its token does not identify a logged-in human.

Avoid implicit access-token delivery. RFC 9700 also prohibits the resource-owner password credentials grant. Native apps and browser apps need client-specific guidance; putting a secret in distributed application code does not make a client confidential.

A backend can retain API tokens and expose an opaque session cookie to the browser. A browser-held access token supports direct API calls but increases exposure to injected JavaScript. Evaluate the trust boundaries, CSRF controls, and recovery requirements rather than prescribing JWTs for every application.

!!! info "External Resources"
    - [Client Credentials Grant](https://www.rfc-editor.org/rfc/rfc6749.html#section-4.4) (IETF)
    - [OAuth Grant Security Recommendations](https://www.rfc-editor.org/rfc/rfc9700.html#section-2) (IETF)
