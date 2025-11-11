## Browser Security Model
### Same-Origin Policy
**Same-Origin Policy (SOP)**  
Browsers isolate content by _origin_—the exact triple **scheme + host + port**. Code may read or modify a resource only when its origin matches the page’s own.

---

**Origin examples**

|URL|Same origin as `https://bank.com`?|Why / why not|
|---|---|---|
|`https://bank.com`|✅ Yes|Identical scheme/host/port|
|`http://bank.com`|❌ No|Different scheme (`http`)|
|`https://bank.com:8443`|❌ No|Port differs (`8443`)|
|`https://api.bank.com`|❌ No|Host differs (`api.bank.com`)|

---

**How browsers enforce it**

- **DOM access** – Attempting `otherWindow.document` across origins raises a `SecurityError`.
    
- **Network responses** – `fetch`/XHR can send to any origin, but browsers hide response data unless the server supplies the proper CORS headers.
    
- **Storage scoping** – Cookies, LocalStorage, IndexedDB and Service Workers are all keyed by origin.
    

---

**Legitimate cross-origin patterns**

|Need|What makes it safe|
|---|---|
|Load libraries from a CDN|Outbound request is fine; SOP blocks reads. If you _must_ read, the CDN sends CORS headers.|
|Show a third-party iframe|Parent can’t touch the iframe’s DOM unless the two sides coordinate with `postMessage`.|
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

- `Cross-Origin-Opener-Policy: same-origin` (COOP) – breaks shared `window` references.
    
- `Cross-Origin-Embedder-Policy: require-corp` (COEP) – forces embedded resources to opt-in with CORS/CORP.
    
- Combining COOP + COEP produces a _cross-origin isolated_ page, unlocking features like `SharedArrayBuffer` without risking data leaks.
    

---

**Quick checklist**

1. Serve everything over HTTPS and preload HSTS.
    
2. Default cookies to `SameSite=Lax; Secure`.
    
3. Validate the `Origin` header on state-changing endpoints to mitigate CSRF.
    
4. Scope CORS narrowly—never `Access-Control-Allow-Origin: *` for authenticated resources.
    
5. Adopt COOP/COEP on pages that need high-resolution timers or stronger isolation.
    
6. Watch DevTools for _“blocked by CORS policy”_ messages to confirm SOP is doing its job.
### CORS & CSP
**CORS – Cross-Origin Resource Sharing**  
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

1. Wildcard `*` with credentials → blocked by browsers.
    
2. Reflecting `Origin` header blindly → opens everyone to your API.
    
3. Forgetting `Vary: Origin` → caches serve private data to the wrong site.
    

---

**CSP – Content Security Policy**  
A defense-in-depth header that restricts where a page may load resources from and which inline behaviors are allowed, shutting down many XSS vectors.

_Typical CSP snippet_

http

CopyEdit

`Content-Security-Policy:   default-src 'self';   script-src 'self' https://cdn.example 'nonce-XYZ';   object-src 'none';   base-uri 'none';   frame-ancestors 'self';`

_Directive highlights_

|Directive|Effect|Usual setting|
|---|---|---|
|`default-src`|Fallback for any fetch that lacks its own directive.|`'self'`|
|`script-src`|Governs `<script>` and inline JS. Nonces or hashes let you forbid `unsafe-inline`.|`'self' 'nonce-…'`|
|`style-src`|Controls CSS and inline styles. Often needs `'unsafe-inline'` unless you hash/nonce styles.|`'self'`|
|`object-src`|Disables Flash, Java, etc.|`'none'`|
|`img-src`, `font-src`, `connect-src`, …|Fine-grained per resource type.|Depends on CDN/API usage|
|`base-uri`|Blocks attackers from changing `<base>` tag and rewriting relative links.|`'none'`|
|`frame-ancestors`|Replaces `X-Frame-Options`; defends against click-jacking.|`'self'`|

_Enforcement vs report-only_

- `Content-Security-Policy` blocks violations.
    
- `Content-Security-Policy-Report-Only` logs them to `report-uri` / `report-to` without breaking the page—ideal for safe rollout.
    

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
|Protects|_Consumers_ of cross-origin responses (reads)|_Producers_ of a page’s own resources (inbound loads, script execution)|
|Configured by|Response of **target** resource|Response of **HTML page**|
|Typical goal|Let front-end at `app.example` call `api.example` safely|Prevent XSS, data injection, click-jacking|
|Risk of mis-config|Leak private API data to any site|Break site features or leave XSS gaps|
|Learning order|Understand Same-Origin Policy → CORS|Understand XSS mechanics → CSP|

Use CORS to **open** controlled cross-origin channels, and CSP to **close** everything else attackers might abuse.
### Secure Cookies & Storage
**Secure Cookies**

