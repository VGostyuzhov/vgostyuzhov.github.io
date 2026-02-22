# Authentication Fundamentals

Authentication (AuthN) is the process of verifying that an entity (user, device, or service) is who they claim to be. It is distinct from Authorization (AuthZ), which determines what an authenticated entity is allowed to do.

## AuthN vs. AuthZ

| Concept | Authentication (AuthN) | Authorization (AuthZ) |
|---------|-----------------------|-----------------------|
| **Question** | Who are you? | What can you do? |
| **Evidence** | Credentials (Passwords, Tokens, Certs) | Permissions, Roles, Scopes |
| **Example** | Logging into an account | Accessing a specific file or API |

## Factors of Authentication

Security is often strengthened by requiring multiple "factors" of authentication (MFA):

1.  **Something you know**: Password, PIN, Security questions.
2.  **Something you have**: Security key (YubiKey), TOTP app, SMS code, Smart card.
3.  **Something you are**: Biometrics (Fingerprint, FaceID, Retina scan).
4.  **Somewhere you are**: (Contextual) IP-based geofencing, GPS location.

## Protocol Comparison Matrix

| Protocol | Primary Use Case | Transport | Data Format |
|----------|------------------|-----------|-------------|
| **[Sessions & Cookies](sessions.md)** | Traditional Web Apps | HTTPS | Set-Cookie headers |
| **[JWT](jwt.md)** | Modern APIs, Microservices | HTTPS Header | JSON (Base64URL) |
| **[OAuth 2.0 / OIDC](oauth2-oidc.md)** | Modern SSO, Third-party Auth | HTTPS / JSON | JSON / JWT |
| **[SAML 2.0](saml.md)** | Enterprise SSO | HTTPS / XML | XML Assertions |
| **[mTLS](mtls.md)** | Service-to-Service (Zero Trust) | TLS Handshake | X.509 Certificates |
| **[WebAuthn](webauthn.md)** | Passwordless, Phishing-resistant | Browser API | Public Key Crypto |
| **[Directory Services](directory-services.md)** | Internal Network / Legacy | LDAP / Kerberos | ASN.1 / Binary |

## Choosing the Right Protocol

- **Building a standard monolith?** Use **Sessions & Cookies**.
- **Building a Mobile App or SPA?** Use **OAuth 2.0 + OIDC (PKCE)**.
- **Securing internal microservices?** Use **mTLS** or **JWT**.
- **Integrating with an Enterprise IDP (Active Directory)?** Use **SAML** or **OIDC**.
- **Want to eliminate passwords?** Use **WebAuthn**.
