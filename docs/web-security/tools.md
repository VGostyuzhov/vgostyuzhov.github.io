# API & Microservice Security

## REST vs GraphQL Concerns

| Dimension | REST (resource-oriented) | GraphQL (query-oriented) | Why it matters |
|-----------|--------------------------|--------------------------|----------------|
| **Attack surface** | Fixed URIs & verbs mean predictable endpoints; WAF rules easier. | Single `/graphql` POST can express many operations; harder for firewalls & rate limits to understand intent. | Assessing risk & building rules. |
| **Over-fetch / under-fetch** | Client may pull entire resource even if only one field needed - wasted bandwidth. | Client specifies exact fields; risk of *very* heavy queries. | Performance & DoS resilience. |
| **Query complexity limits** | Usually bounded by endpoint logic. | Must defend against nested, circular, or expensive queries; implement depth/complexity limits and timeouts. | Avoid CPU/DB exhaustion. |
| **Authorization model** | Per-URI/per-verb RBAC fits neatly: "`PATCH /users/42` requires `edit:user`". | Field-level auth needed (`{ user { ssn } }`); use schema directives or custom resolvers. | Prevent data leakage. |
| **Introspection exposure** | N/A - API docs separate (OpenAPI). | Introspection and `__schema` can reveal everything; disable or restrict in prod. | Recon for attackers. |
| **Error handling** | HTTP status codes (`404`, `401`) convey a lot. | Always `200 OK`; errors inside JSON require extra parsing, and status leaks less obvious. | Client simplicity, caching proxies. |
| **Caching** | URI + method good fit for CDNs (`GET /products/1`). | Response depends on *query string* (post body); need persisted-query caching or APQ, complicates edge caching. | Latency & infra cost. |
| **Versioning** | New endpoint/URI (or `v2/` prefix). | Designed for *non-breaking* evolution; add fields, deprecate old, no version bump. | Long-term maintenance. |
| **Input validation** | Hand-rolled per endpoint or OpenAPI validators. | Strong schema type checks, but watch for **GraphQL injection** via resolver args. | Reduce exploitable bugs. |
| **File uploads** | Multipart endpoints straightforward. | Need separate REST endpoint or GraphQL multipart spec (`apollo-upload`). | Extra work, more room for misconfig. |

---

**Threats unique to GraphQL**

* **Deeply nested queries** - attacker crafts `{a{b{c{d...}}}}` causing resolver N-squared calls. Mitigate with query depth / cost analysis, timeouts, max nodes.
* **Batching introspection** - enumerates hidden types/fields; disable `__schema` or require auth.
* **Field-level auth gaps** - forgetting to check one sensitive resolver leaks data.
* **Alias + fragments DoS** - same expensive field under many aliases bypasses depth limit; count nodes/field repetitions too.

**REST-specific pitfalls**

* **Unfiltered query params** - naive SQL concatenation in `GET /search?q=...`.
* **Verb tunnelling** - proxies stripping `X-HTTP-Method-Override` expose "hidden" verbs.
* **Excessive endpoint sprawl** - forgotten legacy URIs linger unpatched.

---

**Safeguard checklist**

1. Enforce global rate limits; for GraphQL also add **per-query complexity quotas**.
2. Apply **auth in resolvers** (GraphQL) and middleware (REST); default-deny.
3. Validate input types and lengths server-side even with GraphQL schema.
4. Disable GraphQL introspection in production or require admin token.
5. For REST, keep OpenAPI spec updated and use it to generate validators + tests.
6. Cache safely: ETag/Cache-Control on REST; persisted queries + allow-list on GraphQL.
7. Log with enough granularity: full URI and status for REST, query plus variables hash for GraphQL (watch PII).

