# API & Microservice Security

API security requires validation of callers, resource ownership, tenant boundaries, and computational cost. REST and GraphQL differ in interface structure; neither supplies these controls automatically.

## REST vs GraphQL Concerns

Choose controls based on the operation and its data access, not only the interface style. Both designs require explicit authorization and resource limits.

| Dimension | REST (resource-oriented) | GraphQL (query-oriented) | Why it matters |
|-----------|--------------------------|--------------------------|----------------|
| **Attack surface** | Multiple routes and methods expose operations; inventory and consistent policy enforcement are necessary. | One endpoint can express many operations; inspect operation cost and resolver behavior, not only the URL. | Assessing risk & building rules. |
| **Over-fetch / under-fetch** | Client may pull entire resource even if only one field needed - wasted bandwidth. | Client specifies exact fields; risk of *very* heavy queries. | Performance & DoS resilience. |
| **Query complexity limits** | Usually bounded by endpoint logic. | Must defend against nested, circular, or expensive queries; implement depth/complexity limits and timeouts. | Avoid CPU/DB exhaustion. |
| **Authorization model** | Check action, object ownership, tenant, and properties, not only route/verb permissions. | Enforce object, tenant, and field policy through a shared authorization layer used by resolvers. | Prevent data leakage. |
| **Introspection exposure** | N/A - API docs separate (OpenAPI). | Introspection reveals schema structure, not automatically protected records; restrict exposure according to deployment policy. | Recon for attackers. |
| **Error handling** | HTTP status codes (`404`, `401`) convey a lot. | Status depends on media type and failure stage; execution results can contain both data and errors. | Client simplicity, caching proxies. |
| **Caching** | URI + method good fit for CDNs (`GET /products/1`). | Response depends on operation, variables, and caller; persisted queries can simplify caching but do not make private data public. | Latency & infra cost. |
| **Versioning** | Often uses explicit versions, but compatible evolution is also possible. | Designed for *non-breaking* evolution; add fields, deprecate old, no version bump. | Long-term maintenance. |
| **Input validation** | Hand-rolled per endpoint or OpenAPI validators. | Schema types constrain inputs but do not prevent SQL/command injection inside resolvers. | Reduce exploitable bugs. |
| **File uploads** | Multipart endpoints straightforward. | Need separate REST endpoint or GraphQL multipart spec (`apollo-upload`). | Extra work, more room for misconfig. |

---

**GraphQL operation risks**

* **Deeply nested queries** - nested fields and large list sizes can multiply resolver and database work; actual complexity depends on schema and implementation. Mitigate with query depth / cost analysis, timeouts, max nodes.
* **Batching abuse** - multiple operations or aliases can bypass a per-HTTP-request quota; limit total work and authentication attempts across the batch.
* **Field-level auth gaps** - forgetting to check one sensitive resolver leaks data.
* **Alias + fragments DoS** - same expensive field under many aliases bypasses depth limit; count nodes/field repetitions too.

**REST endpoint risks**

* **Unfiltered query params** - naive SQL concatenation in `GET /search?q=...`.
* **Method override inconsistencies** - routing, authentication, authorization, and proxies may interpret override headers differently; disable unused overrides or apply policy to the effective operation consistently.
* **Excessive endpoint sprawl** - forgotten legacy URIs linger unpatched.

---

**Safeguard checklist**

1. Enforce global rate limits; for GraphQL also add **per-query complexity quotas**.
2. Apply shared authorization policy to every object/action/field access, including nested resolvers and tenant boundaries; default-deny.
3. Validate input types and lengths server-side even with GraphQL schema.
4. Restrict introspection if appropriate to the API audience; do not use schema secrecy as access control.
5. For REST, keep OpenAPI spec updated and use it to generate validators + tests.
6. Set cache policy and keys for caller-specific data; include authorization context or avoid shared caching. Persisted-query caching alone does not isolate users.
7. Log operation identifiers, outcomes, and useful security context; redact credentials and sensitive query/variable data. Hashing a low-entropy secret does not guarantee anonymity.

**Authorization example:** A user with `read:invoice` requests invoice `inv-42` from another tenant. The REST handler and GraphQL invoice resolver must both reject access after checking tenant membership and ownership. Hiding the invoice ID or disabling introspection cannot replace that check.

