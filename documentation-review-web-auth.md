# Documentation review: Web Application Security and Authentication

Review date: 2026-10-13. Implementation date: 2026-10-13. Standard: `AGENTS.md`. Scope: all 11 Markdown files in `docs/web-security/` and `docs/authentication/` (897 source lines).

## Implementation Status

Findings 1–18 below have been addressed across all 11 scoped documentation pages. The original review is retained as a historical record; its source line numbers and descriptions refer to the pre-change content.

- Replaced the duplicated Authentication landing page with a topic map, conceptual distinctions, and learning objectives.
- Added protocol exchanges, validation checklists, lifecycle controls, and failure examples for sessions, JWT, OAuth/OIDC, SAML, WebAuthn, mTLS, and directory services.
- Corrected CORS, cookie scoping, cross-origin script behavior, storage access, and CSP limitations.
- Corrected API object/tenant authorization, GraphQL status handling, token transport, and distributed rate-limit guidance.
- Corrected injection controls, XSS categories, IMDSv2 mechanics, and resource references; separated prototype pollution from mass assignment.
- Added primary-source resource blocks to revised topic sections. No Astra validation is claimed.

## Original Assessment

These sections provide useful reference material, but they do not yet consistently meet the project's educational and technical accuracy standard. Correct the security misconceptions before expanding coverage. Authentication needs worked protocol flows and a real section landing page; web security needs more precise browser semantics and clearer separation of primary mitigations from additional controls.

Astra was requested and three review agents were started before the scope was narrowed. All failed with a usage-limit error before producing findings. This report is the primary agent's review, with selected claims checked against primary standards and official security guidance. It is not an Astra-validated review.

At review time, documentation source files had not been changed. They have since been revised as recorded above. Missing resource blocks on legacy pages are recorded as opportunities to support future revisions, not automatic formatting violations.

## Complete page coverage

| Page | Assessment | What works | Required improvement |
|---|---|---|---|
| `docs/web-security/index.md` | Major corrections | Useful origin table, header examples, CSP rollout steps, storage comparison, and relevant resources. | Correct CORS safelist, cookie scoping, cross-origin script execution, Cache Storage availability, and protection limits. Add a small request/preflight walkthrough. |
| `docs/web-security/tools.md` | Major corrections | Broad API comparison and useful rate-limiting dimensions and algorithm table. | Correct token transport/encryption guidance and GraphQL HTTP behavior. Explain ownership/tenant authorization, quota identity, and bounded distributed-counter behavior. |
| `docs/web-security/vulnerabilities.md` | Major corrections | Consistent root cause/impact/mitigation structure; useful breadth and dedicated XSS/injection sections. | Correct JWT-adjacent CSRF assumptions, XSS/SSTI mitigation claims, IMDSv2 mechanics, and broken source URL. Separate vulnerability classes that have different root causes. |
| `docs/authentication/index.md` | Replace landing content | Its session explanation is readable. | It is byte-for-byte identical to `sessions.md`, with no section overview or learning path. Replace with an authentication overview and topic navigation. |
| `docs/authentication/sessions.md` | Needs improvement | Clear session lifecycle and useful cookie-attribute summary. | Qualify SameSite and HttpOnly protections. Explain entropy, idle/absolute expiry, privilege-change rotation, logout invalidation, and cookie scope with one concrete exchange. |
| `docs/authentication/jwt.md` | Major corrections | Introduces claims, expiry, permission staleness, and asymmetric signing. | Signature verification is presented as sufficient; replay and token structure explanations are incomplete. Add a validation checklist and rejected-token example. |
| `docs/authentication/oauth2-oidc.md` | Major corrections | Correctly distinguishes delegated authorization from the identity layer and identifies the four principal roles. | Names flows without teaching them; overstates PKCE requirements and IdP outage effects. Explain token purposes, front/back channels, request binding, and refresh-token protection. |
| `docs/authentication/saml.md` | Major corrections | Introduces IdP/SP roles, metadata, replay, and XML signature wrapping. | Granting access after signature verification alone is unsafe guidance. Explain complete assertion validation and binding-specific protections. |
| `docs/authentication/mtls.md` | Needs improvement | Useful machine-to-machine use cases and certificate lifecycle concerns. | Explain proof of private-key possession, service identity matching, authorization, termination boundaries, and version-specific handshake ordering. |
| `docs/authentication/webauthn.md` | Major corrections | Clear registration/authentication split and attention to recovery. | Assumes universal hardware protection and PIN/biometric verification. Explain synced credentials, RP ID versus origin, server-side validation, and user presence versus verification. |
| `docs/authentication/directory-services.md` | Major corrections | Introduces directory access, Kerberos tickets, and relevant attacks. | Qualify LDAP bind methods, correct Golden Ticket recovery guidance, and distinguish SSO authentication from authorization. Expand ticket/session-key mechanics. |