|Attribute / Pattern|Purpose|Best-practice value|
|---|---|---|
|`Secure`|Sends cookie only over HTTPS transport.|Always set.|
|`HttpOnly`|Hides cookie from JavaScript (`document.cookie`) to stop XSS exfil.|Always set on session/auth cookies.|
|`SameSite`|Controls cross-site _sending_ of the cookie → CSRF defense.|`Lax` by default; use `Strict` for highly sensitive cookies; if `None`, also add `Secure`.|
|`Expires` / `Max-Age`|Defines lifetime; omit to create a session cookie.|Keep lifetimes short; rotate tokens.|
|`Path`|Scopes cookie to part of site.|Narrow scope (e.g., `/account`).|
|`Domain`|Allows sub-domains to share cookies.|Omit when possible; fewer hosts means smaller attack surface.|
|`Priority`|Dictates eviction order under memory pressure.|`High` for auth cookies.|
|`__Host-` prefix|_Must_ be `Secure`, no `Domain`, `Path=/`. Prevents host-switching tricks.||
|`__Secure-` prefix|_Must_ be `Secure`. Helpful lint for CI/CD pipelines.||

Example auth cookie (single header line):

mathematica

CopyEdit

`Set-Cookie: __Host-session=abc123; Secure; HttpOnly; SameSite=Lax; Path=/; Max-Age=1800`

_Threats & mitigations_

- **XSS** → mark cookies `HttpOnly` and eliminate injectable JS.
    
- **CSRF** → use `SameSite` plus server-side `Origin`/`CSRF-token` checks.
    
- **Downgrade/mixing** → enforce HTTPS with HSTS; without `Secure` the browser sends the cookie over HTTP.
    
- **Session fixation** → regenerate cookie after login; invalidate on logout.
    

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

- Never put secrets (JWTs, OAuth refresh tokens) in **localStorage** or **sessionStorage**—they’re one XSS away from theft.
    
- If your front-end _must_ store a bearer token, use a **`Secure`, `HttpOnly`, `SameSite` cookie** so it’s at least shielded from XSS; pair with CSRF mitigations.
    
- Treat **IndexedDB** and **Cache Storage** like a local database, not a vault; encrypt data at rest if compromise would be sensitive.
    
- Periodically clear obsolete data—browsers may evict storage unpredictably when space runs low.
    

---

**Checklist**

- Mark every auth/session cookie: `Secure; HttpOnly; SameSite=Lax/Strict`.
    
- Prefer the `__Host-` prefix for single-domain, site-wide cookies.
    
- Use short lifetimes plus server-side rotation for session cookies.
    
- Store only non-sensitive user preferences in `localStorage`; anything confidential belongs on the server or in an encrypted, HttpOnly cookie.
    
- Combine strong CSP with input sanitization to lower XSS risk, reducing the attack surface on all client-side storage mechanisms.
## Common Web Vulnerabilities
**Common Web Vulnerabilities – Quick Reference**

---

**Cross‑Site Scripting (XSS)**  
- *What happens* Attacker injects JavaScript into a page viewed by other users.  
- *Impact* Session theft, DOM modification, phishing overlays, drive‑by malware.  
- *Typical root causes* Unsanitised user input echoed into HTML, attributes, or JS.  
- *Mitigations* Context‑aware output encoding (HTML vs JS vs URL), CSP `script-src` with nonces/hashes, HttpOnly cookies, strict input validation.

**SQL Injection (SQLi)**  
- *What happens* User‑supplied data alters back‑end SQL queries.  
- *Impact* Data exfiltration, authentication bypass, stored‑procedure execution.  
- *Typical root causes* String‑concatenated queries, insufficient parameterisation.  
- *Mitigations* Prepared statements / ORM placeholders, least‑privilege DB user, input whitelists, WAF filtering.

**Server‑Side Request Forgery (SSRF)**  
- *What happens* Server forced to fetch internal resources on attacker’s behalf.  
- *Impact* Cloud metadata theft, port scanning, internal service exploitation.  
- *Typical root causes* Unvalidated URLs in image fetchers, webhooks, PDF converters.  
- *Mitigations* Allow‑list outbound destinations, deny localhost/169.254.169.254, network egress filters, require signed URLs.

**Insecure Direct Object Reference (IDOR)**  
- *What happens* User manipulates IDs in URLs or JSON to access others’ data.  
- *Impact* Horizontal/vertical privilege escalation, data leakage.  
- *Typical root causes* Authorisation checks tied only to user‑supplied identifiers.  
- *Mitigations* Enforce access control on every request, use unpredictable IDs (UUID), avoid exposing internal keys.

**Cross‑Site Request Forgery (CSRF)**  
- *What happens* Logged‑in victim’s browser sends unwanted state‑changing request.  
- *Impact* Account takeover actions (change email, transfer funds).  
- *Typical root causes* Session cookies auto‑sent, no secondary validation.  
- *Mitigations* `SameSite=Lax/Strict`, double‑submit CSRF tokens, verify `Origin/Referer`, use REST patterns with access tokens instead of cookies.

