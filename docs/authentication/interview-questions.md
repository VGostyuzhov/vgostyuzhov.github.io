# Tricky Authentication & Authorization Interview Questions

Nineteen practice questions test authentication and authorization reasoning using documented vulnerability patterns and design tradeoffs.

Answer aloud before expanding the guidance. Each question has separate expandable answer guidance and a worked example with concrete inputs, commented code or protocol messages, and expected decisions. Code marked as pseudocode illustrates application logic; placeholders and demo keys are not production credentials. State assumptions, identify the accepted credential or authorization decision, explain why the proposed protection is insufficient, and name a test that would distinguish a correct design. The [architecture comparison](comparison.md) and [design patterns](design-patterns.md) provide longer worked examples.

## 1. The JWT Signature Is Valid. Why Might the API Still Reject It?

**Study first:** [JWT acceptance](jwt.md#acceptance-and-authorization) and [token roles](oauth2-oidc.md#roles-and-token-purposes).

**Scenario:** An API accepts tokens signed by a trusted provider. A caller presents a correctly signed OIDC ID token issued to the browser application. Its subject is a real user and it has not expired. Should the API accept it?

??? note "Answer guidance and common trap"
    **No, not merely on those facts.** A signature establishes integrity under a key; it does not establish that this credential is intended for this recipient or purpose. An ID token authenticates a user to its OIDC client. The API needs the credential type and audience defined by its own access-token contract.

    Validate the trusted issuer/key context, allowed algorithm, audience, time bounds, required claims, and token purpose before a separate authorization decision. A valid identity also needs permission for the requested resource. Use distinct validation policies for different token types rather than a generic “any signed JWT” middleware.

    **Trap:** “The issuer signed it, so its roles are safe to use everywhere.”

    **Test:** A signed token with the browser client as audience fails at the API even when a matching API access token succeeds.

??? example "Worked example"
    This runnable PyJWT example creates real signed tokens with a public demo key. Use it only as a local fixture: anyone who knows this key can mint tokens. Production follows the actual issuer's key distribution and token profile.

    ```python
    import time
    import jwt  # PyJWT; example dependency, not Python's standard library.

    DEMO_KEY = "documentation-only-secret-not-for-production"
    now = int(time.time())
    common = {"iss": "https://id.example", "sub": "user-42",
              "iat": now, "exp": now + 600}
    id_token = jwt.encode({**common, "aud": "browser-client"},
                          DEMO_KEY, algorithm="HS256")
    access_token = jwt.encode({**common, "aud": "orders-api"},
                              DEMO_KEY, algorithm="HS256")
    print(id_token)      # A complete header.payload.signature compact JWT.
    print(access_token)  # Separately signed; not an edited ID-token payload.

    for label, raw in (("ID token", id_token), ("access token", access_token)):
        try:
            claims = jwt.decode(
                raw, DEMO_KEY, algorithms=["HS256"],
                issuer="https://id.example", audience="orders-api",
                options={"require": ["iss", "sub", "aud", "iat", "exp"]},
            )
            print(label, "passes these token checks")
        except jwt.InvalidAudienceError:
            print(label, "rejected: recipient is not orders-api")
    # Expected: ID token rejected; access token passes these checks.
    # A real API also enforces its token-purpose profile and resource policy.
    ```

    Decode either payload for inspection: the audience differs despite the same issuer and subject. Editing `aud` inside the first token without re-signing it must fail signature verification. Neither decoding nor passing these illustrative checks replaces the API's full access-token acceptance contract.

**Follow-up:** What if two token types share the same signing key and some overlapping claims? Explain profile-specific acceptance rules; key separation can help but does not replace audience/purpose checks.

!!! info "External Resources"
    - [JWT Audience and Token Confusion Defenses](https://www.rfc-editor.org/info/rfc8725/) (IETF)
    - [OIDC ID Token Validation](https://openid.net/specs/openid-connect-core-1_0.html#IDTokenValidation) (OpenID Foundation)

## 2. Can You Guarantee Revocation Within 30 Seconds with 15-Minute JWTs?

**Study first:** [Revocation and current authorization](oauth2-oidc.md#revocation-and-current-authorization).

**Scenario:** Access tokens last 15 minutes and APIs validate them locally. An employee loses an administrator role. Product requires that new administrator requests stop succeeding within 30 seconds. The team revokes the refresh token and clears the browser cookie. Is the requirement met?

??? note "Answer guidance and common trap"
    **Not by those actions alone.** The copied access token remains usable under the stated local validation policy until expiry. Browser logout does not remove another copy, and stopping renewal does not necessarily reject an existing token.

    The API needs a way to learn that authority changed. A **current permission lookup** asks the application's policy store whether this user is still an administrator. A **revocation lookup** asks whether this token/session has been invalidated. **Introspection** asks the authorization server whether the token is active; verify which account changes actually make it inactive. These checks answer different questions, and the API must still authorize the requested operation.

    A successful result can be **cached** to avoid a lookup on every request. Suppose the API learns “administrator allowed” at 12:00:00 and keeps that result for 60 seconds. The role is removed at 12:00:01, but the API can continue using the cached allow until 12:01:00. That already exceeds the 30-second removal requirement.

    Updates can also take time to reach another region's database copy (**replication**) or to cause an API to discard its saved result (**cache invalidation**). Bound these delays on every request path. For example, a maximum 5-second state propagation delay plus 20-second cache reuse and 2-second processing allowance gives a 27-second bound under those stated assumptions. If fresh evidence cannot be obtained, privileged requests must follow an explicit outage policy; continuing indefinitely would break the guarantee. See the [full mechanism and timing explanation](oauth2-oidc.md#revocation-and-current-authorization).

    **Trap:** Equating “cannot get another token” with “cannot use this token.”

    **Test:** Copy a credential in a controlled test, revoke the role, and measure when every API region stops accepting new privileged requests. Separately decide what happens to operations already in progress.

??? example "Worked example"
    This timeline distinguishes the signed credential's expiry from a cached permission decision. Times and bounds are illustrative.

    ```text
    12:00:00  JWT issued: sub=admin-7, role=admin, exp=12:15:00
    12:00:59  API caches current_permission(admin-7, admin)=allow for 60s
    12:01:00  Administrator role removed; refresh grant revoked
    12:01:31  Copied JWT is still unexpired; cached allow is still usable
    12:01:59  Old cached allow finally expires
    # A 30-second removal guarantee failed even with an immediate central update.
    ```

    ```python
    # Application pseudocode: this check runs on every privileged operation.
    claims = validate_access_token(request.token)  # Signature + profile + times.
    permission = permissions.lookup(claims["sub"], request.tenant, "admin")
    if permission.unavailable or permission.too_old:
        deny()  # This privileged operation requires bounded fresh evidence.
    if not permission.allowed:
        deny()
    # Example budget: propagation <= 5s, cache reuse <= 20s, overhead <= 2s.
    # Enforced worst-case total: 27s, including a lookup just before propagation.
    ```

    Cache deadlines start at the authority evaluation and must not be reset by cache hits. A permission lookup must actually reflect role removal. Merely introspecting a token that the issuer still considers active does not supply that guarantee.

**Follow-up:** Can you still call the system stateless? Explain which part verifies tokens locally and which part needs current authorization state; terminology cannot override the revocation requirement.

!!! info "External Resources"
    - [Token Introspection and Cache Freshness](https://www.rfc-editor.org/rfc/rfc7662.html) (IETF)
    - [OAuth Token Revocation](https://www.rfc-editor.org/rfc/rfc7009.html) (IETF)

## 3. Does PKCE Let You Remove State, Nonce, and Client Authentication?

**Study first:** [Authorization code + PKCE](oauth2-oidc.md#authorization-code-flow-with-pkce).

**Scenario:** A developer adds PKCE to an OIDC login and deletes the library's transaction handling because “the verifier prevents all login attacks.” Which protections remain necessary?

??? note "Answer guidance and common trap"
    PKCE binds code redemption to the initiating transaction's verifier. It does not authenticate a confidential client or validate an ID token. Client authentication remains required by that client's profile, and issuer, redirect, and token validation remain necessary.

    State correlates the browser transaction; an OIDC nonce, when sent, must match the token. Under specified conditions, correctly implemented PKCE or nonce handling can provide CSRF protection. Therefore “no state always means exploitable” is also too broad. The safe implementation is the library's supported, transaction-bound defense, not deletion based on a slogan.

    **Trap:** Treating three differently scoped controls as interchangeable strings.

    **Test:** A callback from a different browser transaction, wrong issuer, or wrong nonce must not create a session; a stolen code without its verifier must not redeem successfully.

??? example "Worked example"
    A code-flow request carries separate values for separate checks. The verifier/challenge pair below is the published RFC 7636 S256 test vector, for testing only. Production generates a new unpredictable verifier for every transaction.

    ```http
    GET /authorize?response_type=code&client_id=web-client&redirect_uri=https%3A%2F%2Fapp.example%2Fcallback&scope=openid&state=S-A&nonce=N-A&code_challenge=E9Melhoa2OwvFrEMTJguCHaoeK1t8URWbuGJSstw-cM&code_challenge_method=S256 HTTP/1.1
    Host: id.example
    ```

    ```python
    # Application pseudocode; a maintained OIDC library performs these bindings.
    # Stored when this browser started login; never put the verifier in the URL.
    # Test-only verifier: dBjftJeZ4CVP-mB92K27uhbUJU1p1r_wW1gFWFOEjXk
    tx = transaction_store.for_browser(request.session)
    require(request.query["state"] == tx.state)       # Callback correlation.
    tokens = oidc.redeem_code(
        code=request.query["code"], verifier=tx.verifier,
        client_auth=registered_client_auth,            # Confidential client proof.
    )
    id_claims = oidc.validate_id_token(tokens.id_token) # Issuer, aud, signature, times.
    require(id_claims["nonce"] == tx.nonce)            # N-B must not satisfy N-A.
    consume(tx)                                      # Transaction cannot be reused.
    # Wrong verifier: token endpoint rejects redemption.
    # Wrong nonce: client rejects login even if token signature is valid.
    ```

    This illustrates a state-and-nonce-based implementation. Another conforming library may use a different supported transaction/CSRF defense; PKCE alone is not permission to delete its validation.

**Follow-up:** Would a client secret compiled into a distributed CLI make it confidential? No; evaluate who can recover or use that credential.

!!! info "External Resources"
    - [Code Injection and CSRF Defenses](https://www.rfc-editor.org/rfc/rfc9700.html#section-2.1.1) (IETF)
    - [PKCE Protocol](https://www.rfc-editor.org/rfc/rfc7636.html#section-4) (IETF)

## 4. Are SameSite and HttpOnly Enough to Protect a BFF?

**Study first:** [Cookie scope and CSRF](sessions.md#cookie-scope-and-csrf) and [the BFF pattern](design-patterns.md#2-protect-a-browser-application-with-a-backend-for-frontend).

**Scenario:** `app.example.com` uses an HttpOnly, SameSite cookie. Untrusted customer content runs at `uploads.example.com`, both over HTTPS. The team removes CSRF checks and says XSS cannot harm the session because JavaScript cannot read the cookie. What is wrong?

??? note "Answer guidance and common trap"
    Same-site is not same-origin: sibling HTTPS subdomains can be same-site. SameSite alone therefore does not establish that a request came from trusted application code. SOP/CORS may restrict response reading while some unwanted requests still reach the server.

    HttpOnly prevents direct cookie reads, but script executing in the application's origin can still issue authenticated requests. Keep appropriate CSRF validation, safe-method semantics, and explicit cookie scope; fix XSS and isolate untrusted content under a suitable separate site/origin architecture.

    **Trap:** “Cannot extract the credential” means “cannot use the user's authority.”

    **Test:** Requests from the untrusted sibling context cannot complete a state change without the required CSRF evidence. Also test injected same-origin script separately; CSRF tokens alone do not neutralize XSS.

??? example "Worked example"
    A sibling page can send a simple form request without reading the application's cookie. This is a local test fixture, not a claim that CORS permits reading the response.

    ```http
    Set-Cookie: __Host-session=<opaque-id>; Secure; HttpOnly; SameSite=Lax; Path=/
    ```

    ```html
    <!-- Hosted on https://uploads.example.com, a different origin but same site. -->
    <form action="https://app.example.com/account/email" method="POST">
      <input name="email" value="changed@example.test">
      <!-- No valid application CSRF token is available to this page. -->
      <button>Submit test request</button>
    </form>
    ```

    ```python
    # Application pseudocode for the receiving endpoint.
    session = require_valid_session(request.cookies["__Host-session"])
    csrf.verify(request, session)  # Reject missing/session-mismatched CSRF evidence.
    authorize(session.user, "change-email", session.account)
    # HttpOnly blocks document.cookie reads; it does not stop automatic cookie sends.
    # Same-origin XSS can often read CSRF evidence and invoke this endpoint itself.
    ```

    Keep state changes off GET routes and apply the framework's full CSRF policy; a SameSite cookie is an additional control.

**Follow-up:** Does changing Lax to Strict isolate an untrusted same-site subdomain? No; it does not change the site/origin distinction.

!!! info "External Resources"
    - [SameSite and CSRF Limitations](https://cheatsheetseries.owasp.org/cheatsheets/Cross-Site_Request_Forgery_Prevention_Cheat_Sheet.html) (OWASP)
    - [Browser Architecture Threats](https://www.rfc-editor.org/rfc/rfc10017.html#section-5) (IETF)

## 5. Can Two Accounts Be Linked Because Both Providers Verify the Same Email?

**Study first:** [Account linking](design-patterns.md#account-linking-and-unlinking).

**Scenario:** A local privileged account and a new federated identity report the same email address. The new provider marks the email verified. Should the application automatically merge the accounts?

??? note "Answer guidance and common trap"
    **Not as a general rule.** Establishing identity in one provider's context is not automatically authority over an existing account in another context. Email can change or be reassigned, and provider/tenant trust policies differ.

    Map federated identities by trusted issuer and stable subject. For linking, require a deliberately authorized process, normally proof of control of the existing account and the new identity, recent authentication where appropriate, and transaction-bound CSRF protection. A controlled enterprise provisioning policy may define exceptions, but they must be explicit and scoped.

    **Trap:** “Verified” is treated as a universal account-ownership assertion.

    **Test:** An identity from an unapproved issuer or another tenant cannot acquire an existing account through an email collision or a forged linking callback.

??? example "Worked example"
    The same email appears in two different provider identities. These are decoded claims for illustration, not complete signed ID tokens.

    ```json
    {
      "existing_identity": {
        "iss": "https://issuer-a.example", "sub": "17", "email": "admin@example.com"
      },
      "new_identity": {
        "iss": "https://issuer-b.example", "sub": "93",
        "email": "admin@example.com", "email_verified": true
      }
    }
    ```

    ```python
    # Application pseudocode: issuer/subject is the identity key, not email.
    new_identity = (validated_claims["iss"], validated_claims["sub"])
    # UNSAFE: accounts.by_email(validated_claims["email"]).link(new_identity)
    existing = require_existing_account_session()
    require_recent_authentication(existing)
    link_tx = require_bound_link_transaction(existing, callback)
    require(link_tx.new_identity == new_identity)
    authorize(existing.user, "link-identity", existing.account)
    identities.link_uniquely(existing.account, new_identity)
    # Reject a callback for another browser/account; never silently merge by email.
    ```

    The callback also needs normal provider and transaction validation. A separately approved enterprise linking policy must define its narrower authority explicitly.

**Follow-up:** Is signing in through the new identity enough to unlink the old one or replace recovery factors? Explain why changing account-control mechanisms requires separate authorization.

!!! info "External Resources"
    - [Authentication and Account Linking](https://cheatsheetseries.owasp.org/cheatsheets/Authentication_Cheat_Sheet.html) (OWASP)
    - [OAuth Account-Linking Vulnerability Patterns](https://portswigger.net/web-security/oauth) (PortSwigger)

## 6. A Python Client Calls Your API. Which OAuth Grant Should It Use?

**Study first:** [Human CLI versus daemon](design-patterns.md#4-authenticate-a-human-cli-separately-from-a-daemon).

**Scenario:** An interviewer asks for authentication for a Python client. You answer “client credentials.” What requirement have you not established?

??? note "Answer guidance and common trap"
    The programming language says nothing about whose authority the client exercises. Clarify whether it is a human-operated tool, a confidential unattended application, or a workload acting under a user's delegation.

    A user tool commonly uses authorization code + PKCE or an appropriate device flow. A confidential application's independent authority can use client credentials. A cloud/CI workload may use a provider-specific federation profile. A scheduled user-delegated integration may instead retain a renewable user grant under explicit consent and lifecycle rules.

    **Trap:** “Machine makes request” is confused with “no human authority is involved.”

    **Test:** Audit records and policy distinguish application-only access from actions on behalf of a user, including the correct tenant, scope, and actor.

??? example "Worked example"
    The request language does not identify the principal. These are token-endpoint request bodies after the appropriate setup.

    ```text
    # Alice's interactive CLI: redeem a browser-issued code with its PKCE verifier.
    # Public client: no supposedly secret credential embedded in the distributed CLI.
    grant_type=authorization_code&client_id=cli&code=<one-time-code>&code_verifier=<verifier>&redirect_uri=<registered-uri>

    # Organization-owned daemon: authenticate the confidential application itself.
    # Client authentication is supplied separately by its registered method.
    grant_type=client_credentials&scope=reports.read
    ```

    ```python
    # Application audit pseudocode: retain whose authority is being exercised.
    audit(principal="alice", mode="user-delegated", action="report.read")
    audit(principal="nightly-export", mode="application", action="report.export")
    # Offboarding Alice must not require retaining her login for the nightly job.
    ```

    A supported workload-federation exchange can replace a stored daemon credential and may use a different grant/profile.

**Follow-up:** A nightly business export must continue after the employee who configured it leaves. Decide whether the business should authorize a separately owned application identity instead of retaining that employee's account.

!!! info "External Resources"
    - [Client Credentials Grant](https://www.rfc-editor.org/rfc/rfc6749.html#section-4.4) (IETF)
    - [Native Application OAuth](https://www.rfc-editor.org/rfc/rfc8252.html) (IETF)

## 7. Does mTLS at the Token Endpoint Stop a Stolen Access Token?

**Study first:** [Sender-constrained token use](mtls.md#oauth-client-authentication-and-token-binding).

**Scenario:** A partner authenticates to the token endpoint with mTLS. An attacker later steals only its access token. The API accepts ordinary bearer tokens. Can the attacker call the API?

??? note "Answer guidance and common trap"
    **Yes, under the stated assumptions.** Client authentication at issuance and sender-constraining token use are distinct mechanisms. The API's bearer contract does not require the client's private-key proof.

    Certificate-bound access tokens require the issuer to bind the credential and the resource server to validate the presenting certificate against that binding. DPoP is another supported proof-of-possession design. Both need correct end-to-end enforcement. Neither prevents misuse by an attacker who can use the matching private key or the authorized execution environment.

    **Trap:** An authenticated connection to one server is assumed to protect later use at another server.

    **Test:** Present the token without the bound proof or with a different certificate/key; a sender-constrained API rejects it.

??? example "Worked example"
    The first request uses a bearer token. In the second case, the API contract explicitly requires a certificate-bound credential.

    ```bash
    # A copied bearer token can be presented without the issuance client certificate.
    curl https://api.example/reports -H "Authorization: Bearer $ACCESS_TOKEN"

    # For a certificate-bound API, the client also proves possession of its key.
    curl https://api.example/reports \
      --cert partner-cert.pem --key partner-key.pem \
      -H "Authorization: Bearer $BOUND_ACCESS_TOKEN"
    # Copying the token without the bound private-key capability must not suffice.
    ```

    ```json
    {
      "cnf": {"x5t#S256": "<base64url-SHA256-of-bound-certificate-DER>"}
    }
    ```

    The JSON shows the certificate-binding claim used by the profile, not a complete token. The API validates the token and compares the authenticated client certificate with that binding, obtaining the certificate from validated TLS state or explicitly trusted proxy evidence, never an untrusted caller header. A different valid certificate must fail the binding check; trusting its CA alone is insufficient.

**Follow-up:** TLS terminates at a load balancer. Explain how the verifier receives authenticated client-certificate evidence and why a caller-supplied certificate header is not proof.

!!! info "External Resources"
    - [mTLS Authentication versus Certificate-Bound Tokens](https://www.rfc-editor.org/rfc/rfc8705.html) (IETF)
    - [DPoP](https://www.rfc-editor.org/rfc/rfc9449.html) (IETF)

## 8. Can an Account with FIDO2 Still Be Taken Over?

**Study first:** [WebAuthn threats and recovery](webauthn.md#threats-recovery-and-policy) and [sensitive actions](design-patterns.md#7-require-stronger-authentication-for-sensitive-actions).

**Scenario:** An application requires a phishing-resistant security key at login. Support can reset the factor after answering profile questions, and administrators keep long-lived sessions. Is takeover eliminated?

??? note "Answer guidance and common trap"
    **No.** The authenticator protects a particular ceremony and relying-party scope. It does not make recovery, enrollment, or subsequent sessions equally strong. A weak reset path or copied valid session can bypass repeating the original login.

    Protect recovery and credential changes with an appropriate verification/approval policy; manage session exposure, expiry, invalidation, and fresh authentication for sensitive actions. Distinguish user presence from required user verification, and device-bound credentials from synced credentials.

    **Trap:** A strong primary factor is assumed to strengthen every account-control path automatically.

    **Test:** Recovery cannot silently replace the factor with weaker evidence; sensitive operations require the configured server-validated authentication context.

??? example "Worked example"
    A weak recovery path can replace a strong login credential without defeating its cryptography.

    ```text
    # Unsafe recovery sequence:
    POST /support/reset-factor  -> profile answers accepted
    POST /webauthn/register     -> attacker's public key enrolled for Alice
    POST /login                -> attacker now uses the enrolled credential
    # Alice's original security key was never cloned or phished.
    ```

    ```python
    # Application pseudocode for a sensitive payout operation.
    session = require_valid_session(request.cookie)
    authorize(session.user, "payout.change", account)
    require_recent_verified_authentication(session, max_age_seconds=300)
    approval = require_server_owned_approval(session.user, request.approval_id)
    require(approval.action == "payout.change" and approval.account == account.id)
    require(not approval.expired and not approval.cancelled)
    require(approval.destination == request.destination)
    consume_and_execute(approval)  # Coordinate consumption and operation execution.
    # A new login alone does not approve a different destination supplied afterward.
    ```

    Recovery, credential enrollment, and session invalidation each need their own controls. The 300-second recency value is illustrative.

**Follow-up:** Would requiring a fresh WebAuthn login prove approval of a particular payout destination? Not unless the design binds and verifies the intended operation; login proof alone does not sign arbitrary transaction parameters.

!!! info "External Resources"
    - [WebAuthn Security Considerations](https://www.w3.org/TR/webauthn-3/#sctn-security-considerations) (W3C)
    - [Transaction Authorization](https://cheatsheetseries.owasp.org/cheatsheets/Transaction_Authorization_Cheat_Sheet.html) (OWASP)

## 9. Does Replacing Sequential IDs with UUIDs Fix IDOR?

**Study first:** [Object and property authorization](../web-security/vulnerabilities.md#common-web-vulnerabilities) and [authorization models](comparison.md#authorization-decisions-and-their-caches).

**Scenario:** A route verifies login and the `invoice.read` permission, then fetches an invoice using its UUID. The client also supplies a tenant ID. Why can this remain vulnerable?

??? note "Answer guidance and common trap"
    Unpredictability reduces enumeration, not unauthorized use of a discovered identifier. A permission to read invoices does not authorize every invoice. Check the current subject's relationship to the object and tenant, using trusted membership/policy rather than the request's tenant assertion.

    Constrain the lookup to authorized data where feasible, and apply consistent policy to lists, nested objects, exports, batch operations, and alternate interfaces. Do not rely on hiding IDs in the UI.

    **Trap:** Knowledge of an identifier is treated as evidence of entitlement.

    **Test:** An authorized user with a known UUID from another tenant is denied; their own permitted invoice succeeds. Include list/export endpoints, not just the detail route.

??? example "Worked example"
    A scoped query prevents knowledge of another invoice UUID from becoming authority to read it.

    ```python
    # Application pseudocode; placeholders are bound parameters, not string SQL.
    tenant = require_authorized_tenant(current_user, request.tenant_id)
    invoice = db.query_one(
        "SELECT * FROM invoices WHERE id = :id AND tenant_id = :tenant",
        {"id": request.invoice_id, "tenant": tenant.id},
    )
    if invoice is None:
        deny_not_found()  # Disclosure policy; authorization still needs enforcement.
    authorize(current_user, "invoice.read", invoice)  # Ownership/relationship policy.
    # UUID knowledge and a broad invoice.read scope cannot bypass these restrictions.
    ```

    ```text
    Alice / tenant-A / own invoice UUID          -> allowed if policy permits
    Alice / tenant-B / known foreign invoice UUID -> denied
    # Run the same cross-tenant test against list, batch, and export routes.
    ```

**Follow-up:** Should the denial be 403 or 404? Choose a consistent disclosure policy; status choice cannot replace authorization, and timing or other endpoints may still reveal existence.

!!! info "External Resources"
    - [IDOR Prevention and Multi-Identity Testing](https://cheatsheetseries.owasp.org/cheatsheets/Insecure_Direct_Object_Reference_Prevention_Cheat_Sheet.html) (OWASP)

## 10. Is a Tenant Administrator Allowed to Edit Every Field?

**Study first:** [Authorization decisions](comparison.md#authorization-decisions-and-their-caches) and [mass assignment](../web-security/vulnerabilities.md#common-web-vulnerabilities).

**Scenario:** A user is an administrator in tenant A. A profile endpoint binds the request JSON to a model. The request includes `tenantId`, `isGlobalAdmin`, and `ownerId`. Is checking the administrator role enough?

??? note "Answer guidance and common trap"
    A role is scoped authority, not a universal right to mutate every property. Define the permitted action, tenant, object, and fields. Derive privileged ownership/tenant values server-side and allow-list editable fields. Enforce business rules for transfers or privilege grants separately.

    RBAC can supply coarse role permissions; resource attributes and relationships may be needed for ownership and cross-tenant collaboration. Choose a policy model that represents the actual rules rather than creating global administrator shortcuts.

    **Trap:** Route-level permission is treated as object- and property-level permission.

    **Test:** An otherwise permitted update cannot change immutable tenant/ownership fields or elevate global privileges through additional JSON properties.

??? example "Worked example"
    An otherwise legitimate request includes fields the caller must not control.

    ```json
    {
      "displayName": "Alice",
      "tenantId": "tenant-B",
      "isGlobalAdmin": true,
      "ownerId": "another-user"
    }
    ```

    ```python
    # Application pseudocode: do not deserialize the whole request into the DB model.
    editable = {"displayName"}
    require(set(request.json) <= editable)  # Example contract: reject unknown fields.
    profile = load_profile_in_authorized_tenant(current_user, request.profile_id)
    authorize(current_user, "profile.edit", profile)
    profile.display_name = validate_display_name(request.json["displayName"])
    save(profile)
    # Tenant/owner/global-role changes require separately authorized operations.
    ```

    A tenant administrator's role does not make privileged model properties editable.

**Follow-up:** Customer support needs to assist tenant A. Preserve the support actor, define its permitted operations, and avoid unrestricted impersonation as the default solution.

!!! info "External Resources"
    - [Mass Assignment Prevention](https://cheatsheetseries.owasp.org/cheatsheets/Mass_Assignment_Cheat_Sheet.html) (OWASP)
    - [Authorization Policy Models](https://cheatsheetseries.owasp.org/cheatsheets/Authorization_Cheat_Sheet.html) (OWASP)

## 11. Can Internal Services Trust an X-User Header from the Gateway?

**Study first:** [Proxy identity boundaries](mtls.md#identity-and-authorization-boundaries).

**Scenario:** The gateway validates login and forwards `X-User: user-42`. Internal services accept the header because they are on a private network. What must be true for this design to be safe?

??? note "Answer guidance and common trap"
    The system needs an enforced and authenticated trust boundary, not merely a header naming a user. Strip caller-supplied identity headers at ingress, authenticate the gateway/service channel, restrict direct access, and define who can assert which identities. A signed internal assertion can help when validated for issuer, recipient, expiry, and purpose.

    Services must still enforce resource/action policy at the correct layer. A deliberate centralized enforcement design is possible, but bypass paths, alternate callers, and privileged internal workloads must be included in its threat model.

    **Trap:** Network location is treated as proof of identity and authorization.

    **Test:** A direct service request and a request with a forged ingress header fail, while an authenticated, policy-approved gateway call succeeds.

??? example "Worked example"
    The external caller supplies a header that must not become trusted identity.

    ```http
    GET /billing/accounts HTTP/1.1
    Host: app.example
    X-User: administrator
    ```

    ```python
    # Gateway pseudocode: remove untrusted assertions before forwarding.
    headers.remove_all("X-User")
    user = validate_external_credential(request)
    headers.set("X-User", user.id)
    forward_over_authenticated_gateway_channel(headers)

    # Billing pseudocode: the authenticated internal peer must be this gateway.
    require(peer.identity == "spiffe://example/gateway")
    require(peer.may_assert_users)
    user = resolve_gateway_asserted_user(request.headers["X-User"])
    authorize(user, "billing.read", requested_account)
    # A reporting workload with a different valid CA-issued certificate fails above.
    ```

    Direct bypass paths must be restricted or enforce equivalent authenticated authority checks. A header string alone never proves who asserted it.

**Follow-up:** If another workload has a certificate from the same CA, can it assert arbitrary users? No; authenticate the expected asserting identity and its permitted authority, not just CA membership.

!!! info "External Resources"
    - [Microservices Authentication and Identity Propagation](https://cheatsheetseries.owasp.org/cheatsheets/Microservices_Security_Cheat_Sheet.html) (OWASP)
    - [Identity Propagation Trust Patterns](https://cheatsheetseries.owasp.org/cheatsheets/Identity_Propagation_Patterns_Cheat_Sheet.html) (OWASP)

## 12. Can a Service Use Its Own Privileges When the User Is Denied?

**Study first:** [User delegation](design-patterns.md#3-preserve-user-delegation-across-service-calls).

**Scenario:** A user asks orders to retrieve a billing record. The user's downstream token is rejected. Orders retries using its powerful billing service credential, which succeeds. Is this a resilience feature?

??? note "Answer guidance and common trap"
    It changes the authority of the operation and can make orders a confused deputy: the caller induces a more privileged service to perform an action the caller was not allowed to request. Distinguish failed authentication from insufficient authorization; neither justifies silently escalating to application authority.

    For delegated actions, preserve the user subject and authenticated service actor under an explicit downstream policy. A supported token exchange can issue a billing-specific credential, but the STS must enforce allowed audience and permissions; exchange is not automatically safe downscoping. Genuine application-only tasks need a separate authorized contract.

    **Trap:** A successful technical retry is assumed to preserve security semantics.

    **Test:** A user denied access to a billing record stays denied through the orders API, even though orders has permissions for other legitimate system tasks.

??? example "Worked example"
    A service credential must not silently replace denied user authority.

    ```python
    # Application pseudocode: unsafe fallback, shown only to identify the bug.
    result = billing.read(invoice_id, token=user_delegated_token)
    if result.denied:
        # BUG: billing now sees application authority instead of the denied user.
        result = billing.read(invoice_id, token=reconciliation_service_token)
    return result
    ```

    ```python
    # Delegated design: obtain billing-specific authority under enforced STS policy.
    token = sts.exchange(subject_token=user_token, audience="billing-api",
                         scope="invoice.read", actor=authenticated_orders_service)
    result = billing.read(invoice_id, token=token)
    if result.denied:
        deny()  # Do not escalate to application-only authority.
    # Billing checks user entitlement AND the acting service's delegated capability.
    ```

    The exchange call is conceptual. A real protocol uses the provider's supported token-exchange profile and authenticated service identity; a client cannot merely claim a trusted actor.

**Follow-up:** Which identity belongs in the audit log? Preserve both the affected subject and acting service under the profile, plus the decision and resource; do not overwrite the actor with the user.

!!! info "External Resources"
    - [Delegation, Impersonation, and Actor Claims](https://www.rfc-editor.org/rfc/rfc8693.html#section-1.1) (IETF)

## 13. A SAML Signature Validates. Can You Use Any Assertion in the Response?

**Study first:** [SAML signed-content validation](saml.md#assertion-validation).

**Scenario:** A SAML response contains multiple XML elements. The signature library reports success, but application code separately searches for the first assertion and reads its subject. What could go wrong?

??? note "Answer guidance and common trap"
    The application may consume a different assertion from the content actually protected by the signature. This mismatch is central to signature-wrapping attacks. Use the library's validated result under the chosen SAML profile; reject duplicate IDs and ambiguous structure rather than performing an unrelated XML search.

    Also validate trusted issuer/key, intended audience, recipient/destination as required, time conditions, SP-initiated request correlation, status, and replay policy. A signed assertion for another SP is not authority to establish this application's session.

    **Trap:** “Some signature is valid” is treated as “all parsed identity data is trusted.”

    **Test:** A trusted signed assertion for another audience, or an unsigned injected assertion beside valid signed content, cannot change the authenticated account.

??? example "Worked example"
    This deliberately simplified XML illustrates the parser mismatch. It is not a valid SAML response or a reusable attack payload.

    ```xml
    <Response>
      <Assertion ID="B"><Subject>administrator</Subject></Assertion>
      <Assertion ID="A"><Subject>alice</Subject></Assertion>
      <!-- Verified signature references A, not B; actual XML signature omitted. -->
      <Signature><Reference URI="#A" /></Signature>
    </Response>
    ```

    ```python
    # Library API names are illustrative.
    validated = saml_library.validate_response(raw_xml, trusted_configuration)
    # UNSAFE: user = parse(raw_xml).first_assertion().subject  # Could select B.
    user = validated.authenticated_assertion.subject         # Consume validated A.
    # Library validation must also enforce the selected SAML profile's coverage,
    # audience, recipient, time, request correlation, status, and replay policy.
    ```

    Use the library's documented validated result. A signature success flag does not authorize a second unrelated XML search.

**Follow-up:** Does POST binding fix this? No; transport binding alone does not ensure the application consumes the correct authenticated content.

!!! info "External Resources"
    - [SAML Validation and Signature-Wrapping Defenses](https://cheatsheetseries.owasp.org/cheatsheets/SAML_Security_Cheat_Sheet.html) (OWASP)

## 14. If a Job Was Authorized When Queued, Can It Run Tomorrow?

**Study first:** [Queued-operation authority](design-patterns.md#queued-operations-and-durable-approval).

**Scenario:** A user schedules a sensitive export. Before execution, the user is removed from the tenant and the requested resource changes ownership. The job contains yesterday's access token and an `authorized=true` field. Should it proceed?

??? note "Answer guidance and common trap"
    Define the business authorization contract first. A job requiring current entitlement should recheck the user/resource/tenant policy at execution. A deliberately durable approval may survive some later changes, but needs server-controlled scope, integrity, expiry/cancellation, and parameter binding. A caller-controlled boolean and an expired bearer token establish neither model.

    Authenticate the worker separately. Bind approved parameters to the actual operation and coordinate final authorization with execution to address races. Use idempotency and replay controls where the operation requires them; they do not replace authorization.

    **Trap:** An old check is assumed to authorize any later operation with similar parameters.

    **Test:** Change membership, ownership, and export parameters between approval and execution and verify the explicitly chosen policy in each case.

??? example "Worked example"
    The queue records a requested operation, not a bearer token pretending to be tomorrow's approval.

    ```json
    {
      "requestedBy": "alice", "tenant": "tenant-A", "action": "invoice.export",
      "invoiceIds": [42, 43], "destination": "approved-bucket", "jobId": "job-8"
    }
    ```

    ```python
    # Current-entitlement contract: application pseudocode at worker execution.
    authenticate_worker()
    job = load_server_owned_job("job-8")
    authorize_current(job.requested_by, job.action, job.invoice_ids, job.tenant)
    execute_once(job)  # Coordinate final authorization with execution.
    # Alice removed from tenant-A before execution -> deny under this contract.

    ```

    A different, deliberately durable-approval contract uses a protected approval record instead:

    ```python
    # Alternative execution contract, not a fallback after current-authority denial.
    authenticate_worker()
    job = load_server_owned_job("job-8")
    approval = load_approval(job.approval_id)
    require(not approval.expired and not approval.cancelled)
    require(approval.bound_parameters == job.parameters)
    check_explicit_survival_rules(approval, job)
    consume_approval_and_execute_coordinated(approval, job)
    # Never read approval then execute without guarding a cancellation/replay race.
    # External side effects also require idempotency/recovery handling.
    ```

    These are alternative contracts, not two sequential fallbacks. A destination change must not inherit approval for the original destination.

**Follow-up:** Must a permission removal cancel a transaction already committed? Explain the system's execution boundary and business policy instead of promising retroactive revocation.

!!! info "External Resources"
    - [Transaction Data Binding and Final Authorization Checks](https://cheatsheetseries.owasp.org/cheatsheets/Transaction_Authorization_Cheat_Sheet.html) (OWASP)

## 15. Can You Cache an Allow Decision by User ID and Keep Serving During an Outage?

**Study first:** [Authorization caches](comparison.md#authorization-decisions-and-their-caches).

**Scenario:** A shared authorization cache uses only `userId` as its key. During a policy-service outage, the application reuses any cached allow decision. One user belongs to multiple tenants with different permissions. What is the failure mode?

??? note "Answer guidance and common trap"
    An allow decision is about a particular subject, action, resource, tenant, and relevant context. Omitting policy inputs can reuse permission for a different operation. Define a cache key covering the decision's actual inputs, a validity bound, and invalidation/version behavior. Permission changes and data changes can invalidate the result even if the identity is unchanged.

    **Concrete collision:** If `cache[u7] = allow` was produced by reading invoice 42 in tenant A, reusing it for `u7 / tenant-B / invoice-99 / write` changes three policy inputs. A decision key could include `(subject, tenant, action, resource, relevant context/policy version)`; the actual inputs depend on the policy. Even that key needs a freshness bound when membership or ownership changes. See [authorization decisions and their caches](comparison.md#authorization-decisions-and-their-caches).

    Separate authentication verification from current authorization. A known unexpired credential may remain verifiable with trusted cached keys; that does not make an unrelated cached allow valid. Define which operations can use bounded cached policy and which must stop when fresh evidence is unavailable. Unknown keys and expired credentials are not resolved by a fail-open rule.

    **Trap:** Preserving availability is used to silently broaden the scope or lifetime of an earlier permission.

    **Test:** The same subject's allow for tenant A/read cannot authorize tenant B/write; expired policy evidence is handled according to the documented operation-specific outage policy.

??? example "Worked example"
    The cache key must distinguish the operation that was actually authorized.

    ```python
    # Application pseudocode: collision in the unsafe design.
    cache["u7"] = True  # Produced by tenant-A / invoice-42 / read.
    # Reusing cache["u7"] for tenant-B / invoice-99 / write incorrectly allows it.

    # Example key; include every policy input actually affecting the decision.
    key = (subject.id, tenant.id, action, resource.id, context.version, policy.version)
    decision = cache.get(key)
    if decision is None or now >= decision.valid_until:
        decision = policy_service.evaluate(subject, tenant, action, resource, context)
        # Set valid_until from authority evaluation + allowed cache TTL.
        # Cache hits must not move this deadline; policy versions alone do not
        # invalidate changed ownership unless the design propagates that change.
    if decision.unavailable:
        deny()  # This operation's policy requires fresh evidence on a cache miss.
    require(decision.allowed)
    # A matching key can still become stale after membership/ownership changes.
    ```

    Specify invalidation and maximum cache age as well as the key. An unexpired authentication token cannot replace a missing authorization decision.

**Follow-up:** What would you measure? Cache age, invalidation delay, authorization failures, and outage behavior, without logging raw credentials or sensitive resource contents.

!!! info "External Resources"
    - [Authorization Checks and Safe Failure](https://cheatsheetseries.owasp.org/cheatsheets/Authorization_Cheat_Sheet.html) (OWASP)
    - [Token Introspection Cache Considerations](https://www.rfc-editor.org/rfc/rfc7662.html#section-4) (IETF)

## 16. Why Is a Longer-Lived Refresh Token Different from a Long-Lived Access Token?

**Study first:** [Refresh exchange](oauth2-oidc.md#refresh-exchange-and-reuse-detection) and [lifetime selection](oauth2-oidc.md#selecting-lifetimes).

**Scenario:** Access tokens last five minutes, but the refresh grant can last a day. Has the design merely moved the same risk elsewhere?

??? note "Answer guidance and common trap"
    Renewal is concentrated at the authorization server, where client binding, expiry, grant revocation, rotation/reuse detection, or sender constraints can be checked. The refresh token is not sent to each API. This can reduce credential distribution and allow renewal to stop without querying the issuer on every API request.

    A stolen usable refresh token is still highly valuable: it may mint replacement access tokens until renewal is stopped. Protect its storage and set idle/absolute limits. Short access expiry is insufficient if the attacker can continually renew. The design trades different exposure and enforcement points; a longer lifetime is not safe merely because the credential is called a refresh token.

    **Test:** Revoke the renewable grant, confirm refresh fails, and separately measure how long a previously copied access token still works.

??? example "Worked example"
    The refresh endpoint and API accept different credentials, with separate deadlines.

    ```http
    POST /token HTTP/1.1
    Host: id.example
    Content-Type: application/x-www-form-urlencoded

    grant_type=refresh_token&client_id=public-client&refresh_token=R1
    ```

    ```json
    {
      "access_token": "A2", "token_type": "Bearer", "expires_in": 300,
      "refresh_token": "R2"
    }
    ```

    ```text
    # R1/A2/R2 are readable placeholders, not real token strings.
    10:00  A1 issued; access-token expiry is 10:05
    10:04  R1 redeemed -> A2 and replacement R2; A2 expires 10:09
    10:05  Grant revoked -> subsequent R2 refresh fails under that policy
    10:06  A2 may still pass API checks until 10:09 without earlier invalidation
    # Sending R2 to the API is invalid: it is a token-endpoint credential.
    ```

    This public-client illustration omits confidential-client authentication. Protecting renewal centrally does not make a stolen usable refresh token low risk.

## 17. Does Refresh-Token Reuse Always Prove Theft?

**Study first:** [Rotation and concurrent refreshes](oauth2-oidc.md#refresh-exchange-and-reuse-detection).

**Scenario:** Two tabs redeem the same rotating refresh token at nearly the same time. One succeeds; the other receives `invalid_grant`. Should the application blindly retry the old credential?

??? note "Answer guidance and common trap"
    Concurrent legitimate callers or a lost response can produce reuse, so it is not conclusive attribution of theft. The issuer nevertheless cannot safely assume that the second caller is legitimate. The RFC 9700 rotation defense revokes the affected active refresh credential when reuse is detected. Coordinate refresh per grant and store replacements safely; follow documented recovery behavior rather than repeatedly presenting an invalid predecessor.

    Rotation does not reset a separately fixed absolute deadline. Browser refresh-token rotation must not extend a pre-established original expiry. Nor does rotation automatically invalidate previously issued access tokens.

    **Test:** Exercise concurrent calls and a lost refresh response. Verify bounded recovery, no indefinite old-token acceptance, and preservation of the original absolute deadline.

??? example "Worked example"
    A legitimate race and theft can produce the same reused-predecessor signal.

    ```text
    10:00:00.000  Tab A -> POST /token with R0
    10:00:00.020  Issuer -> A1 + R1; invalidate R0
    10:00:00.030  Tab B -> POST /token with R0
    10:00:00.040  Issuer -> invalid_grant; detected reuse revokes active refresh R1
    # Issuer cannot reliably tell whether Tab B is legitimate or an attacker.
    # Absolute grant deadline remains 18:00; rotating R0 did not reset it.
    ```

    ```python
    # Coordination pseudocode; all consumers of this grant must share the mechanism.
    with shared_refresh_lock(grant_id):
        state = token_store.reload(grant_id)  # Do not use a stale tab's token copy.
        if state.access_token_needs_refresh:
            response = refresh_using_current_credential(state)
            token_store.atomic_replace(grant_id, response)
    # A lost response still needs provider-documented recovery, not blind R0 retries.
    ```

    A lock local to one tab cannot coordinate another tab or worker. The implementation must match where the credential is shared.

## 18. Does a Newly Issued ID Token Prove Recent Human Authentication?

**Study first:** [Independent lifetime policies](oauth2-oidc.md#independent-lifetime-policies) and [sensitive-action freshness](design-patterns.md#7-require-stronger-authentication-for-sensitive-actions).

**Scenario:** The client refreshes credentials at 14:00 and receives an ID token with a new `iat`. The user signed in at 09:00. A payout change requires authentication within five minutes. Is the new issuance time enough?

??? note "Answer guidance and common trap"
    No. Issuance and user authentication are different events. Check trusted authentication-time evidence and the required context; obtain a new interactive authentication when the recency requirement is not met. Refresh can occur while the user is absent.

    Conversely, expiry of the ID token originally used to establish a session does not by itself terminate that application's server session. Session idle/absolute limits and action-specific freshness must be enforced separately.

    **Test:** Refresh an old authenticated grant and confirm it does not silently satisfy recent-authentication policy. Also test session expiry independently of the ID token's expiry.

??? example "Worked example"
    These decoded claims use fixed Unix timestamps; the token itself would also need valid signature and profile checks.

    ```json
    {
      "iss": "https://id.example", "sub": "alice", "aud": "web-client",
      "auth_time": 1800000000,
      "iat": 1800018000,
      "exp": 1800018300
    }
    ```

    ```python
    # Application pseudocode: validate signature/profile before trusting these claims.
    claims = validate_id_token(raw_token, expected_issuer, expected_client_id)
    # In this fixed-clock fixture, now equals issuance, five hours after login.
    now = 1800018000
    require("auth_time" in claims)           # Required recency evidence.
    require(claims["auth_time"] <= now + allowed_clock_skew)  # Reject future evidence.
    require(now < claims["exp"])                 # Token has not expired.
    require(now - claims["auth_time"] <= 300)    # Fails: 18000 > 300 seconds.
    # Checking now - iat would incorrectly treat refresh as recent human login.
    # Request supported max_age=300 in a new authentication interaction, then
    # validate returned auth_time and the action's required authentication context.
    ```

    The application session has independent idle/absolute limits. Passing a recency check also does not bind approval to an arbitrary payout destination.

## 19. Can an API Accept an Unknown Signing Key During an Issuer Outage?

**Study first:** [Trusted keys and rotation](jwt.md#trusted-keys-and-rotation).

**Scenario:** Known cached keys still verify existing tokens, but a new token contains an unknown `kid`. Fetching the configured JWKS fails. May the API skip verification because the outage is temporary?

??? note "Answer guidance and common trap"
    No. A bounded refresh can retrieve a new trusted key when available; failure does not establish trust in the unknown key. Reject credentials that cannot be validated. Known cached keys may still validate appropriate unexpired tokens under the deployment's cache policy.

    Planned rollover publishes new keys early and retains old public keys for valid outstanding tokens. Compromise response instead requires rejecting affected signing authority; simply publishing a replacement leaves old-key tokens potentially usable.

    **Test:** Unknown IDs fail during an outage; random IDs cannot trigger unlimited key-fetch traffic; retirement/compromise policy reaches every verifier within its stated bound.

??? example "Worked example"
    An untrusted header can select only within the configured trusted key context.

    ```json
    {
      "alg": "RS256", "kid": "K2", "typ": "at+jwt"
    }
    ```

    ```python
    # Verifier pseudocode; a maintained library implements actual JOSE verification.
    header = inspect_untrusted_header(raw_token)
    require(header["alg"] in configured_algorithms)
    key = trusted_cache.find(configured_issuer, header["kid"])
    if key is None:
        bounded_refresh_from_configured_jwks()  # Rate-limited, coordinated fetch.
        key = trusted_cache.find(configured_issuer, header["kid"])
    if key is None:
        reject_token()  # Outage is not permission to skip signature verification.
    claims = verify_signature_and_profile(raw_token, key, configured_profile)
    # Never fetch an arbitrary jku/x5u supplied by this token to establish trust.
    ```

    Known K1 may still verify appropriate unexpired tokens under cache policy. If K1 is compromised, adding K2 alone does not remove K1's acceptance; distribute explicit rejection of the affected authority.

## Self-Assessment

Use this rubric to assess your answers:

| Score | Evidence in your answer |
|---|---|
| 0 | Repeats the incorrect assumption or lists acronyms without identifying the decision |
| 1 | Spots the issue but cannot explain why the control is insufficient |
| 2 | Explains the boundary and a suitable correction under stated assumptions |
| 3 | Also compares a realistic alternative, identifies residual risk, and gives a rejection/failure test |

Repeat questions after changing one assumption: add another tenant, remove the browser, steal only the token, compromise the execution environment, or make the issuer unavailable. Rehearse the reasoning rather than memorizing the first sentence of an answer.
