# Session-Based Authentication

A server-side session associates an unpredictable identifier with authenticated state. The browser normally transports that identifier in a cookie. Use a framework's session implementation instead of designing identifiers or cookie cryptography independently.

## How it Works

1. The user submits credentials over HTTPS. The application verifies them and completes any required additional authentication.
2. The server creates a new session identifier using a cryptographically secure random generator. Store identity, creation time, activity time, and authentication strength on the server.
3. The server delivers the identifier through `Set-Cookie`. Rotate any pre-authentication identifier to prevent fixation.
4. The browser sends the cookie only on requests matching its scope and browser policy.
5. The server checks that the session exists, is unexpired and unrevoked, then authorizes the requested action separately.

A conceptual exchange uses a placeholder identifier; production values must be unpredictable and sufficiently long. OWASP recommends at least 64 bits of entropy; framework defaults or at least 128 random bits provide a practical margin for a custom identifier.

```http
HTTP/1.1 200 OK
Set-Cookie: __Host-session=<random-session-id>; Secure; HttpOnly; SameSite=Lax; Path=/
Cache-Control: no-store
```

```http
GET /account HTTP/1.1
Host: app.example
Cookie: __Host-session=<random-session-id>
```

A nonexistent or expired session must fail even if the cookie is otherwise well formed. Never log raw session identifiers.

!!! info "External Resources"
    - [Session ID Properties and Lifecycle](https://cheatsheetseries.owasp.org/cheatsheets/Session_Management_Cheat_Sheet.html) (OWASP)

## Cookie Scope and CSRF

Cookie matching differs from the browser's origin model. Cookies are not isolated by port. Host/domain and path determine delivery scope, while Secure and SameSite add transport and cross-site restrictions. Path is not an access-control boundary between mutually untrusted applications.

| Attribute | Benefit | Limitation |
|---|---|---|
| `Secure` | Restricts delivery to secure transport | Does not prevent endpoint compromise or replace TLS validation |
| `HttpOnly` | Prevents access through `document.cookie` | XSS can still issue authenticated requests and read application data |
| `SameSite=Lax` | Restricts many cross-site cookie sends | Allows top-level navigation using safe methods; never change state through GET |
| `SameSite=Strict` | Excludes cross-site delivery more broadly | Can interrupt legitimate login/navigation flows; same-site is not same-origin |
| `SameSite=None; Secure` | Supports required cross-site delivery | Requires explicit CSRF protection and remains subject to browser privacy policy |
| `__Host-` prefix | Requires Secure, no Domain, and `Path=/` in supporting browsers | Does not bind a cookie to a port |

For cookie-authenticated state changes, use the framework's CSRF protection: a synchronizer token or a signed, session-bound double-submit token, together with appropriate Origin/Referer or Fetch Metadata validation. SameSite is an additional layer. An attacker-controlled sibling subdomain can be same-site while being cross-origin.

!!! info "External Resources"
    - [CSRF Prevention and SameSite Limitations](https://cheatsheetseries.owasp.org/cheatsheets/Cross-Site_Request_Forgery_Prevention_Cheat_Sheet.html) (OWASP)
    - [Cookie Isolation Limits](https://www.rfc-editor.org/rfc/rfc6265.html#section-8.5) (IETF)

## Expiry, Rotation, and Revocation

Enforce expiry on the server; deleting a browser cookie is insufficient to revoke a stolen identifier.

- **Idle timeout:** End the session after a policy-defined period without activity.
- **Absolute timeout:** End the session after its maximum age, even if continuously active.
- **Rotation:** Replace the identifier after login and privilege changes; invalidate the previous identifier. Coordinate concurrent requests through the framework's supported mechanism.
- **Logout and compromise:** Invalidate the server record and clear the cookie with matching scope. Provide account-wide session revocation when required.
- **Sensitive actions:** Require recent or stronger authentication for credential changes, recovery, or high-impact operations.
- **Multiple servers:** Use shared or consistently replicated state, or a documented routing strategy. Revocation latency depends on caches, replicas, and ongoing requests.

**Failure example:** An attacker sets a known pre-login identifier in a victim's browser. If login retains it, the attacker can reuse it. Issuing a fresh identifier and invalidating the old one removes this fixation path; it does not prevent later theft of the new identifier.

!!! info "External Resources"
    - [Session Expiration, Reauthentication, and Logging](https://cheatsheetseries.owasp.org/cheatsheets/Session_Management_Cheat_Sheet.html) (OWASP)