**Deserialization of Untrusted Data**  
- *What happens* Crafted payload triggers gadget chain when server deserialises.  
- *Impact* Remote code execution, privilege escalation.  
- *Typical root causes* Java/PHP/Python object deserialisation without allow‑list.  
- *Mitigations* Use safe formats (JSON), enforce class allow‑list, upgrade libraries, isolate deserialisation in low‑privilege context.

**Command / Path Injection**  
- *What happens* User input inserted into shell command or file path.  
- *Impact* Arbitrary command execution, file overwrite or read.  
- *Typical root causes* `os.system`, back‑ticks, unsanitised file names in `exec` or `open`.  
- *Mitigations* Use parameter APIs (`execFile`), allow‑list file names, escape/quote args, run under least‑privilege account.

**Directory Traversal**  
- *What happens* `../../../etc/passwd` accesses files outside web root.  
- *Impact* Sensitive file disclosure, configuration theft.  
- *Typical root causes* Concatenating user path fragments, no canonicalisation.  
- *Mitigations* Resolve to absolute path then validate against base dir, strip `..`, serve files via mapped IDs.

**Broken Access Control**  
- *What happens* Endpoints rely on client‑side checks or omit authorisation layer.  
- *Impact* Privilege escalation, data leaks, admin‑only functions exposed.  
- *Typical root causes* RBAC not enforced on back‑end, hidden UI elements only.  
- *Mitigations* Centralised authZ middleware, deny‑by‑default, penetration tests.

**Insecure File Upload**  
- *What happens* Attacker uploads executable script or oversized file.  
- *Impact* Remote code execution, DoS storage exhaustion.  
- *Typical root causes* File type verified by extension only, no size limits.  
- *Mitigations* Content‑type/extension validation, virus scanning, rename + store outside web root, limit size and count.

**Prototype Pollution / Mass Assignment**  
- *What happens* Attacker overwrites JavaScript object prototype keys or model attributes.  
- *Impact* Add arbitrary properties, escalate privileges.  
- *Typical root causes* Merging user JSON into objects (`Object.assign(req.body, obj)`).  
- *Mitigations* `Object.create(null)`, deep‑clone allow‑list, framework‐level param filtering.

**Weak TLS Configuration**  
- *What happens* Old ciphers, TLS 1.0, missing HSTS.  
- *Impact* Downgrade attacks, session hijack.  
- *Typical root causes* Legacy servers, mis‑configured load balancers.  
- *Mitigations* TLS 1.3 (fallback TLS 1.2), modern cipher suites, HSTS preload, certificate pinning for mobile apps.

### XSS Variants
**XSS Variants – what changes, what stays the same**

| Variant             | Where payload is stored or reflected | How the script gets executed | Typical entry point examples | Distinctive challenges |
|---------------------|--------------------------------------|------------------------------|------------------------------|------------------------|
| **Reflected XSS**   | Lives only in the *request* (URL, form field, header); echoed in the *same* response. | Victim clicks attacker‑crafted link or submits form; browser immediately runs script. | `https://site.com/search?q=<script>` | Payload must convince user to visit link; easy to spot in access logs. |
| **Stored (Persistent) XSS** | Written into server‑side datastore (DB, comment, profile) and served to *all* future visitors. | Victim simply views the page that reads the stored data. | Forum post, support ticket, user bio field. | High impact, long‑lived; “sleeping” payloads evade scanning. |
| **DOM‑Based XSS**   | Injection never touches server; JavaScript on page reads attacker‑controlled value (e.g., `location.hash`) and writes it to the DOM with `innerHTML`. | Script runs purely in client’s browser as page logic executes. | `https://site.com/#<img src=x onerror=alert(1)>` | Server logs look clean; CSP with strict `script-src` helps most. |
| **Blind XSS**       | Like stored XSS but payload fires in a *different* application or admin interface the attacker never sees. | Attacker gets no direct feedback—must beacon outbound (e.g., webhook). | Contact‑us form reflected in help‑desk dashboard. | Hard to test manually; use out‑of‑band detection (`<script src=//burp.collab/>`). |
| **Self‑XSS**        | Victim pastes payload into their own browser console thinking it’s “hack tool” or “coupon generator”. | Runs in victim’s context; social‑engineering angle. | “Paste this in console for free skins!” | Browsers now warn users; educate rather than patch server. |
| **Mutation / Universal XSS (UXSS)** | Browser re‑parses a benign string into executable code (e.g., SVG, CSS escapes, innerHTML auto‑close). | Exploits parser quirks; may bypass usual sanitizers. | `<math><mtext></math><img src onerror=alert>` | Requires context‑aware sanitisation libs; keep dependencies patched. |

**Key constants**

* All variants exploit unsanitised content ending up in an *executable* context (HTML, attribute, URL, JS string, CSS).  
* Output‑encoding appropriate to the *sink* is the universal mitigation, backed by a tight `Content‑Security‑Policy`.  
* `HttpOnly` cookies cut off one common goal (session theft) but not others (defacement, phishing, keylogging).  

