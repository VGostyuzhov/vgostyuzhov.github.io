# Web Security Fundamentals

Browser security depends on origin isolation, request credentials, safe handling of untrusted content, and server-side authorization. Learn the browser model here, then study [Web Vulnerabilities](vulnerabilities.md) and [API & Microservice Security](tools.md).

## Browser Security Model

### Same-Origin Policy

An **origin** is the triple scheme, host, and effective port. Same-Origin Policy (SOP) restricts cross-origin DOM access and reading many network responses. It does not prevent all cross-origin requests, embedding, or state changes.

| URL | Same origin as `https://bank.example`? | Reason |
|---|---|---|
| `https://bank.example:443` | Yes | Same effective HTTPS port |
| `http://bank.example` | No | Different scheme |
| `https://bank.example:8443` | No | Different port |
| `https://api.bank.example` | No | Different host |

**Site versus origin:** A schemeful site uses the scheme and registrable domain. HTTPS subdomains can be same-site while cross-origin. This distinction matters for SameSite cookies and for applications that trust sibling subdomains.

| Operation | Browser behavior | Security implication |
|---|---|---|
| Read another origin's DOM | Generally blocked | Embedding an iframe does not grant its parent access |
| Send a cross-origin fetch | May require preflight; response access requires CORS approval | SOP/CORS do not replace server authorization or CSRF checks |
| Load a classic script from another origin | Can execute with the embedding page's privileges | Trust the script supplier; consider CSP and Subresource Integrity for suitable static assets |
| Exchange `postMessage` events | Exchanges data without granting general DOM access | Check message origin, source, and data; use a specific target origin |
| Use localStorage or IndexedDB | Normally separated by origin, with additional browser partitioning policies | Same-origin injected script can access exposed application data |
| Send a cookie | Uses host/domain/path and cookie policy, not origin-port isolation | Do not place mutually untrusted cookie-using applications on separate ports of one host |

**Additional isolation:** `Cross-Origin-Opener-Policy: same-origin` separates browsing context groups, while `Cross-Origin-Embedder-Policy: require-corp` requires appropriate resource opt-in. Together they can enable cross-origin isolation features. They are additional isolation controls, not proof that a page cannot leak data; test effects on embeds, popups, and authentication flows.

