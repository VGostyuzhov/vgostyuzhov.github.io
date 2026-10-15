# Choosing an Authentication Architecture

Choose an architecture from the caller, trust boundaries, and authorization requirements. A cookie, JWT, OAuth grant, and client certificate solve different problems and often belong in the same design. Use the [worked design patterns](design-patterns.md) to practice defending a choice and the [incident case studies](case-studies.md) to test its assumptions.

## Questions Before Choosing a Protocol

Clarify the requirements before drawing a login flow.

| Question | Why it changes the design |
|---|---|
| Is the caller a person, a service, or a service acting for a person? | Separates interactive login, application identity, and delegated authority |
| Can the client protect a credential? | A browser or widely distributed CLI cannot keep an embedded client secret confidential |
| Who owns the identity and account lifecycle? | Determines whether local authentication, enterprise federation, or workload identity is appropriate |
| Which API, tenant, objects, and actions are needed? | Defines audience restrictions and authorization policy rather than a global authenticated/not-authenticated flag |
| Must access end immediately after compromise or offboarding? | Determines acceptable cache, session, and token-validation delays |
| Can the client reach a browser and receive a callback? | Changes the choice between authorization code and a supported device flow |
| What can an attacker compromise? | Browser script, endpoint, integration provider, issuing key, and network position require different controls |
| What happens during an identity-provider outage? | New login, renewal, and existing-request validation have different dependencies |

**Example:** “Expose our application to a machine client” is incomplete. A partner's nightly export runs as an application; a personal desktop tool acts for a user; a CI job represents a workload. They should not all receive a shared administrator password.