**Detection quick‑tips**

* Reflected → grep logs for `<script>` in query strings.  
* Stored / Blind → fuzz stateful inputs, monitor for callbacks.  
* DOM → run static analysis (`eslint-plugin-xss`) and dynamic scanners (`DOM Invader`).  

Patch the fundamentals—encode + CSP—then treat each variant’s quirks during testing.

### Injection Flaws
**Injection Flaws – shortcut guide**

| Injection type        | Where attacker sends payload                | Typical goal & impact                    | Classic payload snippet                    | Primary defences                                                   |
|-----------------------|---------------------------------------------|------------------------------------------|--------------------------------------------|--------------------------------------------------------------------|
| **SQL Injection (SQLi)**     | Query parameters, form fields, HTTP headers | Dump/alter database, bypass auth, RCE via xp_cmdshell. | `' OR 1=1--`                                   | Prepared statements / ORM bind variables, least‑priv DB user, input allow‑lists, WAF. |
| **NoSQL / Mongo Injection**  | JSON body or query string consumed by NoSQL driver | Read/modify NoSQL docs, escalate role.   | `{ "$ne": null }`                              | Use driver‑level parameter APIs, stringify user input, schema validation, limit operators. |
| **Command / Shell Injection**| Any value passed to OS shell (`exec`, `system`) | Execute arbitrary commands, elevate privileges. | `; rm -rf /`                                   | `execve()` with argument array, allow‑list commands, escape/quote, run as non‑root. |
| **LDAP Injection**           | Filters in directory search/bind      | Read/modify directory entries, bypass login. | `*)(uid=*)`                                    | Use parameterised LDAP APIs, escape RFC‑4515 special chars, least‑priv LDAP creds. |
| **Path / File Inclusion**    | File paths, `include()` arguments     | Read sensitive files, RCE via uploaded script. | `../../../../etc/passwd`                       | Resolve to canonical path then compare to base dir, store uploads outside web root, use allow‑list extensions. |
| **XML / XXE**                | XML documents parsed by server        | Read files, SSRF via external entities.  | `<!ENTITY x SYSTEM "file:///etc/passwd">`      | Disable external entities (`XXE`), prefer JSON, use parsing libraries with `secureProcessing`. |
| **Expression / Template Injection (SSTI)** | Unescaped template delimiters (`{{ }}` `#{}`) | Execute code on the server, leak secrets. | `{{7*7}}`                                       | Configure template engine sandbox, escape user input, strict template compilation modes. |
| **Log Injection**            | Values logged without sanitisation    | Forge log entries, inject newlines → log poisoning. | `User=admin\n[ERROR] …`                       | Replace control chars, structured logging (JSON), encode before log write. |

---

*Root causes*  
- Concatenating untrusted data into **code**, **queries**, **paths**, **commands**, or **log lines**.  
- Missing contextual escaping / validation.  
- Over‑privileged service accounts amplifying the blast radius.

*Universal mitigation playbook*  
1. **Parameterise, don’t concatenate** — use prepared statements or driver APIs.  
2. **Constrain identities** — DB, LDAP, or OS users should have minimum privileges.  
3. **Input allow‑lists** — reject unexpected types, formats, ranges early.  
4. **Contextual output encoding** — SQL quotes, shell escapes, LDAP escapes, etc.  
5. **Security‑focused code reviews + fuzzers** — automate detection of risky string ops.  
6. **Runtime protections** — WAF/IDS signatures for common injection payloads.  

Treat any spot where user data meets an interpreter as toxic until you’ve isolated or neutralised it.

### SSRF & Deserialization
## API & Microservice Security
### REST vs GraphQL Concerns
**REST vs GraphQL – key engineering & security concerns**

---

| Dimension | REST (resource‑oriented) | GraphQL (query‑oriented) | Why it matters |
|-----------|--------------------------|--------------------------|----------------|
| **Attack surface** | Fixed URIs & verbs mean predictable endpoints; WAF rules easier. | Single `/graphql` POST can express many operations; harder for firewalls & rate limits to understand intent. | Assessing risk & building rules. |
| **Over‑fetch / under‑fetch** | Client may pull entire resource even if only one field needed → wasted bandwidth. | Client specifies exact fields; risk of *very* heavy queries (“select *n \* deep*”). | Performance & DoS resilience. |
| **Query complexity limits** | Usually bounded by endpoint logic. | Must defend against nested, circular, or expensive queries; implement depth/complexity limits and timeouts. | Avoid CPU/DB exhaustion. |
| **Authorization model** | Per‑URI/per‑verb RBAC fits neatly: “`PATCH /users/42` requires `edit:user`”. | Field‑level auth needed (`{ user { ssn } }`); use schema directives or custom resolvers. | Prevent data leakage. |
| **Introspection exposure** | N/A – API docs separate (OpenAPI). | Introspection and `__schema` can reveal everything; disable or restrict in prod. | Recon for attackers. |
| **Error handling** | HTTP status codes (`404`, `401`) convey a lot. | Always `200 OK`; errors inside JSON require extra parsing, and status leaks less obvious. | Client simplicity, caching proxies. |
| **Caching** | URI + method good fit for CDNs (`GET /products/1`). | Response depends on *query string* (post body); need persisted‑query caching or APQ, complicates edge caching. | Latency & infra cost. |
| **Versioning** | New endpoint/URI (or `v2/` prefix). | Designed for *non‑breaking* evolution; add fields, deprecate old, no version bump. | Long‑term maintenance. |
| **Input validation** | Hand‑rolled per endpoint or OpenAPI validators. | Strong schema type checks, but watch for **GraphQL injection** via resolver args. | Reduce exploitable bugs. |
| **File uploads** | Multipart endpoints straightforward. | Need separate REST endpoint or GraphQL multipart spec (`apollo-upload`). | Extra work, more room for misconfig. |