!!! info "External Resources"
    - [REST Security Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/REST_Security_Cheat_Sheet.html) (OWASP)
    - [GraphQL Security Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/GraphQL_Cheat_Sheet.html) (OWASP)
    - [API Security Top 10](https://owasp.org/API-Security/) (OWASP)

## AuthN & AuthZ Patterns

Authentication (AuthN) proves identity, while Authorization (AuthZ) checks permissions. For a deep dive into protocols and their security threats, see the **[Authentication Section](../authentication/index.md)**.

**Key Web Patterns:**

*   **[Sessions & Cookies](../authentication/sessions.md)**: Traditional stateful web auth.
*   **[JWT (JSON Web Tokens)](../authentication/jwt.md)**: Stateless token-based auth for APIs and SPAs.
*   **[OAuth 2.0 & OIDC](../authentication/oauth2-oidc.md)**: Modern delegated authorization and identity.
*   **[mTLS](../authentication/mtls.md)**: Mutual TLS for secure service-to-service communication.
*   **[WebAuthn](../authentication/webauthn.md)**: Modern, phishing-resistant passwordless authentication.

**Best-practice cheat-sheet**

* Prefer **single source of truth**: central IdP (OIDC/SAML) + hierarchy of short-lived JWT access tokens.
* Enforce MFA on the IdP; require **step-up auth** for sensitive operations.
* Pair JWT with **opaque refresh tokens** stored in Secure, HttpOnly, SameSite cookies.
* Sign and **also** encrypt tokens crossing untrusted networks; validate every claim (`exp`, `nbf`, `aud`).
* Deny-by-default in AuthZ: start sealed, then open specific permissions via policy.
* Maintain **least-privilege roles**; schedule automatic role attestation & recertification.
* Log every decision: *who*, *what*, *why*, *result* - feed into SIEM for anomaly detection.
* Rotate secrets and certs automatically; use short TTLs to shrink blast radius.
* Apply defence-in-depth: rate limiting, IP/reputation checks, CAPTCHAs, device posture signals.

Treat identity as the new perimeter: strong AuthN + granular AuthZ = core of modern zero-trust architecture.

!!! info "External Resources"
    - [Authentication Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Authentication_Cheat_Sheet.html) (OWASP)
    - [Authorization Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Authorization_Cheat_Sheet.html) (OWASP)
    - [OAuth 2.0 Security Best Current Practice](https://datatracker.ietf.org/doc/html/draft-ietf-oauth-security-topics) (IETF)

## Rate Limiting & Throttling

**Why you need it**

* Protect against brute-force logins, scraping, abuse, accidental infinite loops.
* Keep backend latency predictable under load; safeguard downstream paid services.
* Enable fair-use tiers and monetisation (freemium, quota, cost control).

---

**Key dimensions to limit**

| Dimension      | Typical key used           | Example limit                 | Notes                                   |
|----------------|----------------------------|-------------------------------|-----------------------------------------|
| **Identity**   | User ID / OAuth client     | _1 000 requests / hour_       | Fairness between paying tiers.          |
| **Credential** | API key / token            | _10 req / sec_ burst, 100 / min | Quota resets on key rotation.           |
| **IP address** | Source IPv4/IPv6           | _50 req / min_                | Handle NATs - consider **/24** pooling. |
| **Endpoint**   | Verb + path (`POST /login`) | _5 logins / min per user_    | Mitigate credential stuffing.           |
| **Concurrency**| Simultaneous open requests | _20 in-flight_                | Protect slow resources (video transcode).|

Combine dimensions (e.g., *per-user-per-endpoint*) for stricter control.

---

**Popular algorithms**

| Algorithm          | How it works in a sentence                               | Strengths                      | Gotchas / overhead |
|--------------------|----------------------------------------------------------|--------------------------------|--------------------|
| **Fixed Window**   | Counter per key resets every period (e.g., minute).      | Simple; O(1) storage           | Burst at window edges ("thundering herd"). |
| **Sliding Window** | Store timestamps; count events in the last *N* seconds.  | Smooth; fewer edge bursts      | Needs sorted list or ring buffer. |
| **Leaky Bucket**   | Queue drains at constant rate; overflow = reject.        | Controls sustained rate cleanly| Burst allowance requires queue length tuning. |
| **Token Bucket**   | Tokens accumulate at rate *r* up to capacity; each req consumes 1. | Allows bursts then smooths     | Requires atomic decrement op in distributed store. |
| **Concurrent Semaphore** | `max_in_flight` permits; request blocks or fails if none. | Protects slow downstream calls | Must release tokens even on error/time-out. |

Choose algorithm based on whether you want to allow bursts (token bucket) or strictly cap (fixed window).

---

**HTTP signalling**

* **429 Too Many Requests** - canonical response when rejecting.
* **`Retry-After:`** seconds or HTTP-date - tells client when to try again.
* Custom headers (`X-RateLimit-Limit`, `X-RateLimit-Remaining`, `X-RateLimit-Reset`) assist well-behaved clients.

---

**Implementation patterns**

| Layer            | Tooling examples                         | Benefits                           | Caveats |
|------------------|------------------------------------------|------------------------------------|---------|
| **Edge / CDN**   | Cloudflare, Fastly, CloudFront           | Blocks most bots close to source   | Limited per-user auth granularity. |
| **API Gateway**  | Kong, Apigee, AWS API Gateway, Envoy     | Central policy, metrics, keys      | Added hop; may need local caches. |
| **Service mesh** | Istio EnvoyFilters, Linkerd limiters     | Per-service quotas, retries        | Latency in sidecar path. |
| **App code**     | Redis Lua / Go middleware / Rack attack  | Full context (user, plan)          | Requires language-level consistency. |

Distributed setups must keep counters in **shared stores** (Redis, Memcached, DynamoDB, clustered NGINX key-value) or approximate via sliding log window in CDN.

---

**Best-practice cheat-sheet**

* **Layered**: block obvious floods at edge; finer auth-aware limits inside.
* **Return 429** plus `Retry-After`; avoid 403 (semantics differ).
* Treat **idempotent GET** differently from **state-changing POST**; stricter on write.
* Provide **burst capacity** (token bucket) so UX doesn't break on page load sprite requests.
* Document limits publicly; clients build exponential backoff.
* Instrument and alert: spikes, sustained 429 ratio, backend latency.
* Sync clocks across nodes; inaccurate `expires_at` skew weakens enforcement.
* For login endpoints: couple rate limit with **progressive delays** and **CAPTCHA** past threshold.
* Keep limits in config / policy engine, not hard-coded; hot-patch during incidents.

!!! info "External Resources"
    - [Rate Limiting - OWASP Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Denial_of_Service_Cheat_Sheet.html) (OWASP)
    - [API Rate Limiting Best Practices](https://cloud.google.com/architecture/rate-limiting-strategies-techniques) (Google Cloud)
    - [RFC 6585 - 429 Too Many Requests](https://datatracker.ietf.org/doc/html/rfc6585#section-4) (IETF)
