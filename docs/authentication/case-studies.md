# Authentication Incident Case Studies

These case studies examine authentication failures and defenses. Each summarizes a published incident, explains its design implications, and includes interview practice questions.

## Cloudflare, July 2022: Phishing-Resistant Login

**Reported facts:** In a report published on August 9, 2022, Cloudflare described a phishing campaign on July 20. Three employees entered credentials into the phishing site. The company reported that its required FIDO2 security keys prevented the attacker from completing authentication and that no Cloudflare systems were compromised in this event. It also reset exposed credentials and active sessions. [Cloudflare's incident report](https://blog.cloudflare.com/2022-07-sms-phishing-attacks/).

```text
Employee password entered at fake site
                 |
Attacker attempts login at real provider
                 |
Required origin-bound authenticator proof unavailable
```

**Design analysis:** The decisive property was not simply the number of factors. A reusable code can be relayed in real time; an authenticator scoped to the legitimate relying party resists signing for an unrelated phishing origin. The receiving service must still validate the ceremony and expected origin correctly. This case does not demonstrate protection against every endpoint compromise, recovery attack, or theft of a session created after authentication.

**Credential and recipient:** The target login service required an authenticator proof for its own relying-party context. The stolen password alone could not satisfy that ceremony. This successful boundary applies to login; a later application session is a separate credential. See [the WebAuthn phishing example](webauthn.md#registration-and-authentication).

**Practice prompt:** “Our workforce uses passwords plus one-time codes. What would you change, and what remains at risk?”

A strong answer proposes phishing-resistant authentication, secure enrollment/recovery, consistent enforcement across applications, and a session-theft response. It avoids blaming users or claiming that employee training makes a phishing-resistant protocol unnecessary.

**Follow-up:** An administrator asks for a temporary password-only exception. Describe who can approve it, its scope and duration, monitoring, and safer recovery options. Explain that the exception changes the authentication guarantee for that account.

!!! info "External Resources"
    - [WebAuthn Security Considerations](https://www.w3.org/TR/webauthn-3/#sctn-security-considerations) (W3C)

## GitHub, April 2022: Stolen Third-Party OAuth Tokens

**Reported facts:** GitHub's April 15 report, updated through April 27, described abuse of stolen OAuth user tokens issued to Heroku and Travis CI integrations. Attackers used the tokens to list organizations and repositories and download selected private repositories. GitHub said it did not believe the tokens had been stolen through compromise of GitHub's systems. [GitHub's incident report](https://github.blog/news-insights/company-news/security-alert-stolen-oauth-user-tokens/).

```text
User authorizes integration
          |
Integration retains usable API credential
          |
Credential stolen --> API accepts permitted token use
```

**Design analysis:** Strong interactive login does not force a fresh login for every use of an issued integration credential. Its effective scope, audience, accessible repositories, storage, and lifecycle determine exposure. Using a standards-based delegation protocol does not make the integration provider's credential storage trustworthy.

**Credential and recipient:** The repository API accepted an already issued integration token. The attacker reused existing delegated authority rather than proving the user's password again. Short access-token expiry can reduce exposure, but a stolen usable refresh grant can issue replacements; see [refresh mechanics](oauth2-oidc.md#refresh-exchange-and-reuse-detection) and [lifetime selection](oauth2-oidc.md#selecting-lifetimes). These are design considerations, not claims about which token format or refresh policy the incident used.

**Practice prompt:** “A build integration needs repository access. How would you limit damage if that integration is compromised?”

Defend repository-specific installation or authorization where available, minimal permissions, credential isolation, and renewable access with a deliberate exposure bound. Explain integration revocation, review of repository access, and investigation of downstream secrets. Where supported, assess sender constraints, while recognizing that compromise of the integration's signing/key-use environment can defeat token-only theft protections.

**Follow-up:** “Would rotating every user's password solve it?” Identify the credential actually accepted by the API. Password changes do not universally revoke independent integration tokens; invoke the provider's relevant grant/token controls and verify their effect.

!!! info "External Resources"
    - [OAuth Token Replay Prevention](https://www.rfc-editor.org/rfc/rfc9700.html#section-2.2) (IETF)
    - [OAuth Token Revocation](https://www.rfc-editor.org/rfc/rfc7009.html) (IETF)

## Cloudflare, October 2023: Session Material in Support Data

**Reported facts:** Cloudflare's October 20 report described unauthorized activity detected on October 18 using an administrative Okta session. It said the attacker appeared to have obtained a session token from a support ticket after access to Okta's support system. Cloudflare reported that its detection and response contained the incident without impact to customer information or systems. [Cloudflare's incident report](https://blog.cloudflare.com/how-cloudflare-mitigated-yet-another-okta-compromise/).

```text
Administrative login establishes session
                 |
Support data exposes usable session material
                 |
Attacker reuses session without repeating original login
```

**Design analysis:** Authentication strength and session handling are separate controls. Moving credential-bearing diagnostics into a support platform can create a new trust boundary. Existing session authority may be usable without repeating the original authenticator ceremony.

**Credential and recipient:** An administrative service accepted session material representing an earlier successful authentication. Copying that material could avoid a new authenticator ceremony. Containment therefore needs to invalidate the state accepted by that service and inspect other authority created or retained by the attacker. See [session invalidation](sessions.md#expiry-rotation-and-revocation) and [logout scopes](oauth2-oidc.md#revocation-and-current-authorization).

**Practice prompt:** “Support needs a browser trace to debug SSO. What must the diagnostic workflow protect?”

Require credential-aware collection/redaction, restricted support access, minimal retention, and incident response for accidental disclosure. Treat session cookies, bearer headers, authorization codes, and reset links as potential credentials. For administrative actions, consider short privilege windows, fresh authentication, and supported session/device binding without claiming universal browser support.

**Follow-up:** “We revoked the IdP session. Is the incident over?” Inventory local application sessions, API grants, credentials created during unauthorized access, and cached authorization decisions. Verify the actual revocation boundary instead of assuming a single logout action reaches every relying application.

!!! info "External Resources"
    - [Session Logging, Expiration, and Invalidation](https://cheatsheetseries.owasp.org/cheatsheets/Session_Management_Cheat_Sheet.html) (OWASP)

## Compare the Failed Assumptions

Use the cases to ask which artifact the attacker controls and which recipient accepts it. The controls are complementary rather than interchangeable.

| Attacker capability | Control to reason about | Incomplete answer |
|---|---|---|
| Password entered at a phishing origin | Origin-bound authentication and enforced authenticator policy | Add any second factor and assume real-time relay is impossible |
| Usable integration token stolen | Scope/resource limits, protected credential storage, lifecycle, and supported sender constraints | The user's MFA will be requested before every API call |
| Active administrative session copied | Session exposure prevention, action policy, invalidation, and detection | A strong initial login makes copied sessions harmless |

For each case, practice naming the accepted credential, an effective preventive control, a residual risk, and a verifiable containment action. Then apply the answer to a [machine integration](design-patterns.md#1-expose-an-application-to-an-external-machine-client), [browser BFF](design-patterns.md#2-protect-a-browser-application-with-a-backend-for-frontend), or [sensitive admin operation](design-patterns.md#7-require-stronger-authentication-for-sensitive-actions).

!!! info "External Resources"
    - [Authentication Controls and Reauthentication](https://cheatsheetseries.owasp.org/cheatsheets/Authentication_Cheat_Sheet.html) (OWASP)
    - [OAuth Security Best Current Practice](https://www.rfc-editor.org/rfc/rfc9700.html) (IETF)