---

**Threats unique to GraphQL**

* **Deeply nested queries** – attacker crafts `{a{b{c{d…}}}}` causing resolver N‑squared calls ⇒ Mitigate with query depth / cost analysis, timeouts, max nodes.  
* **Batching introspection** – enumerates hidden types/fields; disable `__schema` or require auth.  
* **Field‑level auth gaps** – forgetting to check one sensitive resolver leaks data.  
* **Alias + fragments DoS** – same expensive field under many aliases bypasses depth limit; count nodes/field repetitions too.

**REST‑specific pitfalls**

* **Unfiltered query params** – naive SQL concatenation in `GET /search?q=…`.  
* **Verb tunnelling** – proxies stripping `X‑HTTP‑Method‑Override` expose “hidden” verbs.  
* **Excessive endpoint sprawl** – forgotten legacy URIs linger unpatched.

---

**Safeguard checklist**

1. Enforce global rate limits; for GraphQL also add **per‑query complexity quotas**.  
2. Apply **auth in resolvers** (GraphQL) and middleware (REST); default‑deny.  
3. Validate input types and lengths server‑side even with GraphQL schema.  
4. Disable GraphQL introspection in production or require admin token.  
5. For REST, keep OpenAPI spec updated and use it to generate validators + tests.  
6. Cache safely: ETag/Cache‑Control on REST; persisted queries + allow‑list on GraphQL.  
7. Log with enough granularity: full URI and status for REST, query plus variables hash for GraphQL (watch PII).

### AuthN & AuthZ Patterns
**Authentication (AuthN) vs Authorization (AuthZ)**  
*AuthN* = “Who are you?” — process of proving identity.  
*AuthZ* = “What can you do?” — process of checking permissions **after** identity is known.

---

**Common AuthN patterns**

| Pattern / Mechanism | Where it’s used | Key strengths | Typical pitfalls |
|---------------------|-----------------|---------------|------------------|
| Session cookie + server‑side store | Web apps with browsers | Simple; invalidates on server; supports rotation | CSRF risk; horizontal scaling needs sticky sessions or shared cache |
| JWT bearer tokens | SPAs, mobile, microservices | Self‑contained claims, stateless | Long TTL = replay window; must validate `aud`, `iss`, `exp`; cannot revoke easily |
| OAuth 2.0 (authZ framework repurposed for AuthN) | “Login with …” social sign‑in | Delegates identity, MFA possible | Confusion between *auth code* & token; open redirects; PKCE required for public clients |
| OpenID Connect (OIDC) | Modern SSO (Okta, Auth0, Azure) | Adds identity layer on top of OAuth; interoperable | ID token integrity only as strong as issuer; mis‑configured scopes leak data |
| SAML 2.0 | Enterprise SSO, legacy federations | Mature, supports signed XML assertions | Verbose XML → signature wrapping attacks; clock skew issues |
| Mutual TLS (mTLS) | Service‑to‑service inside zero‑trust perimeter | Strong cryptographic identity, channel encryption | Certificate lifecycle & rotation overhead |
| API Keys | Simple server APIs & IoT | Easy to issue; low overhead | No user context, often hard‑coded, replayable without TLS |

---

**Common AuthZ patterns**

| Pattern | Idea in one line | Good for | Watch‑outs |
|---------|-----------------|----------|------------|
| Role‑Based Access Control (RBAC) | Assign roles → roles own permissions → users get roles | Org apps, admin panels | Role explosion, coarse granularity |
| Attribute‑Based Access Control (ABAC) | Policies evaluate user/resource/env **attributes** | Cloud IAM (“allow if tag=dev”) | Harder to audit; complex policy debugging |
| Permission‑Based / Capability Tokens (PBAC) | Signed token lists exact ops permitted on resource | OAuth `scope`, Macaroons | Need short TTL or revocation list |
| ReBAC (Relationship‑Based) | Graph defines “who can access what” via edges | Social networks, Google Zanzibar | Requires graph service; latency considerations |
| ACL (Access‑Control List) | Resource holds list of subjects allowed | File systems, object stores | Scalability; orphaned identities |
| Guard clauses in code | Check permission inside handler (`if user.id == post.owner_id`) | Simple micro‑services | Easy to miss a path; duplicate logic |

