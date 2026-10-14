# Tricky Authentication & Authorization Interview Questions

Fifteen practice questions test authentication and authorization reasoning using documented vulnerability patterns and design tradeoffs.

Answer aloud before expanding the guidance. State assumptions, identify the accepted credential or authorization decision, explain why the proposed protection is insufficient, and name a test that would distinguish a correct design. The [architecture comparison](comparison.md) and [design patterns](design-patterns.md) provide longer worked examples.

## 1. The JWT Signature Is Valid. Why Might the API Still Reject It?

**Scenario:** An API accepts tokens signed by a trusted provider. A caller presents a correctly signed OIDC ID token issued to the browser application. Its subject is a real user and it has not expired. Should the API accept it?

??? note "Answer guidance and common trap"
    **No, not merely on those facts.** A signature establishes integrity under a key; it does not establish that this credential is intended for this recipient or purpose. An ID token authenticates a user to its OIDC client. The API needs the credential type and audience defined by its own access-token contract.

    Validate the trusted issuer/key context, allowed algorithm, audience, time bounds, required claims, and token purpose before a separate authorization decision. A valid identity also needs permission for the requested resource. Use distinct validation policies for different token types rather than a generic “any signed JWT” middleware.

    **Trap:** “The issuer signed it, so its roles are safe to use everywhere.”

    **Test:** A signed token with the browser client as audience fails at the API even when a matching API access token succeeds.

**Follow-up:** What if two token types share the same signing key and some overlapping claims? Explain profile-specific acceptance rules; key separation can help but does not replace audience/purpose checks.

