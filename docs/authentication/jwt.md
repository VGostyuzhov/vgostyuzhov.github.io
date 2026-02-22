# JSON Web Tokens (JWT)

JWT is an open standard (RFC 7519) for representing claims between two parties. It is widely used for stateless authentication in modern web applications and microservices.

## Structure of a JWT

A JWT consists of three parts separated by dots (`.`):

1.  **Header**: Contains metadata (e.g., algorithm used for signing).
2.  **Payload**: Contains the "claims" (data like user ID, expiration, roles).
3.  **Signature**: Created by hashing the header and payload with a secret key or private key.

Format: `Base64URL(Header).Base64URL(Payload).Signature`

## How it Works (Stateless Auth)

1.  **Login**: User authenticates; server generates a JWT signed with a secret.
2.  **Storage**: Client stores the JWT (e.g., in a cookie or memory).
3.  **Authorization**: Client sends the JWT in the `Authorization: Bearer <token>` header.
4.  **Verification**: Server verifies the signature using its secret. If valid, the server trusts the payload without checking a database.

## Pros and Cons

| Pros | Cons |
|------|------|
| **Stateless**: No need for a session store on the server; easy to scale horizontally. | **Revocation is Hard**: Once issued, a token is valid until it expires (unless a blacklist/revocation list is used). |
| **Mobile Friendly**: Works well in environments where cookies are difficult to manage. | **Payload Size**: Large payloads increase request size. |
| **Decoupled**: Authentication can be handled by a separate identity provider. | **Data Stale**: If user permissions change, the token remains valid with old permissions. |

## Threats and Mitigations

| Threat | Mitigation |
|--------|------------|
| **`alg: none` Attack** | Explicitly reject tokens where the algorithm is set to `none`. |
| **Weak Signing Key** | Use strong, randomly generated secrets (HMAC) or RSA/ECDSA keys. |
| **Token Theft (XSS)** | Store tokens in `HttpOnly` cookies if possible, or use very short-lived access tokens. |
| **Information Leakage** | Never put sensitive info (passwords, PII) in the payload, as it is only Base64 encoded, not encrypted. |
| **Replay Attack** | Always validate the `exp` (expiration) and `jti` (unique ID) claims. |

## Best Practices

-   **Short-lived Access Tokens**: Use tokens that expire in minutes.
-   **Refresh Tokens**: Use long-lived, opaque refresh tokens to get new access tokens.
-   **Signature Validation**: Always validate `iss` (issuer) and `aud` (audience) claims.
-   **Prefer Asymmetric Signing**: Use RS256 (RSA) or ES256 (ECDSA) so only the IDP can sign, but any service can verify with a public key.