---

**Best‑practice cheat‑sheet**

* Prefer **single source of truth**: central IdP (OIDC/SAML) + hierarchy of short‑lived JWT access tokens.  
* Enforce MFA on the IdP; require **step‑up auth** for sensitive operations.  
* Pair JWT with **opaque refresh tokens** stored in Secure, HttpOnly, SameSite cookies.  
* Sign and **also** encrypt tokens crossing untrusted networks; validate every claim (`exp`, `nbf`, `aud`).  
* Deny‑by‑default in AuthZ: start sealed, then open specific permissions via policy.  
* Maintain **least‑privilege roles**; schedule automatic role attestation & recertification.  
* Log every decision: *who*, *what*, *why*, *result* — feed into SIEM for anomaly detection.  
* Rotate secrets and certs automatically; use short TTLs to shrink blast radius.  
* Apply defence‑in‑depth: rate limiting, IP/reputation checks, CAPTCHAs, device posture signals.  

Treat identity as the new perimeter: strong AuthN + granular AuthZ = core of modern zero‑trust architecture.

### Rate Limiting & Throttling
**Rate Limiting & Throttling – essentials for resilient APIs**

---

**Why you need it**

* Protect against brute‑force logins, scraping, abuse, accidental infinite loops.
* Keep backend latency predictable under load; safeguard downstream paid services.
* Enable fair‑use tiers and monetisation (freemium, quota, cost control).

---

**Key dimensions to limit**

| Dimension      | Typical key used           | Example limit                 | Notes                                   |
|----------------|----------------------------|-------------------------------|-----------------------------------------|
| **Identity**   | User ID / OAuth client     | _1 000 requests / hour_       | Fairness between paying tiers.          |
| **Credential** | API key / token            | _10 req / sec_ burst, 100 / min | Quota resets on key rotation.           |
| **IP address** | Source IPv4/IPv6           | _50 req / min_                | Handle NATs → consider **/24** pooling. |
| **Endpoint**   | Verb + path (`POST /login`) | _5 logins / min per user_    | Mitigate credential stuffing.           |
| **Concurrency**| Simultaneous open requests | _20 in‑flight_                | Protect slow resources (video transcode).|

Combine dimensions (e.g., *per‑user‑per‑endpoint*) for stricter control.

---

**Popular algorithms**

| Algorithm          | How it works in a sentence                               | Strengths                      | Gotchas / overhead |
|--------------------|----------------------------------------------------------|--------------------------------|--------------------|
| **Fixed Window**   | Counter per key resets every period (e.g., minute).      | Simple; O(1) storage           | Burst at window edges (“thundering herd”). |
| **Sliding Window** | Store timestamps; count events in the last *N* seconds.  | Smooth; fewer edge bursts      | Needs sorted list or ring buffer. |
| **Leaky Bucket**   | Queue drains at constant rate; overflow = reject.        | Controls sustained rate cleanly| Burst allowance requires queue length tuning. |
| **Token Bucket**   | Tokens accumulate at rate *r* up to capacity; each req consumes 1. | Allows bursts then smooths     | Requires atomic decrement op in distributed store. |
| **Concurrent Semaphore** | `max_in_flight` permits; request blocks or fails if none. | Protects slow downstream calls | Must release tokens even on error/time‑out. |

Choose algorithm based on whether you want to allow bursts (token bucket) or strictly cap (fixed window).

---

**HTTP signalling**

* **429 Too Many Requests** – canonical response when rejecting.  
* **`Retry‑After:`** seconds or HTTP‑date → tells client when to try again.  
* Custom headers (`X‑RateLimit‑Limit`, `X‑RateLimit‑Remaining`, `X‑RateLimit‑Reset`) assist well‑behaved clients.

---

**Implementation patterns**

| Layer            | Tooling examples                         | Benefits                           | Caveats |
|------------------|------------------------------------------|------------------------------------|---------|
| **Edge / CDN**   | Cloudflare, Fastly, CloudFront           | Blocks most bots close to source   | Limited per‑user auth granularity. |
| **API Gateway**  | Kong, Apigee, AWS API Gateway, Envoy     | Central policy, metrics, keys      | Added hop; may need local caches. |
| **Service mesh** | Istio EnvoyFilters, Linkerd limiters     | Per‑service quotas, retries        | Latency in sidecar path. |
| **App code**     | Redis Lua / Go middleware / Rack attack  | Full context (user, plan)          | Requires language‑level consistency. |

Distributed setups must keep counters in **shared stores** (Redis, Memcached, DynamoDB, clustered NGINX key‑value) or approximate via sliding log window in CDN.

---

**Best‑practice cheat‑sheet**