!!! info "External Resources"
    - [Authentication Architecture Patterns](https://cheatsheetseries.owasp.org/cheatsheets/Authentication_Patterns_Cheat_Sheet.html) (OWASP)
    - [OAuth Client Types and Grants](https://www.rfc-editor.org/rfc/rfc6749.html#section-2.1) (IETF)

## What to Use When

A **Backend for Frontend (BFF)** handles OAuth and API calls on behalf of browser code; see the [BFF pattern](design-patterns.md#2-protect-a-browser-application-with-a-backend-for-frontend). A **Security Token Service (STS)** validates a presented credential and issues a credential under its trust policy; see [delegation](design-patterns.md#3-preserve-user-delegation-across-service-calls), where a service acts with explicitly granted authority for a user. A **sender constraint** requires proof of a credential-bound key when a token is used, instead of accepting possession of the token alone.

The starting choices below assume HTTPS, maintained protocol libraries, and explicit server-side authorization.

| Situation | Reasonable starting choice | Why | Main cost or limitation | Choose differently when |
|---|---|---|---|---|
| First-party server-rendered application | Framework-managed server session; local login or OIDC as needed | Straightforward browser continuity and server-side invalidation | CSRF protection and session-state availability | Direct third-party API access or federation requires additional protocols |
| Browser UI with a controllable backend | OIDC authorization code + PKCE at a BFF; opaque browser session cookie | API/refresh credentials stay in the backend | Proxy capacity, session store, and CSRF controls | A direct-to-API browser client is a real constraint and its token exposure is accepted |
| Standalone SPA or native mobile app | Public-client authorization code + PKCE; native app uses an external user agent | Supports user-delegated access without an embedded secret | Client compromise can expose or use credentials | A backend can retain credentials, or a constrained device needs a device flow |
| Human-operated CLI | System-browser code flow + PKCE where supported; device flow for appropriate input/callback constraints | Preserves user identity and provider login controls | Secure local storage and login-phishing considerations | Unattended automation should receive a workload/application identity instead |
| External partner daemon | OAuth client credentials with a distinct registered application identity | Separates each integration's permissions and lifecycle from human sessions | Registration, key/secret lifecycle, token service dependency | A small non-delegated API may justify carefully scoped API keys; stronger replay resistance may justify certificate-bound tokens |
| Internal workloads | Short-lived platform/workload identity and authenticated service channels | Avoids sharing one long-lived credential across services | Issuance bootstrap, trust distribution, and rotation | Cross-domain API permissions require an OAuth access credential as well |
| Service acting for a user downstream | Audience-specific delegation/token exchange where supported | Preserves subject and actor without forwarding one powerful token everywhere | STS policy and interoperability complexity | A call is truly application-only, or a narrow fixed service contract needs no user delegation |
| Enterprise SaaS login | OIDC or SAML matched to the customer's IdP; local tenant/session policy | Reuses customer authentication while retaining application authorization | Tenant onboarding, account mapping, provisioning, and offboarding | No federation requirement exists; avoid unnecessary federation infrastructure |
| CI job accessing cloud resources | Provider-supported workload federation | Replaces stored deployment credentials with constrained temporary access | A malicious trusted workflow can still obtain valid credentials | The platform lacks federation support and a tightly managed credential is necessary |

For sensitive browser applications, favor a backend architecture where feasible: RFC 10017 recommends against browser-only OAuth for business, sensitive, and personal-data applications. Native apps have a separate platform security model. A first-party UI/API can also use ordinary server sessions without introducing OAuth between its own components.

Use [WebAuthn](webauthn.md) for phishing-resistant human authentication where appropriate. It can be the IdP's login method or the application's local login method; it does not replace service identity or API authorization.

!!! info "External Resources"
    - [OAuth for Browser-Based Applications — RFC 10017, August 2026](https://www.rfc-editor.org/rfc/rfc10017.html) (IETF)
    - [OAuth for Native Apps](https://www.rfc-editor.org/rfc/rfc8252.html) (IETF)
    - [OAuth Device Authorization Grant](https://www.rfc-editor.org/rfc/rfc8628.html) (IETF)

## Separate Grant, Client Proof, and Token Use

For OAuth, answer three independent questions: what authorizes issuance, how the client authenticates if applicable, and how an API validates token use.

| Decision | Options | Important distinction |
|---|---|---|
| Grant | Authorization code, client credentials, device grant, supported token exchange | Client credentials represents application authority, not the current human user |
| Client authentication | Managed secret, signed client assertion such as private_key_jwt, or mTLS where supported | Protecting the token endpoint does not automatically sender-constrain the access token |
| Workload federation | Platform assertion exchanged under the target provider's trust policy and grant/profile | This is not necessarily a client-credentials grant or a human OIDC login |
| Access-token use | Bearer, certificate-bound token, DPoP-bound token where supported | Sender constraints require the resource server to validate the matching proof |
| Representation | Opaque token or JWT according to the API contract | A format choice does not determine the grant or authorization policy |

**One integration, three choices:** A partner's daemon uses the **client-credentials grant** because it acts under its own application authority. It uses **private_key_jwt client authentication** so the issuer checks a short-lived assertion signed by its registered key. If the API contract still issues **bearer access tokens**, a thief holding one can use it without that client assertion. To prevent token-only replay, the issuer and API must additionally support and enforce a sender-constrained token. Choosing stronger issuance authentication does not implicitly change the API's token-use contract.

**Attack comparison:** An attacker steals only an API token. Rotating the token-endpoint client secret does not necessarily invalidate that token or prevent its use. A correctly enforced sender constraint can prevent token-only replay, but not an attacker who can also use the bound key. Compromise response must address both issuance and outstanding credentials.

**API keys:** A per-client key can be appropriate for a simple, narrowly scoped integration when its bearer nature, storage, rotation, and revocation are acceptable. Keep keys out of URLs, separate test/production identities, and enforce permissions and quotas per owner. API keys do not provide a standard user-consent/delegation flow. A public browser cannot hide one.

!!! info "External Resources"
    - [JWT Client Authentication](https://www.rfc-editor.org/rfc/rfc7523.html#section-2.2) (IETF)
    - [mTLS Client Authentication and Certificate-Bound Tokens](https://www.rfc-editor.org/rfc/rfc8705.html) (IETF)
    - [DPoP Proof-of-Possession Tokens](https://www.rfc-editor.org/rfc/rfc9449.html) (IETF)

## JWT, Opaque Tokens, and Revocation Tradeoffs

An **opaque token** is a credential interpreted through a server-side contract or lookup rather than by locally decoding claims. **Introspection** queries the authorization server about a token's active state and metadata. Choose validation and revocation behavior together. Token opacity does not inherently make a system secure, and JWT signing does not make authorization permanently correct.

| Design | Strength | Failure or cost to explain |
|---|---|---|
| Local JWT validation | Avoids a per-request issuer lookup; can continue with trusted cached keys | Revocation/permission changes may not be visible until expiry or a separate check; key caches need controlled refresh |
| Opaque token with introspection | Central service can report active state and token metadata | Network dependency, availability, and cache freshness become security decisions |
| Server-side browser session | Server controls login state and can invalidate it | Replicas, caches, and requests already in progress affect effective revocation latency |
| Short access-token lifetime plus refresh | Bounds exposure when access credentials are self-contained | Revoking refresh prevents renewal, but does not inherently end existing access tokens |

A **positive cache** retains a successful check and reuses it without contacting the authority again. For example, an API may save an introspection response saying that a token is active to reduce latency and issuer load. This creates a period when a later revocation is not yet visible. See the [mechanisms and worked timing bound](oauth2-oidc.md#revocation-and-current-authorization) before selecting a cache duration.

**Reasoning exercise:** If an API caches an active introspection result for 60 seconds, a centrally revoked token can remain accepted for up to the remaining cache interval, assuming no other invalidation mechanism. If the requirement is a shorter bound, change the cache or add a revocation signal. Do not promise “instant logout” while keeping a positive cache that contradicts it.

Define whether high-risk operations recheck current account/tenant state. During an outage, a known unexpired credential with trusted cached verification material differs from an unknown key or an expired credential. Do not disable validation to preserve availability.

!!! info "External Resources"
    - [Token Introspection and Caching](https://www.rfc-editor.org/rfc/rfc7662.html) (IETF)
    - [Token Revocation](https://www.rfc-editor.org/rfc/rfc7009.html) (IETF)
    - [JWT Validation Best Practices](https://www.rfc-editor.org/info/rfc8725/) (IETF)

## Authorization Decisions and Their Caches

Authentication identifies the caller; authorization evaluates a specific proposed operation. Consider user `u7`, action `invoice.read`, resource `invoice-42`, and tenant `tenant-A`. The policy might require current tenant membership and ownership of that invoice. Permission to read a different tenant's invoice cannot be reused for this request.

| Model | What policy evaluates | Example |
|---|---|---|
| Role-based access control (RBAC) | Permissions associated with a scoped role | Tenant-A billing administrators may manage Tenant-A invoices |
| Attribute-based access control (ABAC) | Properties of subject, resource, action, and environment | Allow export only for active staff, approved data classification, and an accepted device context |
| Relationship-based access control | Relationships between subject and object | A document owner or explicitly assigned collaborator can read the document |

These models can be combined. The server must constrain editable properties too: an allowed profile update does not authorize setting `isGlobalAdmin` or transferring tenant ownership. See [object and property authorization](../web-security/vulnerabilities.md#common-web-vulnerabilities).

A decision cache stores the result of a policy evaluation, rather than just token validity. Its key must distinguish the inputs that can change the decision: subject, tenant, action, resource, relevant context, and policy/data versions where used. A key of only `userId` can incorrectly reuse a read permission for a write operation or another tenant. If ownership changes, the same key may also need invalidation even though the user and token are unchanged.

**Example:** Cache an allow for `u7 / tenant-A / invoice-42 / read` for at most 10 seconds under a defined policy version. It cannot authorize `invoice-99`, a write, or tenant B. During a policy-service outage, a valid unexpired entry may be usable only for the explicitly approved operation/risk class. An expired or mismatched entry cannot be treated as permission. High-impact changes may require a fresh decision and fail when it cannot be obtained.

Cache entries containing privileged results need protection from unauthorized modification. Define cache age, update propagation, invalidation and outage behavior together; a correct key does not solve stale permissions.

!!! info "External Resources"
    - [Authorization Models, Per-Request Checks, and Safe Failure](https://cheatsheetseries.owasp.org/cheatsheets/Authorization_Cheat_Sheet.html) (OWASP)

## OIDC, SAML, and Enterprise Lifecycle

OIDC is often convenient for modern application integration; SAML remains appropriate where customer IdPs and enterprise contracts require it. Protocol choice alone does not settle tenant isolation or employee offboarding.

| Requirement | Design consequence |
|---|---|
| Customer-controlled login | Explicitly associate allowed issuers/metadata and account mappings with the customer tenant |
| Stable identity | Map issuer and protocol-defined subject; do not automatically link accounts by email alone |
| Removal from a customer organization | Reconcile provisioning/deprovisioning and application membership; decide how local sessions and API tokens are ended |
| Shared signing infrastructure across tenants | Validate the tenant/issuer context as well as the cryptographic signature |
| Enterprise groups or roles | Define a controlled mapping; do not accept arbitrary role claims as global administrator authority |

A SAML or OIDC login can create the same kind of local application session. Logout from the IdP is not proof that every application session, integration grant, and cached token has ended.

!!! info "External Resources"
    - [OIDC Core](https://openid.net/specs/openid-connect-core-1_0.html) (OpenID Foundation)
    - [SAML Security](https://cheatsheetseries.owasp.org/cheatsheets/SAML_Security_Cheat_Sheet.html) (OWASP)
    - [Tenant, Subject, and Actor Claim Validation](https://learn.microsoft.com/en-us/entra/identity-platform/claims-validation) (Microsoft; platform-specific example)

## Defending a Choice in an Interview

Explain your choice through its trust boundaries, failure modes, and operational tradeoffs.

Structure your answer as follows:

1. State actors, protected assets, trust boundaries, and assumptions.
2. Choose a grant/login mechanism, client proof, session/credential storage, and authorization model.
3. Trace issuance, validation, audience/tenant checks, and the most dangerous misuse path.
4. Explain expiry, rotation, revocation, recovery, and issuer/key outages.
5. Compare one realistic alternative and identify the requirement that would change your choice.
6. Describe observable evidence: rejection tests, audit fields without secrets, and revocation/failure tests.

**Short answer example:** “For a partner's unattended export, I would register a separate application identity and issue short-lived API access tokens with the minimum export permission. The API derives tenant access from trusted registration and policy, not a caller-provided tenant header. I would compare managed client credentials with workload federation based on the partner's platform, and use a supported sender constraint if token-only replay must be prevented.”

Practice with changed constraints: the client becomes a desktop tool; the export must run after its original employee leaves; the issuer is unavailable; a privileged token appears in a support attachment. Each change should alter a specific part of the answer.

!!! info "External Resources"
    - [Security Engineer Interview Preparation](https://amazon.jobs/content/en/how-we-hire/security-engineer-interview-prep) (Amazon)
    - [The Backend for Frontend Pattern](https://auth0.com/blog/the-backend-for-frontend-pattern-bff/) (Auth0)
    - [Identity Propagation Patterns](https://cheatsheetseries.owasp.org/cheatsheets/Identity_Propagation_Patterns_Cheat_Sheet.html) (OWASP)
