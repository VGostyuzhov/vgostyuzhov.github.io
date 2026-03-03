# Session-Based Authentication

Session-based authentication is the "traditional" method where the server maintains state about the user's logged-in status.

## How it Works

1.  **Login**: User submits credentials to the server.
2.  **Creation**: Server verifies credentials and creates a session record in a data store (memory, database, or Redis).
3.  **Delivery**: Server sends a unique **Session ID** back to the client via a `Set-Cookie` header.
4.  **Persistence**: The browser automatically includes this cookie in every subsequent request to the same domain.
5.  **Verification**: Server looks up the Session ID in its store to identify the user.

## Security Attributes for Cookies

To secure session cookies, specific attributes must be used:

-   `HttpOnly`: Prevents JavaScript from accessing the cookie (mitigates XSS-based session theft).
-   `Secure`: Ensures the cookie is only sent over HTTPS.
-   `SameSite`: (Lax/Strict) Prevents the cookie from being sent in cross-site requests (mitigates CSRF).
-   `__Host-` Prefix: Hardens the cookie by requiring it to be Secure, have no Domain attribute, and be scoped to the root path.

## Pros and Cons

| Pros | Cons |
|------|------|
| **Instant Revocation**: Deleting the session on the server immediately logs the user out. | **Scalability**: Requires a shared session store (like Redis) for multi-node deployments. |
| **Simplicity**: Most web frameworks handle sessions out-of-the-box. | **CSRF Risk**: Browsers auto-send cookies, making apps vulnerable to CSRF if not properly protected. |
| **Small Payloads**: Only a small ID is sent, not the full user profile. | **Stateful**: Server must remember every active user. |

## Threats and Mitigations

| Threat | Mitigation |
|--------|------------|
| **Session Hijacking (XSS)** | Use `HttpOnly` flag and strong CSP. |
| **Cross-Site Request Forgery (CSRF)** | Use `SameSite=Lax/Strict` and anti-CSRF tokens. |
| **Session Fixation** | Regenerate the Session ID immediately after a successful login. |
| **Man-in-the-Middle (MitM)** | Use `Secure` flag and enforce HSTS. |
| **Brute Force** | Implement rate limiting on the login endpoint. |