* **Layer‑ed**: block obvious floods at edge; finer auth‑aware limits inside.  
* **Return 429** plus `Retry‑After`; avoid 403 (semantics differ).  
* Treat **idempotent GET** differently from **state‑changing POST**; stricter on write.  
* Provide **burst capacity** (token bucket) so UX doesn’t break on page load sprite requests.  
* Document limits publicly; clients build exponential backoff.  
* Instrument and alert: spikes, sustained 429 ratio, backend latency.  
* Sync clocks across nodes; inaccurate `expires_at` skew weakens enforcement.  
* For login endpoints: couple rate limit with **progressive delays** and **CAPTCHA** past threshold.  
* Keep limits in config / policy engine, not hard‑coded; hot‑patch during incidents.  

Proper rate limiting turns your API from *best‑effort* into *predictably fair*—good for users, ops, and the bottom line.

## Secure Development Practices
### SDLC & Threat Modeling
**Secure SDLC & Threat Modeling – integrating security from idea to prod**

---

**1. SDLC phases and security activities**

| SDLC phase | Primary security goal | Key activities / artifacts | Shifts when done early |
|------------|----------------------|----------------------------|------------------------|
| **Requirements** | Capture security & compliance needs | Misuse‑case workshop, data‑classification matrix, privacy impact assessment | Limits costly redesign later |
| **Design** | Build in safeguards before code | Threat Model (STRIDE / PASTA), security architecture review, crypto choices, trust‑boundary diagrams | Architects pick correct patterns first time |
| **Implementation** | Prevent vulnerabilities in code | Secure coding standards, SAST in CI, secret‑scan, peer review checklist, dependency audit (SBOM) | Bugs found minutes after commit |
| **Verification / Test** | Prove controls work | DAST, API fuzzing, unit + integration security tests, container image scan | Gate broken builds, fail fast |
| **Deployment** | Ship hardened artifacts | IaC lint (Terraform, K8s), least‑priv service accounts, signed images, supply‑chain attestations | Blocks misconfig drift on every release |
| **Operations & Maintenance** | Detect & respond quickly | Runtime SBOM diff, patch cadence, vuln management, alert tuning, chaos drills | Shortens mean‑time‑to‑detect + repair |
| **Retirement** | Remove or quarantine safely | Data retention policy, key destruction, sun‑setting playbook | No zombie attack surface |

---

**2. Core Threat Modeling workflow (STRIDE example)**

1. **Diagram the system** – identify data flows, processes, external entities, data stores.  
2. **Identify threats** – for each element ask STRIDE: *Spoofing, Tampering, Repudiation, Information disclosure, Denial of service, Elevation of privilege*.  
3. **Rank risk** – Likelihood × Impact → high‑risk items bubble up.  
4. **Mitigate early** – add or strengthen controls (e.g., auth, input validation, rate limit, logging).  
5. **Validate** – ensure mitigations lower risk; peer review and update artifacts.  
6. **Iterate** – revisit at every significant design/code change.

> **Outputs**: updated DFD, threat list, risk matrix, mitigation backlog items linked to sprint board.

---

**3. Embedding security in modern CI/CD**

* **Pre‑commit hooks** – secret scanners, lint rules.  
* **PR pipeline** – SAST, dependency‑vuln DB check, license policy, code‑review bot enforcing security checklist.  
* **Build pipeline** – Reproducible build, SBOM emission, container scanning, signature (Sigstore/Cosign).  
* **Deploy pipeline** – IaC security gates, policy as code (OPA Gatekeeper), runtime admission control.  
* **Post‑deploy** – Canary alert rules, CSP / HSTS headers verification, synthetic probes.

---

**4. Checklists by role**

| Role | Must‑do security tasks |
|------|------------------------|
| Product Owner | Capture regulations (GDPR, PCI); define risk appetite. |
| Architect | Produce threat model; approve crypto & auth patterns. |
| Developer | Follow language‑specific secure coding guide; fix SAST findings promptly. |
| DevOps | Enforce least‑priv in cloud/IaC; rotate secrets; backup & test restores. |
| QA / Security Engineer | Design test cases for abuse, run DAST/penetration tests. |
| SRE | Monitor logs, deploy WAF, tune alerts, lead incident drills. |

---

**5. Metrics to prove SSDLC maturity**

* % stories with **security acceptance criteria**  
* Time‑to‑remediate critical SAST issues ⩽ **7 days**  
* Coverage of **threat models per high‑risk feature**  
* Mean time between **dependency updates**  
* Ratio of **incidents traced to design flaws vs implementation bugs**  

Continuous tracking turns ad‑hoc “pen‑test at the end” into reproducible, data‑driven secure development.

---

Adopt a Secure SDLC mindset: **shift left**, automate, threat‑model early, and treat security findings like any other quality metric. The payoff is fewer production fires and faster, safer releases.

### Dependency Management
**Dependency Management – keep the supply‑chain from owning you**

---

**1. Core goals**