## Priority 1: misleading security models and defenses

### 1. JWT signature verification is not a complete acceptance policy

Locations: `docs/authentication/jwt.md:20`, `:34`, `:38`, `:44`.

The walkthrough grants trust after signature verification. Later notes mention issuer/audience, but the primary flow never requires the complete validation policy. Rejecting only `alg: none` also leaves the algorithm policy underspecified. Reading a unique `jti` and checking expiry does not independently stop reuse during the validity window.

Correction: configure allowed algorithms and trusted issuer keys independently of token input; validate token purpose, issuer, audience, time bounds, and application-required claims before making a separate authorization decision. Explain that preventing reuse needs state or an appropriate sender-binding mechanism, while expiry only bounds exposure. Add a valid-signature/wrong-audience rejection example. Sources: [JWT best current practices, sections 3.1 and 3.8–3.12](https://www.rfc-editor.org/info/rfc8725/), [JWT ID claim](https://www.rfc-editor.org/rfc/rfc7519.html#section-4.1.7), and [OAuth replay protection](https://www.rfc-editor.org/rfc/rfc9700.html#section-2.2).

### 2. SAML signature verification alone must not grant access

Locations: `docs/authentication/saml.md:11`, `:25`, `:28`.

The main flow jumps directly from signature verification to access. The wrapping defense does not explicitly require consuming the exact signed content. Switching to POST is presented as the mitigation for insecure redirects, although binding choice alone does not establish trust or validate a redirect target.

Correction: teach a library-based validation checklist covering trusted issuer/key, signed content actually consumed, intended audience/recipient/destination as applicable, time conditions, request correlation for SP-initiated SSO, and replay handling. Treat unsolicited SSO separately. Validate RelayState/return targets and explain binding-specific requirements. Source: [OWASP SAML Security](https://cheatsheetseries.owasp.org/cheatsheets/SAML_Security_Cheat_Sheet.html).

### 3. The CORS simple-request definition is incorrect

Locations: `docs/web-security/index.md:25`, `:89`, `:90`, `:97`.

The page lists `application/json` and all `text/*` MIME types as simple-request types. This gives learners the wrong prediction for common API requests. The surrounding explanation also overstates when cross-origin requests can be sent and conflates authorizing credentials with allowing an `Authorization` request header.

Correction: describe safelisted methods and headers, including value restrictions. The safelisted Content-Type essences are `text/plain`, `application/x-www-form-urlencoded`, and `multipart/form-data`; JSON usually triggers preflight in a cross-origin fetch. Explain preflight caching, explicit Authorization header approval, and the credential mode separately. `Access-Control-Allow-Origin` names one origin or a wildcard, not a list of origins. Include a JSON POST/OPTIONS exchange. Source: [WHATWG Fetch CORS protocol and header safelist](https://fetch.spec.whatwg.org/#cors-safelisted-request-header).

### 4. The browser-isolation model misstates cookies and third-party scripts

Locations: `docs/web-security/index.md:27`, `:35`, `:36`, `:178`, `:209`.

Cookies are grouped with origin-keyed storage even though their scope differs, notably across ports. The CDN row suggests SOP makes loading a library safe by blocking reads; a classic cross-origin script instead executes with the embedding page's privileges. `postMessage` exchanges messages without granting general cross-origin DOM access. Cache Storage is incorrectly restricted to service workers.

Correction: separate cookie host/domain/path and Secure/SameSite rules from origin storage; do not present Path as a security boundary. Describe trusted third-party script execution and relevant controls. Explain message origin/source validation. State that Cache Storage is exposed to both Window and Worker contexts. Sources: [cookie isolation limits](https://www.rfc-editor.org/rfc/rfc6265.html#section-8.5), [cross-origin script fetching](https://fetch.spec.whatwg.org/), and [CacheStorage exposure](https://www.w3.org/TR/service-workers/#cachestorage-interface).

### 5. XSS and SSTI controls are presented as interchangeable or universal

Locations: `docs/web-security/vulnerabilities.md:108`, `:134`, `:150`; `docs/web-security/index.md:234`.

Contextual output encoding is called universal, while the SSTI row recommends escaping input without distinguishing template source from rendered data. This can teach a learner to apply an HTML escaping control to code that has already been interpreted by a template engine.

Correction: teach safe sinks, contextual output encoding, sanitization for intentionally allowed HTML, and avoiding dangerous insertion contexts. For SSTI, keep untrusted data out of template source and pass it as values to fixed templates; explain sandbox limitations. CSP, WAF, and least privilege are additional layers rather than replacements for fixing the injection boundary. Sources: [OWASP XSS Prevention](https://cheatsheetseries.owasp.org/cheatsheets/Cross_Site_Scripting_Prevention_Cheat_Sheet.html) and [OWASP SSTI Prevention](https://cheatsheetseries.owasp.org/cheatsheets/Server_Side_Template_Injection_Prevention_Cheat_Sheet.html).

### 6. Cookie/token advice overstates protection from XSS and CSRF

Locations: `docs/authentication/sessions.md:19`, `:34`; duplicated in `docs/authentication/index.md`; `docs/authentication/jwt.md:36`; `docs/web-security/vulnerabilities.md:38`; `docs/web-security/index.md:216`.

SameSite=Lax does not exclude every cross-site request. HttpOnly reduces direct cookie extraction but does not prevent malicious same-origin script from issuing authenticated actions. REST is not a CSRF defense: the relevant distinction is how credentials are attached and how state-changing requests are validated.

Correction: distinguish site from origin and Lax from Strict; preserve safe-method semantics and apply appropriate CSRF validation. Explain synchronizer or signed, session-bound double-submit token patterns. When recommending cookie storage, include the resulting CSRF implications and state that HttpOnly does not neutralize XSS. Sources: [OWASP CSRF Prevention](https://cheatsheetseries.owasp.org/cheatsheets/Cross-Site_Request_Forgery_Prevention_Cheat_Sheet.html) and [OWASP Session Management](https://cheatsheetseries.owasp.org/cheatsheets/Session_Management_Cheat_Sheet.html).

### 7. WebAuthn is incorrectly described as universally device-bound hardware authentication

Locations: `docs/authentication/webauthn.md:9`, `:10`, `:13`, `:23`, `:29`, `:30`.

The page says private keys never leave secure hardware and authenticators require a PIN or biometric. These statements do not cover multi-device credentials or authentication without required user verification. RP ID and origin are treated as interchangeable.

Correction: distinguish device-bound and backup-eligible credentials, platform and roaming authenticators, and user presence versus user verification. Teach challenge freshness/consumption and server validation of ceremony type, expected origin, RP ID hash, signature, and required flags. The relying party must enforce its verification policy. Source: [W3C WebAuthn, backup state and ceremony verification](https://www.w3.org/TR/webauthn-3/).

### 8. Token encryption advice omits the mandatory transport-security boundary

Location: `docs/web-security/tools.md:67`.

The instruction to sign and also encrypt tokens crossing untrusted networks does not explain TLS and implies application-level token encryption is universally required.

Correction: require correctly validated HTTPS for bearer-token transport; explain that message-level encryption depends on claim confidentiality and intermediary trust. Signed tokens can be appropriate over TLS, while encrypted bearer tokens can still be stolen and replayed. Do not let encryption replace issuer/audience/signature validation. Source: [RFC 6750 security recommendations](https://www.rfc-editor.org/rfc/rfc6750.html#section-5.2).

### 9. Golden Ticket mitigation is too weak to guide recovery

Locations: `docs/authentication/directory-services.md:34`, `:40`.

Regular `krbtgt` rotation is listed as the mitigation without explaining compromised-domain recovery or retention of the previous key. The SSO table also equates authentication with access to the entire domain.

Correction: explain that recovery includes removing attacker persistence and a controlled two-reset process with replication and ticket-lifetime considerations; link to the operational runbook rather than giving an unsafe shortcut. State that service authorization still applies after SSO. Source: [Microsoft AD forest recovery: reset the krbtgt password](https://learn.microsoft.com/en-us/windows-server/identity/ad-ds/manage/forest-recovery-guide/ad-forest-recovery-reset-the-krbtgt-password).

## Priority 2: protocol precision, coverage, and educational structure

### 10. Replace the duplicated Authentication landing page

Location: `docs/authentication/index.md:1` (entire page).

Verified byte-for-byte duplication of `sessions.md`. Replace it with learning objectives and links to all seven topic pages. Distinguish authentication, authorization, session management, token representation, federation, and transport identity. Recommend a sequence: sessions → JWT → OAuth/OIDC → SAML → WebAuthn; mTLS and directory services as service/enterprise branches. Explain that JWT and cookies are not mutually exclusive authentication protocols.

### 11. Expand OAuth/OIDC from flow names into a complete exchange

Locations: `docs/authentication/oauth2-oidc.md:19`, `:21`, `:29`, `:38`; `docs/web-security/tools.md:79`.

Add an authorization-code sequence that identifies browser redirects, token endpoint exchange, PKCE S256, and the distinct jobs of state and nonce. Separate ID tokens, access tokens, and refresh tokens; explain that ID tokens are for the client and not API bearer credentials. Include issuer/audience/time validation and identity mapping by issuer plus subject. Qualify outage behavior: existing local sessions and locally validated tokens may continue depending on policy and key availability.

Replace “mandatory for almost all clients” with the precise rule: public clients must use PKCE; confidential clients are recommended to use it under RFC 9700. Link the published RFC rather than the superseded draft. Explain refresh-token rotation or sender constraints for public clients. Sources: [RFC 9700](https://www.rfc-editor.org/rfc/rfc9700.html#section-2.1.1) and [OIDC Core validation](https://openid.net/specs/openid-connect-core-1_0.html#IDTokenValidation).

### 12. Qualify JWT structure and teach signing accurately

Locations: `docs/authentication/jwt.md:7`, `:11`, `:13`, `:19`.

Label the three-part example as compact JWS, not every possible JWT. Explain MAC versus asymmetric signature without describing both as hashing with a key. Distinguish Base64URL encoding from encryption, and distinguish cookies automatically attached by a browser from an application explicitly setting a bearer header. Source: [RFC 7519 overview](https://www.rfc-editor.org/rfc/rfc7519.html#section-3).

### 13. mTLS needs identity and authorization beyond a trusted CA

Locations: `docs/authentication/mtls.md:10`, `:12`, `:13`, `:35`.

The abbreviated flow is acceptable only when marked as conceptual and version-dependent. Explain CertificateVerify proof of key possession, certificate validity and intended usage, expected service identity, trust-domain scoping, and the separate authorization decision. Add a concrete trusted-but-wrong-service rejection scenario and describe what identity survives a TLS-terminating proxy. Source: [TLS 1.3 certificate authentication](https://www.rfc-editor.org/rfc/rfc8446.html#section-4.4).

### 14. LDAP and Kerberos need explicit assumptions

Locations: `docs/authentication/directory-services.md:10`, `:16`, `:25`, `:32`.

Label DN/password bind as a simple-bind example, not the entire LDAP authentication model. Explain SASL alternatives and fail-closed TLS/server-identity validation. Show where the client obtains a session key and why a ticket must be accompanied by freshness/authenticator evidence; do not imply the client decrypts its TGT. Describe Kerberoasting as offline guessing against encrypted ticket material rather than receipt of a password hash. Sources: [LDAP authentication methods](https://www.rfc-editor.org/rfc/rfc4513.html) and the additional Kerberos specification recommended for the revision, RFC 4120 (not independently inspected in this review).

### 15. Correct GraphQL generalizations and explain primary API authorization

Locations: `docs/web-security/tools.md:10`, `:12`, `:22`, `:38`, `:40`.

GraphQL does not always return HTTP 200; behavior depends on the failure phase and response media type. Fixed URI/method checks are also insufficient to authorize access to a particular REST object. Explain ownership, tenant boundaries, and property-level checks in both models. Present introspection restrictions as deployment-dependent exposure reduction, not the primary access control. Avoid asserting a universal resolver-call complexity for nested queries. Sources: [GraphQL HTTP behavior](https://graphql.org/learn/serving-over-http/) and [OWASP GraphQL security](https://cheatsheetseries.owasp.org/cheatsheets/GraphQL_Cheat_Sheet.html).

### 16. Separate vulnerability mechanisms and qualify mitigation lists

Locations: `docs/web-security/vulnerabilities.md:31`, `:59`, `:75`, `:100`, `:103`, `:129`.

Keep object-level authorization as the primary IDOR control; unpredictable IDs only reduce enumeration. Replace stripping `..` with a precise containment/allow-list model and note normalization/symlink assumptions. Separate mass assignment from prototype pollution. DOM-based XSS describes the execution path and can overlap with stored/reflected delivery; it does not inherently mean the payload never reached a server. Split mutation XSS and universal XSS rather than presenting them as synonyms. For NoSQL, teach scalar/type validation and preventing attacker-controlled operators instead of generic stringification.

Sources inspected for the key distinctions: [OWASP IDOR Prevention](https://cheatsheetseries.owasp.org/cheatsheets/Insecure_Direct_Object_Reference_Prevention_Cheat_Sheet.html), [OWASP XSS classification](https://owasp.org/www-community/Types_of_Cross-Site_Scripting), and [OWASP Prototype Pollution Prevention](https://cheatsheetseries.owasp.org/cheatsheets/Prototype_Pollution_Prevention_Cheat_Sheet.html). The detailed path/NoSQL and mutation/UXSS replacement text needs additional source verification when implemented.

### 17. Explain IMDSv2 correctly and bound the SSRF mitigation claim

Location: `docs/web-security/vulnerabilities.md:167`.

The required PUT header specifies token lifetime, not a hop limit. The metadata token response has a separately configured IP hop limit. Explain token acquisition and subsequent token-bearing requests, require IMDSv2 where relevant, and describe it as an additional control against particular SSRF capabilities. URL/redirect/IP validation and egress restrictions remain necessary. Sources: [AWS IMDSv2 mechanics](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/configuring-instance-metadata-service.html) and [OWASP SSRF Prevention](https://cheatsheetseries.owasp.org/cheatsheets/Server_Side_Request_Forgery_Prevention_Cheat_Sheet.html).

### 18. Make rate-limiting examples operationally meaningful

Locations: `docs/web-security/tools.md:96`, `:97`, `:109`, `:115`, `:136`.

The fixed-window row correctly notes edge bursts, but the recommendation to use it to “strictly cap” traffic contradicts that limitation. Explain the maximum boundary burst with a numeric example. Quotas should follow the stable abuse/billing identity when rotating an API key must not reset them. Treat IP/subnet grouping as a deployment-dependent signal with NAT/IPv6 tradeoffs. Add atomic update/expiry behavior, concurrency-permit cleanup, and failure policy rather than prescribing a shared store as the only distributed design. These are educational design recommendations; concrete product behavior requires implementation-specific verification.

## Resources, examples, and tone

- All eight Authentication files have zero external links and no fenced code examples. This does not automatically violate a retroactive formatting rule, but it leaves protocol claims unsupported and mechanics hard to practice. Add directly relevant resource blocks when revising each topic; do not fill a quota.
- `docs/web-security/vulnerabilities.md:120`: the XSS Prevention URL using `Cross-Site_Scripting` returned HTTP 404 during this review. The working [OWASP page uses `Cross_Site_Scripting`](https://cheatsheetseries.owasp.org/cheatsheets/Cross_Site_Scripting_Prevention_Cheat_Sheet.html).
- `docs/web-security/tools.md:79`: replace the OAuth security draft link with published RFC 9700.
- `docs/web-security/tools.md:154`: the Google Cloud link redirects to a broader [traffic/load management guide](https://docs.cloud.google.com/architecture/infra-reliability-guide/traffic-load). Update the title/target or choose a more specific algorithm reference.
- `docs/web-security/vulnerabilities.md:92`: identify the CWE Top 25 list as the 2023 edition if retained; do not imply the archived list is current.
- Preserve the useful tables and checklists. Replace metaphors and slogans such as “identity as the new perimeter,” “opens everyone,” and “not a vault” with concrete trust boundaries and failure modes. This is a lower priority than the accuracy fixes.
- For each substantially revised concept page, include a short worked success/failure example with expected results. This is a review recommendation for educational effectiveness, not an existing blanket requirement to add labs everywhere. Use inert identifiers and bounded examples; document prerequisites when commands are included.

## Suggested remediation order and acceptance checks

1. Correct Priority 1 findings without broad stylistic churn.
2. Replace the Authentication landing page and expand JWT, OAuth/OIDC, SAML, and WebAuthn flows and validation rules.
3. Add the remaining session, mTLS, directory, API, and SSRF explanations; fix checked resource links.
4. Review each changed section for assumptions, primary versus additional controls, one useful example where warranted, and directly relevant resources.
5. Run `mkdocs build --clean`, inspect diagnostics, check changed links/anchors, and verify newly added external references. Do not treat a passing build as evidence of security accuracy.

## Validation and limits

- All 11 scoped files were read; the Authentication index/session duplication was confirmed by byte comparison.
- A repository-wide relative-file-target scan found no missing target files. It did not validate every anchor or external URL.
- MkDocs 1.6.1 built successfully in a temporary virtual environment reusing installed packages, with output directed to `/private/tmp/security-docs-review-site`. No dependencies were installed or pinned and this was not a fresh dependency-reproducibility check.
- The build reported three missing anchors in out-of-scope `docs/notes.md`: `#infrastructure-prod--cloud-virtualisation`, `#malware--reversing`, and `#coding--algorithms`. No scoped-page link diagnostics were emitted.
- Source verification was targeted to the findings, not an exhaustive audit of all 30 existing external links or every sentence. Browser behavior was checked against specifications, not reproduced across browser versions. Examples and security scenarios were not executed.
- No findings for other documentation sections are included. Prior changes to `AGENTS.md`/`.gitignore` and the unrelated untracked `tools.yaml` are preserved.

## Implementation Validation

- `mkdocs build --clean` succeeds using the temporary isolated environment and output directory documented above. The three pre-existing `notes.md` anchor diagnostics remain outside this scope.
- All 11 rendered pages have one H1 and rendered information/resource blocks. Code fences are balanced and language-labeled.
- All 33 source Markdown links targeting the scoped pages, including incoming links and local fragments, resolve against the rendered page IDs.
- The SQLite parameterization example was executed in an in-memory database: a matching ID returned the expected row, while SQL-like input remained a value and returned no match.
- New sources were inspected through direct web access, including OAuth/PKCE, SAML Core/Bindings, WebAuthn, PKIX/SPIFFE, Kerberos, and the relevant OWASP guidance. This does not claim exhaustive external-link or browser-version testing.
- `git diff --check` passes. Documentation and the review record are the only new changes from this implementation; earlier instruction changes and unrelated files remain preserved.
