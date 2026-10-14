# Authentication Design Patterns

Seven worked designs explain the choice, trust boundaries, alternatives, and failure cases. Use the [architecture comparison](comparison.md) to compare options and the protocol pages for validation details.

## 1. Expose an Application to an External Machine Client

**Prompt:** A SaaS application already supports employee browser login. A customer's scheduled process now needs to export its own reports. Design access without sharing an employee password or browser session.

**Assumptions:** The process runs on a customer-controlled server, has no interactive user during execution, and can protect a credential or present a supported workload identity. Exports are tenant-scoped and read-only.

```text
Customer daemon -- application proof --> Authorization server
Customer daemon <-- scoped access token -- Authorization server
Customer daemon -- API token ----------> Reports API
Reports API ---- identity + tenant + permission checks ----> Report data
```

**Decision:** Add a dedicated API using an application identity and the client-credentials grant. Register separate customer/integration identities and bind their allowed tenant and actions through server-controlled policy. A browser session remains a different interface for humans.

1. Onboard the integration through an authorized tenant administrator; record owner, environment, permissions, and rotation/revocation contacts.
2. Authenticate the daemon at the token endpoint using supported managed credentials, a signed client assertion, or mTLS. A supported workload-federation alternative may remove stored credentials, but can use a different grant/profile; follow that provider's contract rather than calling every assertion exchange client credentials.
3. Issue a short-lived access credential for the reports API and minimum report permission.
4. At the API, validate the token's intended recipient and application identity, then authorize the requested tenant and report. The permission to call exports is not permission to export every tenant.
5. Enforce per-owner quotas, export size/cost limits, audit events, and response-cache isolation. Never log raw credentials.

**Attack to reject:** A correctly authenticated customer A changes `/tenants/A/reports` to `/tenants/B/reports`. The API rejects the object/tenant access even if the token has `reports.read`. A tenant header is input, not an authority source.

| Alternative | When it fits | Tradeoff |
|---|---|---|
| Per-integration API key | Small, non-delegated API with a manageable bearer-credential lifecycle | Simpler integration; long-lived keys create exposure unless rotation and revocation are reliable |
| mTLS-only identity | A tightly controlled partner service with certificate-based authorization | Avoids an OAuth issuer but still needs certificate lifecycle and application policy |
| Client credentials plus certificate-bound access tokens | Token-only replay is a material concern and all components support binding | API must validate the bound certificate; token-endpoint mTLS alone does not create this property |
| User authorization code grant | The tool genuinely acts under an individual user's delegation | Consent, offboarding, and refresh-token lifecycle differ from application-only access |

**Follow-up:** “Can I revoke the client immediately?” Explain three actions separately: stop new issuance, end renewable grants if any, and make already-issued credentials unusable. Local JWT validation needs an additional revocation check or an accepted expiry bound; disabling a client record alone may not invalidate all outstanding tokens.

**Answer to defend:** Choose application identity because execution is unattended and authorized independently of a human session. Do not choose client credentials merely because the caller is written in Python; a human's Python CLI is a different scenario.

