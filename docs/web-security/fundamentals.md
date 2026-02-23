# Web Security Fundamentals

## Browser Security Model
### Same-Origin Policy
**Same-Origin Policy (SOP)**
Browsers isolate content by _origin_ - the exact triple **scheme + host + port**. Code may read or modify a resource only when its origin matches the page's own.

---

**Origin examples**

|URL|Same origin as `https://bank.com`?|Why / why not|
|---|---|---|
|`https://bank.com`|Yes|Identical scheme/host/port|
|`http://bank.com`|No|Different scheme (`http`)|
|`https://bank.com:8443`|No|Port differs (`8443`)|
|`https://api.bank.com`|No|Host differs (`api.bank.com`)|

---

**How browsers enforce it**

- **DOM access** - Attempting `otherWindow.document` across origins raises a `SecurityError`.

- **Network responses** - `fetch`/XHR can send to any origin, but browsers hide response data unless the server supplies the proper CORS headers.

- **Storage scoping** - Cookies, LocalStorage, IndexedDB and Service Workers are all keyed by origin.

---

**Legitimate cross-origin patterns**

|Need|What makes it safe|
|---|---|
|Load libraries from a CDN|Outbound request is fine; SOP blocks reads. If you _must_ read, the CDN sends CORS headers.|
|Show a third-party iframe|Parent can't touch the iframe's DOM unless the two sides coordinate with `postMessage`.|
|Public API access|Server replies `Access-Control-Allow-Origin: https://your-app.example`.|
|Site analytics|Modern browsers restrict third-party cookies; `SameSite` and user consent become essential.|

---

**Attacker goals & SOP defenses**

|Goal|How SOP helps|What still works|
|---|---|---|
|Steal session cookies via JavaScript|`document.cookie` is inaccessible across origins|Network-layer theft if the site lacks HTTPS|
|Read internal API responses|Browser hides body without CORS|CSRF can _send_ state-changing requests because writes are allowed|
|DOM snooping via iframes|Cross-origin iframes get no DOM access|Click-jacking unless you set `X-Frame-Options` or CSP|

---

**Modern extra isolation**

- `Cross-Origin-Opener-Policy: same-origin` (COOP) - breaks shared `window` references.

- `Cross-Origin-Embedder-Policy: require-corp` (COEP) - forces embedded resources to opt-in with CORS/CORP.

- Combining COOP + COEP produces a _cross-origin isolated_ page, unlocking features like `SharedArrayBuffer` without risking data leaks.

---

**Quick checklist**

1. Serve everything over HTTPS and preload HSTS.

2. Default cookies to `SameSite=Lax; Secure`.

3. Validate the `Origin` header on state-changing endpoints to mitigate CSRF.

4. Scope CORS narrowly - never `Access-Control-Allow-Origin: *` for authenticated resources.

5. Adopt COOP/COEP on pages that need high-resolution timers or stronger isolation.

6. Watch DevTools for _"blocked by CORS policy"_ messages to confirm SOP is doing its job.