!!! info "External Resources"
    - [Same-Origin Policy](https://developer.mozilla.org/en-US/docs/Web/Security/Same-origin_policy) (Mozilla)
    - [Cookie Isolation Limits](https://www.rfc-editor.org/rfc/rfc6265.html#section-8.5) (IETF)
    - [Cross-Origin Isolation](https://web.dev/articles/coop-coep) (Google)

## CORS & CSP

### CORS: Controlled Response Sharing

CORS lets a target server authorize browser JavaScript at another origin to read a response. It does not authenticate the caller, restrict non-browser clients, or by itself prevent CSRF.

| Response header | Meaning |
|---|---|
| `Access-Control-Allow-Origin` | One serialized origin or `*`; not a comma-separated list |
| `Access-Control-Allow-Credentials: true` | Allows sharing with a credentialed request; requires a specific allowed origin |
| `Access-Control-Allow-Methods` | Methods approved by a preflight response |
| `Access-Control-Allow-Headers` | Request headers approved by preflight, such as Content-Type or Authorization |
| `Access-Control-Max-Age` | Preflight cache lifetime, subject to browser limits |
| `Vary: Origin` | Separates cached representations when the response varies by Origin |

A **CORS-safelisted request** uses GET, HEAD, or POST and only safelisted headers/values. Safelisted Content-Type essences are `text/plain`, `application/x-www-form-urlencoded`, and `multipart/form-data`, with additional header-value restrictions. `application/json`, an explicit Authorization header, and methods such as PUT normally trigger preflight. A safelisted request can reach the server even when JavaScript cannot read the response.

**Worked example:** JavaScript at `https://app.example` posts JSON to `https://api.example`. Assume the browser has a matching API cookie and the application deliberately uses cookie-authenticated cross-origin requests.

```javascript
fetch("https://api.example/orders", {
  method: "POST",
  credentials: "include",
  headers: { "Content-Type": "application/json", "X-CSRF-Token": csrfToken },
  body: JSON.stringify({ productId: "demo-product", quantity: 1 })
});
```

The browser first sends a preflight without the application's session cookie. The server can answer only for an allowed origin, method, and header set:

```http
OPTIONS /orders HTTP/1.1
Host: api.example
Origin: https://app.example
Access-Control-Request-Method: POST
Access-Control-Request-Headers: content-type,x-csrf-token
```

```http
HTTP/1.1 204 No Content
Access-Control-Allow-Origin: https://app.example
Access-Control-Allow-Credentials: true
Access-Control-Allow-Methods: POST
Access-Control-Allow-Headers: Content-Type, X-CSRF-Token
Vary: Origin
```

The actual response also needs the allowed-origin and credentials headers for JavaScript to read it. The POST still needs session validation, CSRF validation, and authorization. If preflight rejects a requested method/header, the browser does not send the POST. Preflight can be cached. Allowing credentials does not override cookie scope, SameSite, or third-party-cookie policy.

For bearer-header requests, explicitly allow `Authorization` in preflight; do not confuse that approval with the fetch credentials mode. Never reflect arbitrary Origin values. `Vary: Origin` addresses cache variation, while appropriate private/no-store policy and correct cache keys protect personalized responses.

!!! info "External Resources"
    - [Fetch CORS Protocol and Safelisted Headers](https://fetch.spec.whatwg.org/#http-cors-protocol) (WHATWG)
    - [CSRF Prevention](https://cheatsheetseries.owasp.org/cheatsheets/Cross-Site_Request_Forgery_Prevention_Cheat_Sheet.html) (OWASP)

### CSP: Restricting Resource Loads and Script Execution

Content Security Policy is defense in depth against script execution and unwanted resource loading. Fix unsafe data insertion first; CSP does not make arbitrary HTML or JavaScript injection safe.

```http
Content-Security-Policy: default-src 'self'; script-src 'nonce-<fresh-random-value>'; object-src 'none'; base-uri 'none'; frame-ancestors 'self'
```

This is an illustrative policy, not a universal deployment configuration. Generate a cryptographically unpredictable nonce for each response and place the matching value on trusted script elements. Never deploy the literal placeholder or a fixed nonce. Hash policies suit stable script content. Test all required resources and consider a strict policy using nonces/hashes and supported `strict-dynamic` behavior.

| Directive | Purpose |
|---|---|
| `default-src` | Fallback for fetch directives that lack a specific value |
| `script-src` | Controls script sources and nonce/hash authorization |
| `style-src` | Controls style sources; evaluate nonces/hashes before allowing inline styles |
| `object-src 'none'` | Disables object/embed plugin content |
| `base-uri 'none'` | Prevents injected base elements from changing relative URL interpretation |
| `frame-ancestors` | Restricts which pages can embed this document; not covered by default-src |
| `connect-src` | Restricts script-initiated connection destinations |

Roll out with `Content-Security-Policy-Report-Only` and a configured reporting endpoint. Inspect violations, remove unsafe dependencies, and then enforce the tested policy. Report-only does not block attacks. Pair CSP with appropriate MIME handling and `X-Content-Type-Options: nosniff`; avoid treating scanner scores as a complete security assessment.

!!! info "External Resources"
    - [Content Security Policy Guidance](https://cheatsheetseries.owasp.org/cheatsheets/Content_Security_Policy_Cheat_Sheet.html) (OWASP)
    - [CSP Header and Directives](https://developer.mozilla.org/en-US/docs/Web/HTTP/CSP) (Mozilla)

## Secure Cookies & Storage

### Secure Cookies

| Attribute | Purpose | Limitation |
|---|---|---|
| `Secure` | Restricts delivery to secure transport | Does not prevent a compromised endpoint from using the credential |
| `HttpOnly` | Prevents direct JavaScript reads of the cookie | XSS can still perform authenticated actions |
| `SameSite` | Controls cross-site delivery | Lax permits safe top-level navigations; same-site does not mean same-origin |
| `Expires` / `Max-Age` | Controls browser retention | Server-side session expiry/revocation must be enforced separately |
| `Domain` | Extends host matching when supplied | Omit unless sharing is necessary; sharing expands exposure |
| `Path` | Filters cookie delivery by URL path | Not a security boundary against same-origin scripts |
| `__Host-` prefix | Requires Secure, no Domain, and `Path=/` in supporting browsers | Does not isolate ports |

```http
Set-Cookie: __Host-session=<unpredictable-id>; Secure; HttpOnly; SameSite=Lax; Path=/; Max-Age=1800
```

Select SameSite policy for the actual login and embedding flows; legitimate cross-site delivery may need `SameSite=None; Secure` and explicit CSRF controls. Regenerate identifiers on login/privilege changes, invalidate them on logout, and enforce server-side idle and absolute timeouts. See [Sessions & Cookies](../authentication/sessions.md).

Use HTTPS consistently and deploy HSTS after verifying HTTPS availability. Preloading has domain-wide operational consequences and requires deliberate deployment planning.

!!! info "External Resources"
    - [Using HTTP Cookies](https://developer.mozilla.org/en-US/docs/Web/HTTP/Cookies) (Mozilla)
    - [Session Management](https://cheatsheetseries.owasp.org/cheatsheets/Session_Management_Cheat_Sheet.html) (OWASP)

### Client-Side Storage Options

| Store | Typical lifetime | Access and security properties |
|---|---|---|
| localStorage | Until cleared/evicted | Synchronous JavaScript access; exposed to same-origin XSS |
| sessionStorage | Page session, normally per tab | JavaScript access; tab scoping does not prevent XSS |
| IndexedDB | Until cleared/evicted | Asynchronous structured storage; API complexity provides no XSS defense |
| Cache Storage | Until cleared/evicted | Available to Window and Worker contexts in supported secure contexts; not private to service workers |
| Cookies | Configurable | Browser-managed request delivery; JavaScript reads can be restricted by HttpOnly |

Quotas and partitioning vary by browser and context. Do not use API capacity or asynchronous access as evidence of confidentiality. Encryption at rest does not protect data from injected script if that script can access the decryption key or authorized operations.

**Storage decision:** Prefer keeping API credentials on a backend when feasible. If credentials are in HttpOnly cookies, provide CSRF defenses; if JavaScript can access a bearer token, XSS can obtain or use it. In-memory storage reduces persistence but does not make a compromised runtime trustworthy. Store only data needed by the client and clear obsolete sensitive state.

**Failure example:** Moving a token from localStorage to IndexedDB does not prevent injected JavaScript from reading it. Moving it to an HttpOnly cookie prevents that direct read, but the same script may still issue requests using the cookie.

!!! info "External Resources"
    - [CacheStorage Interface](https://www.w3.org/TR/service-workers/#cachestorage-interface) (W3C)
    - [Browser Storage and Session Security](https://cheatsheetseries.owasp.org/cheatsheets/Session_Management_Cheat_Sheet.html) (OWASP)