!!! info "External Resources"
    - [OAuth Client Credentials Grant](https://www.rfc-editor.org/rfc/rfc6749.html#section-4.4) (IETF)
    - [Signed Client Assertions](https://www.rfc-editor.org/rfc/rfc7523.html#section-2.2) (IETF)
    - [Certificate-Bound Access Tokens](https://www.rfc-editor.org/rfc/rfc8705.html#section-3) (IETF)

## 2. Protect a Browser Application with a Backend for Frontend

**Prompt:** A browser dashboard calls sensitive APIs. A developer proposes keeping a refresh token in localStorage so page reloads preserve login.

**Assumptions:** The team can operate a backend and route browser API requests through it. Same-origin UI/BFF deployment is feasible; cross-origin deployment would add cookie and CORS considerations.

```text
Browser -- OIDC redirects via BFF --> Identity provider
BFF ----- code + PKCE + client proof --> Token endpoint
Browser -- opaque session cookie -----> BFF
BFF ----- API access token -----------> Resource API
```

**Decision:** Use a Backend for Frontend (BFF) as the confidential OAuth/OIDC client and keep access/refresh credentials server-side. The browser receives an opaque, Secure, HttpOnly session cookie. Login request binding, API authorization, and CSRF protections remain mandatory.

**Protection priorities:**

- Validate the login response and rotate the local session identifier. Tie stored API credentials to the correct user/session and intended API.
- Apply CSRF controls to cookie-authenticated operations and logout. SameSite is an additional control, not a complete proof of request legitimacy.
- Make proxy destinations and routes explicit. A generic URL-forwarding endpoint can become an SSRF path or send tokens to an attacker's server.
- Limit session/token-store access, redact observability data, and invalidate the local session appropriately when its grant is unusable.
- Fix XSS through safe rendering and apply CSP. HttpOnly prevents direct cookie reads, but malicious same-origin code can still operate the user's BFF session.

**Tradeoff:** The BFF adds infrastructure, latency, and capacity requirements. A standalone SPA using code + PKCE avoids that proxy, but exposes access credentials to its JavaScript environment. A token-mediating backend keeps refresh credentials server-side while releasing access tokens to the browser; it does not have identical exposure to a full BFF.

**Follow-up:** “Does DPoP make XSS harmless?” No. It can restrict off-device replay of a stolen token, but malicious code in an authorized runtime may request proofs or perform actions. Define what the key store and execution environment actually prevent.

**Rejection tests:** An arbitrary upstream URL is denied; a cross-site state-changing request without valid CSRF evidence fails; a token from user A's server session is never reused for user B.

!!! info "External Resources"
    - [Browser-Based Application Architectures](https://www.rfc-editor.org/rfc/rfc10017.html#section-6) (IETF)
    - [The Backend for Frontend Pattern](https://auth0.com/blog/the-backend-for-frontend-pattern-bff/) (Auth0)
    - [DPoP Security Considerations](https://www.rfc-editor.org/rfc/rfc9449.html#section-11) (IETF)

## 3. Preserve User Delegation Across Service Calls

**Prompt:** A user calls an orders service, which calls billing. Should orders forward the incoming token, use its powerful billing service account, or exchange the token?

**Assumptions:** Billing must know both the user and the service acting for them. The issuer supports an explicit delegation profile and audience-specific credentials.

```text
User -- token intended for orders --> Orders
Orders -- subject token + service authentication + target --> STS
Orders <-- billing-specific delegated token ---------------- STS
Orders -- delegated token --> Billing
```

**Decision:** Use a supported token-exchange profile when it expresses the required delegation. The Security Token Service (STS) validates the input and the calling service, then applies policy for the downstream audience, permissions, and lifetime. Exchange is not automatic permission narrowing: the issuer must enforce it.

A delegated JWT's claims could include:

```json
{
  "iss": "https://issuer.example",
  "aud": "billing-api",
  "sub": "user-42",
  "act": { "sub": "orders-service" },
  "scope": "invoice.read"
}
```

The fragment omits the signature, expiry, and other required claims; trusted key selection and the profile's validation rules still apply. `act` has defined delegation semantics; arbitrary client-supplied `X-User` headers do not. Actor representation and client authentication depend on the chosen provider/profile.

**Authorization:** Billing applies user entitlement, the current actor's permitted delegated capability, tenant/resource policy, and token restrictions. A user allowed to view an invoice does not thereby authorize every service to retrieve it. Keep subject, actor, request ID, and decision in the audit record without storing the token.

**Why alternatives differ:** Forwarding an orders-only token should fail billing's audience check. Making one multi-audience token valid everywhere broadens replay exposure. Calling billing with orders' unrestricted application authority can lose the user's constraints and create a confused-deputy problem. Application-only authority is appropriate for a genuine system task, not as a fallback after user authorization fails.

**Follow-up:** “The downstream call is queued for tomorrow.” Do not persist an expired user token as the job's authorization. Define durable job approval, cancellation and entitlement checks, and narrowly permitted worker identity. For irreversible operations, decide whether authority is fixed at approval time or must be reconfirmed at execution.

!!! info "External Resources"
    - [Token Exchange and Delegation Semantics](https://www.rfc-editor.org/rfc/rfc8693.html#section-1.1) (IETF)
    - [Identity Propagation Patterns](https://cheatsheetseries.owasp.org/cheatsheets/Identity_Propagation_Patterns_Cheat_Sheet.html) (OWASP)

## 4. Authenticate a Human CLI Separately from a Daemon

**Prompt:** Engineers need a command-line tool for production diagnostics. The same binary will also run in scheduled automation.

**Decision:** Give the interactive and unattended modes distinct credential models. Sharing a binary does not mean sharing a principal.

| Mode | Starting choice | Protection |
|---|---|---|
| Local interactive CLI with a browser | Native-app authorization code + PKCE using an external user agent and a supported callback | Validate the redirect and transaction, protect local credential storage, preserve the user's restricted permissions |
| Device or remote environment with unsuitable browser/callback support | Device authorization grant if supported; the user approves on another browser-capable device | Display provider/client context, protect the device code, enforce expiry and polling rules |
| Unattended schedule | Dedicated workload identity or confidential-client application credentials | Restrict the job's purpose and permissions independently of an employee's login |

For a device flow, the tool obtains a device code and user-facing verification instructions. The user authorizes the request at the provider; the tool polls with the device code until completion or expiry. It must honor the returned interval and slowdown/error responses. An embedded “client secret” distributed to every CLI user is not confidential.

**Attack to discuss:** An attacker starts a legitimate device authorization and persuades a victim to approve its code. Authentication at the real IdP can succeed while authorizing the wrong device/client. Phishing-resistant login alone does not prove that the user intended this authorization. Show meaningful client/action context and teach users to initiate their own flows; do not claim short code expiry eliminates real-time phishing.

**Tradeoff:** Device flow works around input constraints but introduces an approval interaction that can be socially engineered. Prefer the normal browser code flow where the native app's supported platform integration makes it practical. Store renewable credentials in an OS-supported secure store and define logout/revocation; do not keep them in shell history or diagnostic logs.

**Follow-up:** “The employee leaves the company.” Revoke the person's access and renewable grants according to policy. A business-critical schedule should already use an owned application identity so continuity does not depend on retaining an employee account.

!!! info "External Resources"
    - [OAuth for Native Apps](https://www.rfc-editor.org/rfc/rfc8252.html) (IETF)
    - [Device Flow and Remote Phishing](https://www.rfc-editor.org/rfc/rfc8628.html#section-5.4) (IETF)

## 5. Add Enterprise SSO to a Multi-Tenant SaaS

**Prompt:** Each customer wants its own IdP, automatic onboarding, and immediate removal of former employees. One customer sends an administrator role in a token. Should the SaaS trust it?

**Assumptions:** Customers control authentication for their organization, but must never control other tenants' identity mappings or application-global privileges.

**Decision:** Associate each approved federation configuration with a customer tenant. Use OIDC or SAML based on integration requirements, then issue a local application session bound to the resolved identity and tenant membership.

1. Authorize and verify configuration changes before trusting a customer's issuer, metadata, signing keys, or domain routing. Email suffixes alone do not authorize federation onboarding.
2. Route login through known tenant configuration. Validate the response under that context, including recipient/audience, issuer, signature, time, and request binding.
3. Map the stable protocol identity to the application's account and membership. Do not auto-merge a privileged local account with any new IdP account that presents the same email.
4. Map external groups/roles through an explicit, tenant-scoped policy. A customer's `admin` value might mean that customer's administrator; it must not mean global SaaS operator.
5. Treat provisioning, deprovisioning, group changes, local sessions, and API grants as separate lifecycle components. Define reconciliation and stale-membership behavior.

**Attack to reject:** A valid token from customer A arrives on customer B's callback or selects tenant B in a request body. A signature under shared provider keys is insufficient; identity and membership must resolve to the intended tenant context.

**Tradeoff:** Just-in-time account creation reduces onboarding work but is not deprovisioning. Provisioning integrations improve lifecycle control but add synchronization delays and failure modes. Define a measurable offboarding bound and a manual emergency path rather than promising immediate removal from SSO alone.

**Follow-up:** “The IdP is down.” Decide whether existing local sessions can continue and which sensitive operations require current membership or fresh authentication. Do not enable a weaker global password fallback or disable issuer validation without a designed recovery policy.

!!! info "External Resources"
    - [OIDC Validation and Subject Claims](https://openid.net/specs/openid-connect-core-1_0.html#IDTokenValidation) (OpenID Foundation)
    - [Tenant, Subject, and Actor Validation](https://learn.microsoft.com/en-us/entra/identity-platform/claims-validation) (Microsoft; platform-specific example)
    - [SAML Security](https://cheatsheetseries.owasp.org/cheatsheets/SAML_Security_Cheat_Sheet.html) (OWASP)

## 6. Replace a CI Deployment Secret with Workload Federation

**Prompt:** A CI pipeline stores a long-lived cloud key. Remove that key without allowing any repository or pull request to obtain production access.

**Assumptions:** The CI platform issues workload assertions and the cloud provider supports federation. The deployment role is limited to a defined application/environment.

```text
CI job -- short-lived signed workload assertion --> Cloud STS
Cloud STS -- verifies issuer, audience, workload conditions --> Temporary role
CI job -- temporary role credentials --> Deployment API
```

**Decision:** Configure a narrowly scoped trust relationship and temporary deployment role. The job's right to request an assertion is not itself cloud authorization; the cloud trust policy decides which assertion is acceptable.

For GitHub Actions, evaluate issuer, intended audience, and subject/workflow conditions supported by the provider. A subject using `environment:production` is not the same as a branch-form subject. Protect the production environment's allowed deployment refs and approvals separately; do not assume an environment-form subject also contains a branch restriction.

| Control | Threat it addresses |
|---|---|
| Exact repository/workload and audience trust conditions | Another project obtaining credentials under the same CI issuer |
| Protected workflow and environment rules | An untrusted change reaching an authorized production job |
| Least-privilege deployment role | A valid deployment credential granting account-wide administration |
| Controlled dependencies and reusable workflows | Compromised actions executing inside a trusted job |
| Isolated runners and redacted logs | Token/credential theft from execution state or artifacts |

**Tradeoff:** Federation removes a stored cloud key but retains the CI trust boundary. An attacker who controls an authorized workflow may legitimately obtain temporary credentials. Short lifetime reduces persistence; it does not prevent damage during a permitted run.

**Rejection tests:** Another repository, wrong audience, and an unauthorized deployment context must fail role assumption. Test how fork pull requests and reusable workflows actually populate claims; avoid a wildcard trust rule introduced merely to make an example succeed.

**Follow-up:** “Production access must be stopped now.” Block future federation and revoke or constrain outstanding sessions through provider-supported controls. Token and temporary-session expiry are separate timers. Investigate the trusted job and runner before restoring access.

!!! info "External Resources"
    - [GitHub Actions Workload Federation](https://docs.github.com/en/actions/concepts/security/openid-connect) (GitHub)
    - [OIDC Claims and Subject Context](https://docs.github.com/en/actions/reference/security/oidc) (GitHub)

## 7. Require Stronger Authentication for Sensitive Actions

**Prompt:** An administrator can read dashboards after login, but changing a payout destination or adding another administrator needs stronger evidence. What happens if their session is stolen?

**Decision:** Separate ordinary session continuity from recent, policy-approved authentication for sensitive operations. Define the authentication context required by the action, validate trustworthy evidence server-side, and bind any resulting approval to the intended operation.

1. Identify the action and authority required; authorize the user before presenting a step-up request.
2. Require sufficiently recent authentication with an accepted method/context, potentially WebAuthn. Check the provider's assurance semantics rather than assuming every “MFA” label means phishing resistance.
3. Validate the step-up response and retain server-controlled evidence of freshness and scope. A client-side `mfa_complete=true` flag is not evidence.
4. Revalidate the target operation and its parameters at execution. A recent login alone is not a cryptographic signature of a payment's amount or destination.
5. Audit elevation, action completion, recovery, and credential enrollment without collecting authenticators or tokens.

**Concrete operation binding:** The application creates a server-owned pending operation with an unpredictable approval ID, the acting user/account, action, target, exact destination/amount where applicable, and an expiry. The step-up transaction references that record. After validating the returned authentication evidence, the server marks only that operation approved. Execution checks the same stored parameters and current policy and consumes the approval once, coordinating the state transition with the operation; an external side effect needs idempotency/recovery handling too. Editing the destination or amount invalidates the approval rather than inheriting the earlier success.

This is server-side binding of authority to an operation. A browser confirmation screen is not a trusted display against browser malware, and a normal WebAuthn login does not itself sign the payment parameters. Stronger transaction-confirmation requirements need a separate, explicit design.

When the OAuth ecosystem supports RFC 9470, an API can signal insufficient user authentication and request context/recency. The client obtains suitable new authorization evidence, and the API verifies the returned credential. A BFF/local-session design can enforce step-up through its own supported server-side session policy instead.

**Tradeoff:** More prompts create friction; broad elevated sessions create misuse windows. Choose action-specific freshness and privilege bounds. Recovery and enrollment require comparable protection: a weak help-desk reset can undermine a strong primary login method.

**Attack to discuss:** Malware uses an already authenticated browser. FIDO2 resists a fake-origin login, but does not automatically prevent actions through that active session. Explain session protection, action authorization, parameter binding where required, and emergency invalidation.

**Follow-up:** “Can customer support bypass step-up?” Define a narrowly authorized, audited recovery workflow with verification and approval controls. Do not treat support access as unrestricted impersonation; preserve the support actor separately from the affected account.

!!! info "External Resources"
    - [OAuth Step-Up Authentication Challenges](https://www.rfc-editor.org/rfc/rfc9470.html) (IETF)
    - [Authentication and Reauthentication Guidance](https://cheatsheetseries.owasp.org/cheatsheets/Authentication_Cheat_Sheet.html) (OWASP)
    - [WebAuthn Security Considerations](https://www.w3.org/TR/webauthn-3/#sctn-security-considerations) (W3C)

## Practice Under Changed Constraints

For a mock interview, spend a few minutes clarifying one scenario, draw its flow, and explain one rejected alternative. Then introduce one of these changes:

| Change | Your answer should address |
|---|---|
| A token was copied from a log | Token use versus issuance, sender constraints, redaction, and outstanding-token invalidation |
| The issuer is unavailable for 20 minutes | Existing verification state versus new logins/renewal; bounded caches and high-risk operation policy |
| A valid user switches the tenant ID | Object and tenant authorization independent of successful authentication |
| The user's permissions change mid-session | Stale claims, session state, cache duration, and revocation expectations |
| A gateway is bypassed | Independent service enforcement or a deliberately enforced, authenticated gateway trust boundary |
| A signing key is compromised | Stop issuance, replace trust, reject affected credentials, recover consumers/caches, and investigate persistence |

A strong answer explains who can do what, under whose authority, for how long, and what evidence each recipient validates. Name the remaining risk and the operational cost; listing protocol acronyms alone is insufficient.