!!! info "External Resources"
    - [Same-Origin Policy - MDN Web Docs](https://developer.mozilla.org/en-US/docs/Web/Security/Same-origin_policy) (Mozilla)
    - [Cross-Origin Opener Policy - web.dev](https://web.dev/articles/coop-coep) (Google)
    - [Browser Security Handbook](https://code.google.com/archive/p/browsersec/wikis/Main.wiki) (Google)

### CORS & CSP
**CORS - Cross-Origin Resource Sharing**
Browsers block JavaScript from reading responses fetched across origins (Same-Origin Policy). CORS is a server-side opt-in that relaxes that rule.

_Key response headers_

|Header|Purpose|Sample value|
|---|---|---|
|`Access-Control-Allow-Origin`|Names the origin(s) allowed to read the response. Use a literal value, never `*` for authenticated endpoints.|`https://app.example`|
|`Access-Control-Allow-Credentials`|Permits cookies/Authorization headers on the request; must be `true` and **Origin cannot be `*`**.|`true`|
|`Access-Control-Allow-Methods`|Lists verbs accepted on the resource.|`GET, POST, OPTIONS`|
|`Access-Control-Allow-Headers`|Whitelists custom request headers.|`Authorization, X-Request-Id`|
|`Access-Control-Max-Age`|Caches pre-flight result in seconds.|`86400`|

_Simple vs pre-flighted_

- **Simple request**: `GET/POST/HEAD`, no custom headers, MIME is `text/*`, `application/x-www-form-urlencoded`, `multipart/form-data`, or `application/json`. Browser sends request immediately and enforces CORS on the response.

- **Pre-flighted**: Anything else triggers an **OPTIONS** probe; only if the server responds with the right `Access-Control-*` headers will the browser send the real request.

_Common pitfalls_

1. Wildcard `*` with credentials - blocked by browsers.

2. Reflecting `Origin` header blindly - opens everyone to your API.

3. Forgetting `Vary: Origin` - caches serve private data to the wrong site.

---

**CSP - Content Security Policy**
A defense-in-depth header that restricts where a page may load resources from and which inline behaviors are allowed, shutting down many XSS vectors.

_Typical CSP snippet_

```http
Content-Security-Policy: default-src 'self'; script-src 'self' https://cdn.example 'nonce-XYZ'; object-src 'none'; base-uri 'none'; frame-ancestors 'self';
```

_Directive highlights_

|Directive|Effect|Usual setting|
|---|---|---|
|`default-src`|Fallback for any fetch that lacks its own directive.|`'self'`|
|`script-src`|Governs `<script>` and inline JS. Nonces or hashes let you forbid `unsafe-inline`.|`'self' 'nonce-...'`|
|`style-src`|Controls CSS and inline styles. Often needs `'unsafe-inline'` unless you hash/nonce styles.|`'self'`|
|`object-src`|Disables Flash, Java, etc.|`'none'`|
|`img-src`, `font-src`, `connect-src`, ...|Fine-grained per resource type.|Depends on CDN/API usage|
|`base-uri`|Blocks attackers from changing `<base>` tag and rewriting relative links.|`'none'`|
|`frame-ancestors`|Replaces `X-Frame-Options`; defends against click-jacking.|`'self'`|

_Enforcement vs report-only_

- `Content-Security-Policy` blocks violations.

- `Content-Security-Policy-Report-Only` logs them to `report-uri` / `report-to` without breaking the page - ideal for safe rollout.

_Best-practice steps_

1. **Inventory resources** loaded by your site.

2. Start with **Report-Only**, collect violation reports, tighten sources.

3. Replace inline `<script>` with nonced or hashed versions; same for styles.

4. Eliminate legacy plugins: set `object-src 'none'`.

5. Combine CSP with **HTTP Response Headers** like `X-Content-Type-Options: nosniff` for layered protection.

---

**Quick comparison**

|Aspect|CORS|CSP|
|---|---|---|
|Protects|_Consumers_ of cross-origin responses (reads)|_Producers_ of a page's own resources (inbound loads, script execution)|
|Configured by|Response of **target** resource|Response of **HTML page**|
|Typical goal|Let front-end at `app.example` call `api.example` safely|Prevent XSS, data injection, click-jacking|
|Risk of mis-config|Leak private API data to any site|Break site features or leave XSS gaps|
|Learning order|Understand Same-Origin Policy then CORS|Understand XSS mechanics then CSP|

Use CORS to **open** controlled cross-origin channels, and CSP to **close** everything else attackers might abuse.

!!! info "External Resources"
    - [Cross-Origin Resource Sharing (CORS) - MDN Web Docs](https://developer.mozilla.org/en-US/docs/Web/HTTP/CORS) (Mozilla)
    - [Content Security Policy - MDN Web Docs](https://developer.mozilla.org/en-US/docs/Web/HTTP/CSP) (Mozilla)
    - [CSP Evaluator](https://csp-evaluator.withgoogle.com/) (Google)

### Secure Cookies & Storage
**Secure Cookies**

|Attribute / Pattern|Purpose|Best-practice value|
|---|---|---|
|`Secure`|Sends cookie only over HTTPS transport.|Always set.|
|`HttpOnly`|Hides cookie from JavaScript (`document.cookie`) to stop XSS exfil.|Always set on session/auth cookies.|
|`SameSite`|Controls cross-site _sending_ of the cookie - CSRF defense.|`Lax` by default; use `Strict` for highly sensitive cookies; if `None`, also add `Secure`.|
|`Expires` / `Max-Age`|Defines lifetime; omit to create a session cookie.|Keep lifetimes short; rotate tokens.|
|`Path`|Scopes cookie to part of site.|Narrow scope (e.g., `/account`).|
|`Domain`|Allows sub-domains to share cookies.|Omit when possible; fewer hosts means smaller attack surface.|
|`Priority`|Dictates eviction order under memory pressure.|`High` for auth cookies.|
|`__Host-` prefix|_Must_ be `Secure`, no `Domain`, `Path=/`. Prevents host-switching tricks.||
|`__Secure-` prefix|_Must_ be `Secure`. Helpful lint for CI/CD pipelines.||

Example auth cookie (single header line):

```http
Set-Cookie: __Host-session=abc123; Secure; HttpOnly; SameSite=Lax; Path=/; Max-Age=1800
```

_Threats & mitigations_

- **XSS** - mark cookies `HttpOnly` and eliminate injectable JS.

- **CSRF** - use `SameSite` plus server-side `Origin`/`CSRF-token` checks.

- **Downgrade/mixing** - enforce HTTPS with HSTS; without `Secure` the browser sends the cookie over HTTP.

- **Session fixation** - regenerate cookie after login; invalidate on logout.

---

**Client-Side Storage Options**

|Store|Capacity (approx.)|JS access|Persistence|Intended use|Security notes|
|---|---|---|---|---|---|
|`localStorage`|5-10 MB / origin|`window.localStorage`|Until cleared|Cache, prefs|Vulnerable to XSS; not sent with requests.|
|`sessionStorage`|Same as above|`window.sessionStorage`|Tab lifetime|Per-tab state|Isolated per tab; still XSS-exposed.|
|`IndexedDB`|Up to hundreds of MB|Asynchronous JS API|Until cleared|Structured data, offline apps|Less XSS-friendly (API awkward); still exposed if attacker runs code.|
|Cache Storage|Large, quota-based|Service Worker API|Until cleared|Offline assets|Accessible only inside Service Worker; still origin-scoped.|
|Cookies|4 KB each, ~180 per domain|With `document.cookie` unless `HttpOnly`|Configurable|Sessions, server flags|Automatically sent on every matching request; subject to CSRF.|

_Rules of thumb_

- Never put secrets (JWTs, OAuth refresh tokens) in **localStorage** or **sessionStorage** - they're one XSS away from theft.

- If your front-end _must_ store a bearer token, use a **`Secure`, `HttpOnly`, `SameSite` cookie** so it's at least shielded from XSS; pair with CSRF mitigations.

- Treat **IndexedDB** and **Cache Storage** like a local database, not a vault; encrypt data at rest if compromise would be sensitive.

- Periodically clear obsolete data - browsers may evict storage unpredictably when space runs low.

---

**Checklist**

- Mark every auth/session cookie: `Secure; HttpOnly; SameSite=Lax/Strict`.

- Prefer the `__Host-` prefix for single-domain, site-wide cookies.

- Use short lifetimes plus server-side rotation for session cookies.

- Store only non-sensitive user preferences in `localStorage`; anything confidential belongs on the server or in an encrypted, HttpOnly cookie.

- Combine strong CSP with input sanitization to lower XSS risk, reducing the attack surface on all client-side storage mechanisms.

!!! info "External Resources"
    - [Using HTTP cookies - MDN Web Docs](https://developer.mozilla.org/en-US/docs/Web/HTTP/Cookies) (Mozilla)
    - [SameSite cookies explained - web.dev](https://web.dev/articles/samesite-cookies-explained) (Google)
    - [Web Storage API - MDN Web Docs](https://developer.mozilla.org/en-US/docs/Web/API/Web_Storage_API) (Mozilla)
