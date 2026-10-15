# Authentication

Authentication establishes an identity; authorization decides which operations that identity may perform. A successful login does not grant unrestricted access. Applications must also manage session lifetime, recovery, credential compromise, and authorization on each protected operation.

Study sessions, JWT, and OAuth/OIDC first. Then compare SAML federation and WebAuthn user authentication. Study mTLS and directory services for service and enterprise environments. For browser origin rules and injection risks, see [Web Security Fundamentals](../web-security/index.md).

## Follow One Account Through the System

A **principal** is an identity to which a system assigns authority, such as a user or service. A **credential** supplies evidence used to authenticate that principal. An **issuer** creates a credential or assertion; a **recipient** validates it. A **trust boundary** separates components or parties whose statements are not accepted without checks.

Consider a browser user viewing an invoice:

1. The user proves control of an authenticator to an identity provider. The login method might be a password or WebAuthn credential.
2. The provider issues an OIDC ID token to the application. Its **claims** are named assertions, such as the issuing provider and authenticated subject. The application validates them rather than treating browser-delivered data as trusted.
3. The application creates a local session and gives the browser its session cookie. Later requests present that cookie instead of repeating the original authentication.
4. If a separate invoice API is involved, the backend presents an access token intended for that API. The API validates that credential independently.
5. The API checks whether the principal may read this particular invoice in this tenant. Login and a general read scope do not establish invoice ownership.
6. Session expiry, token refresh, permission changes, and logout affect different state. Ending one credential does not necessarily end all the others.

The same user appears in several components, but each recipient accepts a particular kind of evidence for a particular purpose. The protocol pages explain those checks; the design pages show how to combine them.

!!! info "External Resources"
    - [Authentication and Session Management](https://cheatsheetseries.owasp.org/cheatsheets/Authentication_Cheat_Sheet.html) (OWASP)

## Architecture and Interview Practice

- **[Choosing an Authentication Architecture](comparison.md):** Compare caller types, protocols, client proofs, token formats, and revocation tradeoffs; practice explaining what to use when.
- **[Authentication Design Patterns](design-patterns.md):** Seven worked designs covering external machine clients, browser BFFs, service delegation, CLIs, enterprise SSO, CI workloads, and sensitive operations.
- **[Authentication Incident Case Studies](case-studies.md):** Analyze phishing-resistant login, stolen integration tokens, and exposed administrative sessions using primary incident accounts.
- **[Tricky Interview Questions](interview-questions.md):** Nineteen authentication/authorization scenarios with hidden answer guidance, common traps, tests, and follow-ups.

## Core Topics

| Topic | Question it answers | Prerequisites |
|---|---|---|
| [Sessions & Cookies](sessions.md) | How does a browser retain a login, and how is it terminated? | HTTP requests, cookies, HTTPS |
| [JWT](jwt.md) | How are claims represented and validated across services? | Encoding, MACs, digital signatures |
| [OAuth 2.0 & OIDC](oauth2-oidc.md) | How are API access and federated login delegated? | HTTP redirects, sessions, JWT validation |
| [SAML 2.0](saml.md) | How does enterprise browser SSO exchange identity assertions? | Federation, XML signatures |
| [mTLS](mtls.md) | How do services authenticate peers through certificates? | TLS, PKI, trust stores |
| [WebAuthn](webauthn.md) | How can a user authenticate with a public-key credential? | Challenges, signatures, browser origins |
| [Directory Services](directory-services.md) | How do LDAP and Kerberos support enterprise identities? | Directories, tickets, shared-key cryptography |

## Concepts to Keep Separate

These mechanisms solve different parts of an authentication system and can be combined.

| Concept | Role | Common misconception |
|---|---|---|
| Credential | Evidence used during authentication, such as a password or signed challenge | An identifier alone proves identity |
| Session | Continuity of an authenticated interaction | A session cookie is valid forever unless the browser closes |
| Cookie | Browser-managed storage and request transport | Cookies and JWTs are competing protocols |
| JWT | Claims representation, commonly integrity protected | Decoding a token validates its claims |
| OAuth access token | Credential for a resource server | Every access token is a JWT or proves a human login |
| OIDC ID token | Authentication statement intended for a client | An API should accept any valid ID token |
| Authorization policy | Checks subject, action, resource, tenant, and context | Successful authentication implies resource access |

**Example:** A user signs in through OIDC, the application creates a server-side session, and a browser cookie carries the opaque session identifier. The backend can separately obtain an access token for an API. The browser does not need to retain that API token.

!!! info "External Resources"
    - [Authentication Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Authentication_Cheat_Sheet.html) (OWASP)
    - [Authorization Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Authorization_Cheat_Sheet.html) (OWASP)

## Learning Objectives

1. Trace the messages and trust boundaries in a login or service-authentication flow.
2. Explain which party issues, stores, presents, and validates each credential.
3. Reject a correctly signed credential intended for a different recipient or purpose.
4. Explain replay, session fixation, CSRF, recovery, and credential lifecycle controls.
5. Apply authorization after authentication and account for tenant and object ownership.

After the core pages, use the comparison to select an architecture, work through relevant patterns, and test your reasoning with cases and questions.