* Guarantee that every build uses **known, reproducible versions**.
* Detect and patch **vulnerabilities** in direct *and* transitive packages.
* Track **licenses** and legal obligations.
* Protect build pipeline from **malicious or hijacked packages**.

---

**2. Lock, pin, and verify**

| Practice | What it does | Typical tool | Extra tips |
|----------|--------------|--------------|------------|
| **Version pinning** | Explicitly states exact version. | `requirements.txt`, `package.json` with `"lodash": "4.17.21"` | Pin *all* deps, not `^1.2`. |
| **Lockfile** | Captures full dep graph with checksums. | `poetry.lock`, `package-lock.json`, `Cargo.lock`, `go.sum` | Commit lockfile; fail build if diff not reviewed. |
| **Checksum / signature verification** | Ensures binary/module wasn’t tampered with. | `npm ci` (shasum), Maven Central PGP, `pip --require-hashes` | Turn on **npm’s `--strict-peer-deps`**. |
| **Reproducible builds** | Bit‑for‑bit identical artefacts from same source. | Go, Rust, Bazel | Store artefact hash in SBOM/attestation. |

---

**3. Automated scanning & monitoring**

| Stage | What to scan | Tools (example) |
|-------|--------------|-----------------|
| **Pull Request** | SCA (static), secret leaks, license policy | Snyk, Renovate, Trivy, Gitleaks |
| **Build** | SBOM generation, checksum compare | CycloneDX, Syft, Grype |
| **Registry / Image** | CVE feed diff, outdated base images | Clair, Anchore, AWS ECR scan |
| **Runtime** | In‑memory lib version, process hash | Falco, eBPF scanners |

*Alert rules* → “Critical CVE present in prod > 30 days” triggers escalation.

---

**4. Update strategy**

* **Renovate / Dependabot** create automatic PRs; configure *batch & schedule* (e.g., every Monday).  
* **SemVer tiers** → auto‑merge patch/minor; manual review for major.  
* Maintain **changelogs** and roll‑forward plans; use canary deploy to catch regressions.  
* If vendor won’t patch quickly, apply **selective patching** (fork with fix) or **security shim** (e.g., `npm audit fix --force` plus lockfile diff).

---

**5. Supply‑chain hardening**

1. **Use private proxies / caches** (e.g., Artifactory, Verdaccio) to avoid live pulls from public registries during build.  
2. Enable **Two‑factor auth & signing** for package publishers in your org.  
3. Require **verified commit signatures** (`git config commit.gpgsign true`) on main branches.  
4. Adopt **Sigstore/cosign** to sign OCI images and verify in cluster admission control.  
5. Implement **Provenance attestations** (SLSA ≥ level 2) so downstream consumers trust your output.  
6. Watch for **typosquatting / dependency confusion**—use scoped package names and set `pypi-proxy=false` for internal libs.

---

**6. License compliance quick‑view**

| License family | Typical constraint | Action |
|----------------|--------------------|--------|
| *Permissive* (MIT, BSD, Apache‑2.0) | Notice & attribution | Keep NOTICE file in artefact. |
| *Weak Copyleft* (MPL‑2.0, LGPL‑2.1) | Modifications to library must be released | Prefer dynamic linking. |
| *Strong Copyleft* (GPL‑3.0) | Derivative work must be OSS | Avoid in SaaS closed source unless business OK. |
| *Patents* (Apache‑2.0) | Grants patent license | Safe for commercial use. |

Automate license scan → block build on forbidden licenses, generate attribution docs for distribution.

---

**7. Incident response when a dependency pops a zero‑day**

* **Assess blast radius** – Which services import the package? Query SBOM DB.  
* **Patch or pin** – If fix exists: update and redeploy. No fix → pin previous safe version or hot‑patch.  
* **Runtime mitigations** – Feature flags, WAF rules, sandboxing.  
* **Audit logs** – Look for IOC patterns (malicious post‑install script calls, unexpected outbound traffic).  
* **Post‑mortem** – Improve monitoring rule (CVE watch), tighten publish pipeline, upstream fix contribution.

---

A disciplined dependency‑management program—pin, scan, sign, and update—turns the Wild West of open‑source supply chains into a manageable, measurable asset instead of a liability.


### Code Review & Linters

* Fail build on **new** high‑severity findings, not historical debt.  
* Cache tool installs to keep pipeline sub‑5 min.

---

**Metrics to watch**

* PR review turnaround < 24 h (< 4 h for hotfixes)  
* Zero open critical linter/SAST findings in main branch  
* Secret‑scan MTTR < 30 min  
* Code‑coverage of security lint rules ≥ 90 %

---

**Common anti‑patterns**

* Rubber‑stamp “LGTM” approvals.  
* Disabling linter rules project‑wide.  
* Skipping review for “trivial” changes.  
* Treating SAST findings as someone else’s problem.

Embed these practices early—let linters catch the predictable, freeing reviewers for deep thinking.