**GraphQL HTTP example:** With `application/graphql-response+json`, a parse or validation error can produce HTTP 400 before execution. A successfully executed query can return HTTP 200 with partial `data` and an `errors` array. Clients must inspect the GraphQL result as well as HTTP status; do not treat all errors as successful execution.

!!! info "External Resources"
    - [Serving GraphQL over HTTP](https://graphql.org/learn/serving-over-http/) (GraphQL Foundation)
    - [REST Security Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/REST_Security_Cheat_Sheet.html) (OWASP)
    - [GraphQL Security Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/GraphQL_Cheat_Sheet.html) (OWASP)

## AuthN & AuthZ Patterns

Authentication (AuthN) proves identity, while Authorization (AuthZ) checks permissions. For a deep dive into protocols and their security threats, see the **[Authentication Section](../authentication/index.md)**.

**Key Web Patterns:**

*   **[Sessions & Cookies](../authentication/sessions.md)**: Traditional stateful web auth.
*   **[JWT (JSON Web Tokens)](../authentication/jwt.md)**: Claims representation with a purpose-specific validation policy.
*   **[OAuth 2.0 & OIDC](../authentication/oauth2-oidc.md)**: Modern delegated authorization and identity.
*   **[mTLS](../authentication/mtls.md)**: Mutual TLS for secure service-to-service communication.
*   **[WebAuthn](../authentication/webauthn.md)**: Modern, phishing-resistant passwordless authentication.

**Best-practice cheat-sheet**

* Select session or token architecture for the trust boundaries. Federation can centralize login, while each service still authorizes its own resources; access tokens may be opaque or JWTs.
* Enforce MFA on the IdP; require **step-up auth** for sensitive operations.
* Protect refresh credentials according to the client profile. A backend can retain them server-side; browser cookie delivery needs appropriate scope and CSRF defenses.
* Use correctly validated HTTPS for bearer-token transport. Apply message-level encryption only when confidentiality requirements call for it; encrypted bearer credentials can still be stolen and replayed. Validate trusted issuer, algorithms/keys, intended audience, time bounds, and token purpose.
* Deny access unless the subject, action, resource, tenant, and context satisfy explicit policy.
* Maintain **least-privilege roles**; schedule automatic role attestation & recertification.
* Record security-relevant authorization outcomes with identity, resource, and reason; balance visibility with privacy, retention, and log-volume limits.
* Rotate secrets and certs automatically; use short TTLs to shrink blast radius.
* Apply defence-in-depth: rate limiting, IP/reputation checks, CAPTCHAs, device posture signals.

**Failure example:** An API receives a correctly signed OIDC ID token. Reject it if the API requires an access token with its own audience and scopes; a valid authentication statement for another client does not authorize this API.

!!! info "External Resources"
    - [Authentication Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Authentication_Cheat_Sheet.html) (OWASP)
    - [Authorization Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Authorization_Cheat_Sheet.html) (OWASP)
    - [OAuth 2.0 Security Best Current Practice](https://www.rfc-editor.org/rfc/rfc9700.html) (IETF)

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
| **Credential** | API key plus stable owner ID | _10 req / sec_ burst, 100 / min | Keep owner-level quotas across key rotation when required. |
| **IP address** | Trusted client-address signal | _50 req / min_ | NAT can group unrelated users; IPv6/subnet aggregation requires a deliberate abuse and fairness policy. |
| **Endpoint**   | Verb + path (`POST /login`) | _5 logins / min per user_    | Mitigate credential stuffing.           |
| **Concurrency**| Simultaneous open requests | _20 in-flight_                | Protect slow resources (video transcode).|

Combine dimensions (e.g., per-account-per-endpoint and source-network limits). Values in this table are illustrations, not production defaults. Trust forwarded client addresses only from known proxies. Avoid using an attacker-supplied account name as the sole global lockout key.

---

**Popular algorithms**

| Algorithm          | How it works in a sentence                               | Strengths                      | Gotchas / overhead |
|--------------------|----------------------------------------------------------|--------------------------------|--------------------|
| **Fixed Window**   | Counter per key resets every period (e.g., minute).      | O(1) state per active key           | Up to twice the window allowance near a reset boundary. |
| **Sliding Window** | Store timestamps; count events in the last *N* seconds.  | Smooth; fewer edge bursts      | Needs sorted list or ring buffer. |
| **Leaky Bucket**   | Queue drains at constant rate; overflow = reject.        | Controls sustained rate cleanly| Burst allowance requires queue length tuning. |
| **Token Bucket**   | Tokens accumulate at rate *r* up to capacity; each req consumes 1. | Allows bursts then smooths     | Refill, check, and decrement must be coordinated atomically. |
| **Concurrent Semaphore** | `max_in_flight` permits; request blocks or fails if none. | Protects slow downstream calls | Must release tokens even on error/time-out. |

Choose the invariant before choosing an algorithm. A fixed window limits each aligned interval, not every rolling interval. A token bucket deliberately allows a bounded burst; a sliding log can enforce a limit over the preceding duration at greater storage cost.

**Boundary example:** A limit of 100 requests per minute admits 100 at 12:00:59 and another 100 at 12:01:00 with aligned fixed windows. That is 200 requests in roughly one second. A rolling 60-second limit would count both groups in the same interval and reject the excess.

---

**HTTP signalling**

* **429 Too Many Requests** - canonical response when rejecting.
* **`Retry-After:`** seconds or HTTP-date - tells client when to try again.
* Custom headers (`X-RateLimit-Limit`, `X-RateLimit-Remaining`, `X-RateLimit-Reset`) assist well-behaved clients.

---

**Implementation patterns**

| Layer            | Tooling examples                         | Benefits                           | Caveats |
|------------------|------------------------------------------|------------------------------------|---------|
| **Edge / CDN**   | Cloudflare, Fastly, CloudFront           | Coarse filtering near clients   | Limited per-user auth granularity. |
| **API Gateway**  | Kong, Apigee, AWS API Gateway, Envoy     | Central policy, metrics, keys      | Added hop; may need local caches. |
| **Service mesh** | Proxy filters / quota integrations where supported     | Service-level enforcement        | Capabilities depend on proxy and integration. |
| **App code**     | Redis Lua / Go middleware / Rack attack  | Full context (user, plan)          | Requires language-level consistency. |

Distributed enforcement must define how limits combine across nodes. A shared atomic counter, centralized quota service, partitioned ownership, or bounded local allocations can be appropriate; independent per-node quotas multiply the effective allowance.

- Atomically update counters and their expiry, or atomically check/decrement a bucket. Concurrent read-then-write operations can oversubscribe the quota.
- Bound memory use and entry lifetimes for attacker-controlled keys.
- Release concurrency permits on errors and cancellation; use appropriate leases/recovery for crashed workers.
- Specify behavior when the limiter is unavailable: failing open preserves availability but admits abuse; failing closed can deny legitimate service. Apply a policy suitable for each operation.
- Test edge bursts, concurrent callers, key rotation, and node failure against the documented invariant.

---

**Best-practice cheat-sheet**

* **Layered**: block obvious floods at edge; finer auth-aware limits inside.
* **Return 429** plus `Retry-After`; avoid 403 (semantics differ).
* Preserve safe-method semantics: GET must not change state. Base quotas on actual operation cost and abuse risk; an expensive read can need stricter limits than a cheap write.
* Provide **burst capacity** (token bucket) so UX doesn't break on page load sprite requests.
* Document limits publicly; clients build exponential backoff.
* Instrument and alert: spikes, sustained 429 ratio, backend latency.
* Use appropriate monotonic timing locally and account for clock skew in distributed expiry calculations.
* For login endpoints: couple rate limit with **progressive delays** and **CAPTCHA** past threshold.
* Keep limits in config / policy engine, not hard-coded; hot-patch during incidents.

!!! info "External Resources"
    - [Rate Limiting - OWASP Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Denial_of_Service_Cheat_Sheet.html) (OWASP)
    - [Traffic and Load Management](https://docs.cloud.google.com/architecture/infra-reliability-guide/traffic-load) (Google Cloud)
    - [RFC 6585 - 429 Too Many Requests](https://datatracker.ietf.org/doc/html/rfc6585#section-4) (IETF)