!!! info "External Resources"
    - [JWT Audience and Token Confusion Defenses](https://www.rfc-editor.org/info/rfc8725/) (IETF)
    - [OIDC ID Token Validation](https://openid.net/specs/openid-connect-core-1_0.html#IDTokenValidation) (OpenID Foundation)

## 2. Can You Guarantee Revocation Within 30 Seconds with 15-Minute JWTs?

**Scenario:** Access tokens last 15 minutes and APIs validate them locally. An employee loses an administrator role. Product requires that new administrator requests stop succeeding within 30 seconds. The team revokes the refresh token and clears the browser cookie. Is the requirement met?

??? note "Answer guidance and common trap"
    **Not by those actions alone.** The copied access token remains usable under the stated local validation policy until expiry. Browser logout does not remove another copy, and stopping renewal does not necessarily reject an existing token.

    Add a sufficiently fresh account/permission or revocation check, use introspection with a suitable cache bound, or change the lifetime/authorization design. Include replication and invalidation delays in the guarantee. A 60-second positive cache already conflicts with a 30-second bound unless another mechanism shortens acceptance.

    **Trap:** Equating “cannot get another token” with “cannot use this token.”

    **Test:** Copy a credential in a controlled test, revoke the role, and measure when every API region stops accepting new privileged requests. Separately decide what happens to operations already in progress.

**Follow-up:** Can you still call the system stateless? Explain which part verifies tokens locally and which part needs current authorization state; terminology cannot override the revocation requirement.

!!! info "External Resources"
    - [Token Introspection and Cache Freshness](https://www.rfc-editor.org/rfc/rfc7662.html) (IETF)
    - [OAuth Token Revocation](https://www.rfc-editor.org/rfc/rfc7009.html) (IETF)

## 3. Does PKCE Let You Remove State, Nonce, and Client Authentication?

**Scenario:** A developer adds PKCE to an OIDC login and deletes the library's transaction handling because “the verifier prevents all login attacks.” Which protections remain necessary?

??? note "Answer guidance and common trap"
    PKCE binds code redemption to the initiating transaction's verifier. It does not authenticate a confidential client or validate an ID token. Client authentication remains required by that client's profile, and issuer, redirect, and token validation remain necessary.

    State correlates the browser transaction; an OIDC nonce, when sent, must match the token. Under specified conditions, correctly implemented PKCE or nonce handling can provide CSRF protection. Therefore “no state always means exploitable” is also too broad. The safe implementation is the library's supported, transaction-bound defense, not deletion based on a slogan.

    **Trap:** Treating three differently scoped controls as interchangeable strings.

    **Test:** A callback from a different browser transaction, wrong issuer, or wrong nonce must not create a session; a stolen code without its verifier must not redeem successfully.

**Follow-up:** Would a client secret compiled into a distributed CLI make it confidential? No; evaluate who can recover or use that credential.

!!! info "External Resources"
    - [Code Injection and CSRF Defenses](https://www.rfc-editor.org/rfc/rfc9700.html#section-2.1.1) (IETF)
    - [PKCE Protocol](https://www.rfc-editor.org/rfc/rfc7636.html#section-4) (IETF)

## 4. Are SameSite and HttpOnly Enough to Protect a BFF?

**Scenario:** `app.example.com` uses an HttpOnly, SameSite cookie. Untrusted customer content runs at `uploads.example.com`, both over HTTPS. The team removes CSRF checks and says XSS cannot harm the session because JavaScript cannot read the cookie. What is wrong?

??? note "Answer guidance and common trap"
    Same-site is not same-origin: sibling HTTPS subdomains can be same-site. SameSite alone therefore does not establish that a request came from trusted application code. SOP/CORS may restrict response reading while some unwanted requests still reach the server.

    HttpOnly prevents direct cookie reads, but script executing in the application's origin can still issue authenticated requests. Keep appropriate CSRF validation, safe-method semantics, and explicit cookie scope; fix XSS and isolate untrusted content under a suitable separate site/origin architecture.

    **Trap:** “Cannot extract the credential” means “cannot use the user's authority.”

    **Test:** Requests from the untrusted sibling context cannot complete a state change without the required CSRF evidence. Also test injected same-origin script separately; CSRF tokens alone do not neutralize XSS.

**Follow-up:** Does changing Lax to Strict isolate an untrusted same-site subdomain? No; it does not change the site/origin distinction.

!!! info "External Resources"
    - [SameSite and CSRF Limitations](https://cheatsheetseries.owasp.org/cheatsheets/Cross-Site_Request_Forgery_Prevention_Cheat_Sheet.html) (OWASP)
    - [Browser Architecture Threats](https://www.rfc-editor.org/rfc/rfc10017.html#section-5) (IETF)

## 5. Can Two Accounts Be Linked Because Both Providers Verify the Same Email?

**Scenario:** A local privileged account and a new federated identity report the same email address. The new provider marks the email verified. Should the application automatically merge the accounts?

??? note "Answer guidance and common trap"
    **Not as a general rule.** Establishing identity in one provider's context is not automatically authority over an existing account in another context. Email can change or be reassigned, and provider/tenant trust policies differ.

    Map federated identities by trusted issuer and stable subject. For linking, require a deliberately authorized process, normally proof of control of the existing account and the new identity, recent authentication where appropriate, and transaction-bound CSRF protection. A controlled enterprise provisioning policy may define exceptions, but they must be explicit and scoped.

    **Trap:** “Verified” is treated as a universal account-ownership assertion.

    **Test:** An identity from an unapproved issuer or another tenant cannot acquire an existing account through an email collision or a forged linking callback.

**Follow-up:** Is signing in through the new identity enough to unlink the old one or replace recovery factors? Explain why changing account-control mechanisms requires separate authorization.

!!! info "External Resources"
    - [Authentication and Account Linking](https://cheatsheetseries.owasp.org/cheatsheets/Authentication_Cheat_Sheet.html) (OWASP)
    - [OAuth Account-Linking Vulnerability Patterns](https://portswigger.net/web-security/oauth) (PortSwigger)

## 6. A Python Client Calls Your API. Which OAuth Grant Should It Use?

**Scenario:** An interviewer asks for authentication for a Python client. You answer “client credentials.” What requirement have you not established?

??? note "Answer guidance and common trap"
    The programming language says nothing about whose authority the client exercises. Clarify whether it is a human-operated tool, a confidential unattended application, or a workload acting under a user's delegation.

    A user tool commonly uses authorization code + PKCE or an appropriate device flow. A confidential application's independent authority can use client credentials. A cloud/CI workload may use a provider-specific federation profile. A scheduled user-delegated integration may instead retain a renewable user grant under explicit consent and lifecycle rules.

    **Trap:** “Machine makes request” is confused with “no human authority is involved.”

    **Test:** Audit records and policy distinguish application-only access from actions on behalf of a user, including the correct tenant, scope, and actor.

**Follow-up:** A nightly business export must continue after the employee who configured it leaves. Decide whether the business should authorize a separately owned application identity instead of retaining that employee's account.

!!! info "External Resources"
    - [Client Credentials Grant](https://www.rfc-editor.org/rfc/rfc6749.html#section-4.4) (IETF)
    - [Native Application OAuth](https://www.rfc-editor.org/rfc/rfc8252.html) (IETF)

## 7. Does mTLS at the Token Endpoint Stop a Stolen Access Token?

**Scenario:** A partner authenticates to the token endpoint with mTLS. An attacker later steals only its access token. The API accepts ordinary bearer tokens. Can the attacker call the API?

??? note "Answer guidance and common trap"
    **Yes, under the stated assumptions.** Client authentication at issuance and sender-constraining token use are distinct mechanisms. The API's bearer contract does not require the client's private-key proof.

    Certificate-bound access tokens require the issuer to bind the credential and the resource server to validate the presenting certificate against that binding. DPoP is another supported proof-of-possession design. Both need correct end-to-end enforcement. Neither prevents misuse by an attacker who can use the matching private key or the authorized execution environment.

    **Trap:** An authenticated connection to one server is assumed to protect later use at another server.

    **Test:** Present the token without the bound proof or with a different certificate/key; a sender-constrained API rejects it.

**Follow-up:** TLS terminates at a load balancer. Explain how the verifier receives authenticated client-certificate evidence and why a caller-supplied certificate header is not proof.

!!! info "External Resources"
    - [mTLS Authentication versus Certificate-Bound Tokens](https://www.rfc-editor.org/rfc/rfc8705.html) (IETF)
    - [DPoP](https://www.rfc-editor.org/rfc/rfc9449.html) (IETF)

## 8. Can an Account with FIDO2 Still Be Taken Over?

**Scenario:** An application requires a phishing-resistant security key at login. Support can reset the factor after answering profile questions, and administrators keep long-lived sessions. Is takeover eliminated?

??? note "Answer guidance and common trap"
    **No.** The authenticator protects a particular ceremony and relying-party scope. It does not make recovery, enrollment, or subsequent sessions equally strong. A weak reset path or copied valid session can bypass repeating the original login.

    Protect recovery and credential changes with an appropriate verification/approval policy; manage session exposure, expiry, invalidation, and fresh authentication for sensitive actions. Distinguish user presence from required user verification, and device-bound credentials from synced credentials.

    **Trap:** A strong primary factor is assumed to strengthen every account-control path automatically.

    **Test:** Recovery cannot silently replace the factor with weaker evidence; sensitive operations require the configured server-validated authentication context.

**Follow-up:** Would requiring a fresh WebAuthn login prove approval of a particular payout destination? Not unless the design binds and verifies the intended operation; login proof alone does not sign arbitrary transaction parameters.

!!! info "External Resources"
    - [WebAuthn Security Considerations](https://www.w3.org/TR/webauthn-3/#sctn-security-considerations) (W3C)
    - [Transaction Authorization](https://cheatsheetseries.owasp.org/cheatsheets/Transaction_Authorization_Cheat_Sheet.html) (OWASP)

## 9. Does Replacing Sequential IDs with UUIDs Fix IDOR?

**Scenario:** A route verifies login and the `invoice.read` permission, then fetches an invoice using its UUID. The client also supplies a tenant ID. Why can this remain vulnerable?

??? note "Answer guidance and common trap"
    Unpredictability reduces enumeration, not unauthorized use of a discovered identifier. A permission to read invoices does not authorize every invoice. Check the current subject's relationship to the object and tenant, using trusted membership/policy rather than the request's tenant assertion.

    Constrain the lookup to authorized data where feasible, and apply consistent policy to lists, nested objects, exports, batch operations, and alternate interfaces. Do not rely on hiding IDs in the UI.

    **Trap:** Knowledge of an identifier is treated as evidence of entitlement.

    **Test:** An authorized user with a known UUID from another tenant is denied; their own permitted invoice succeeds. Include list/export endpoints, not just the detail route.

**Follow-up:** Should the denial be 403 or 404? Choose a consistent disclosure policy; status choice cannot replace authorization, and timing or other endpoints may still reveal existence.

!!! info "External Resources"
    - [IDOR Prevention and Multi-Identity Testing](https://cheatsheetseries.owasp.org/cheatsheets/Insecure_Direct_Object_Reference_Prevention_Cheat_Sheet.html) (OWASP)

## 10. Is a Tenant Administrator Allowed to Edit Every Field?

**Scenario:** A user is an administrator in tenant A. A profile endpoint binds the request JSON to a model. The request includes `tenantId`, `isGlobalAdmin`, and `ownerId`. Is checking the administrator role enough?

??? note "Answer guidance and common trap"
    A role is scoped authority, not a universal right to mutate every property. Define the permitted action, tenant, object, and fields. Derive privileged ownership/tenant values server-side and allow-list editable fields. Enforce business rules for transfers or privilege grants separately.

    RBAC can supply coarse role permissions; resource attributes and relationships may be needed for ownership and cross-tenant collaboration. Choose a policy model that represents the actual rules rather than creating global administrator shortcuts.

    **Trap:** Route-level permission is treated as object- and property-level permission.

    **Test:** An otherwise permitted update cannot change immutable tenant/ownership fields or elevate global privileges through additional JSON properties.

**Follow-up:** Customer support needs to assist tenant A. Preserve the support actor, define its permitted operations, and avoid unrestricted impersonation as the default solution.

!!! info "External Resources"
    - [Mass Assignment Prevention](https://cheatsheetseries.owasp.org/cheatsheets/Mass_Assignment_Cheat_Sheet.html) (OWASP)
    - [Authorization Policy Models](https://cheatsheetseries.owasp.org/cheatsheets/Authorization_Cheat_Sheet.html) (OWASP)

## 11. Can Internal Services Trust an X-User Header from the Gateway?

**Scenario:** The gateway validates login and forwards `X-User: user-42`. Internal services accept the header because they are on a private network. What must be true for this design to be safe?

??? note "Answer guidance and common trap"
    The system needs an enforced and authenticated trust boundary, not merely a header naming a user. Strip caller-supplied identity headers at ingress, authenticate the gateway/service channel, restrict direct access, and define who can assert which identities. A signed internal assertion can help when validated for issuer, recipient, expiry, and purpose.

    Services must still enforce resource/action policy at the correct layer. A deliberate centralized enforcement design is possible, but bypass paths, alternate callers, and privileged internal workloads must be included in its threat model.

    **Trap:** Network location is treated as proof of identity and authorization.

    **Test:** A direct service request and a request with a forged ingress header fail, while an authenticated, policy-approved gateway call succeeds.

**Follow-up:** If another workload has a certificate from the same CA, can it assert arbitrary users? No; authenticate the expected asserting identity and its permitted authority, not just CA membership.

!!! info "External Resources"
    - [Microservices Authentication and Identity Propagation](https://cheatsheetseries.owasp.org/cheatsheets/Microservices_Security_Cheat_Sheet.html) (OWASP)
    - [Identity Propagation Trust Patterns](https://cheatsheetseries.owasp.org/cheatsheets/Identity_Propagation_Patterns_Cheat_Sheet.html) (OWASP)

## 12. Can a Service Use Its Own Privileges When the User Is Denied?

**Scenario:** A user asks orders to retrieve a billing record. The user's downstream token is rejected. Orders retries using its powerful billing service credential, which succeeds. Is this a resilience feature?

??? note "Answer guidance and common trap"
    It changes the authority of the operation and can make orders a confused deputy: the caller induces a more privileged service to perform an action the caller was not allowed to request. Distinguish failed authentication from insufficient authorization; neither justifies silently escalating to application authority.

    For delegated actions, preserve the user subject and authenticated service actor under an explicit downstream policy. A supported token exchange can issue a billing-specific credential, but the STS must enforce allowed audience and permissions; exchange is not automatically safe downscoping. Genuine application-only tasks need a separate authorized contract.

    **Trap:** A successful technical retry is assumed to preserve security semantics.

    **Test:** A user denied access to a billing record stays denied through the orders API, even though orders has permissions for other legitimate system tasks.

**Follow-up:** Which identity belongs in the audit log? Preserve both the affected subject and acting service under the profile, plus the decision and resource; do not overwrite the actor with the user.

!!! info "External Resources"
    - [Delegation, Impersonation, and Actor Claims](https://www.rfc-editor.org/rfc/rfc8693.html#section-1.1) (IETF)

## 13. A SAML Signature Validates. Can You Use Any Assertion in the Response?

**Scenario:** A SAML response contains multiple XML elements. The signature library reports success, but application code separately searches for the first assertion and reads its subject. What could go wrong?

??? note "Answer guidance and common trap"
    The application may consume a different assertion from the content actually protected by the signature. This mismatch is central to signature-wrapping attacks. Use the library's validated result under the chosen SAML profile; reject duplicate IDs and ambiguous structure rather than performing an unrelated XML search.

    Also validate trusted issuer/key, intended audience, recipient/destination as required, time conditions, SP-initiated request correlation, status, and replay policy. A signed assertion for another SP is not authority to establish this application's session.

    **Trap:** “Some signature is valid” is treated as “all parsed identity data is trusted.”

    **Test:** A trusted signed assertion for another audience, or an unsigned injected assertion beside valid signed content, cannot change the authenticated account.

**Follow-up:** Does POST binding fix this? No; transport binding alone does not ensure the application consumes the correct authenticated content.

!!! info "External Resources"
    - [SAML Validation and Signature-Wrapping Defenses](https://cheatsheetseries.owasp.org/cheatsheets/SAML_Security_Cheat_Sheet.html) (OWASP)

## 14. If a Job Was Authorized When Queued, Can It Run Tomorrow?

**Scenario:** A user schedules a sensitive export. Before execution, the user is removed from the tenant and the requested resource changes ownership. The job contains yesterday's access token and an `authorized=true` field. Should it proceed?

??? note "Answer guidance and common trap"
    Define the business authorization contract first. A job requiring current entitlement should recheck the user/resource/tenant policy at execution. A deliberately durable approval may survive some later changes, but needs server-controlled scope, integrity, expiry/cancellation, and parameter binding. A caller-controlled boolean and an expired bearer token establish neither model.

    Authenticate the worker separately. Bind approved parameters to the actual operation and coordinate final authorization with execution to address races. Use idempotency and replay controls where the operation requires them; they do not replace authorization.

    **Trap:** An old check is assumed to authorize any later operation with similar parameters.

    **Test:** Change membership, ownership, and export parameters between approval and execution and verify the explicitly chosen policy in each case.

**Follow-up:** Must a permission removal cancel a transaction already committed? Explain the system's execution boundary and business policy instead of promising retroactive revocation.

!!! info "External Resources"
    - [Transaction Data Binding and Final Authorization Checks](https://cheatsheetseries.owasp.org/cheatsheets/Transaction_Authorization_Cheat_Sheet.html) (OWASP)

## 15. Can You Cache an Allow Decision by User ID and Keep Serving During an Outage?

**Scenario:** A shared authorization cache uses only `userId` as its key. During a policy-service outage, the application reuses any cached allow decision. One user belongs to multiple tenants with different permissions. What is the failure mode?

??? note "Answer guidance and common trap"
    An allow decision is about a particular subject, action, resource, tenant, and relevant context. Omitting policy inputs can reuse permission for a different operation. Define a cache key covering the decision's actual inputs, a validity bound, and invalidation/version behavior. Permission changes and data changes can invalidate the result even if the identity is unchanged.

    Separate authentication verification from current authorization. A known unexpired credential may remain verifiable with trusted cached keys; that does not make an unrelated cached allow valid. Define which operations can use bounded cached policy and which must stop when fresh evidence is unavailable. Unknown keys and expired credentials are not resolved by a fail-open rule.

    **Trap:** Preserving availability is used to silently broaden the scope or lifetime of an earlier permission.

    **Test:** The same subject's allow for tenant A/read cannot authorize tenant B/write; expired policy evidence is handled according to the documented operation-specific outage policy.

**Follow-up:** What would you measure? Cache age, invalidation delay, authorization failures, and outage behavior, without logging raw credentials or sensitive resource contents.

!!! info "External Resources"
    - [Authorization Checks and Safe Failure](https://cheatsheetseries.owasp.org/cheatsheets/Authorization_Cheat_Sheet.html) (OWASP)
    - [Token Introspection Cache Considerations](https://www.rfc-editor.org/rfc/rfc7662.html#section-4) (IETF)

## Self-Assessment

Use this rubric to assess your answers:

| Score | Evidence in your answer |
|---|---|
| 0 | Repeats the incorrect assumption or lists acronyms without identifying the decision |
| 1 | Spots the issue but cannot explain why the control is insufficient |
| 2 | Explains the boundary and a suitable correction under stated assumptions |
| 3 | Also compares a realistic alternative, identifies residual risk, and gives a rejection/failure test |

Repeat questions after changing one assumption: add another tenant, remove the browser, steal only the token, compromise the execution environment, or make the issuer unavailable. Rehearse the reasoning rather than memorizing the first sentence of an answer.
