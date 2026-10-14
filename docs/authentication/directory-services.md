# Directory Services (LDAP & Kerberos)

LDAP accesses directory data and supports authentication through bind operations. Kerberos authenticates principals using tickets and session keys. Active Directory commonly combines both, but directory access, authentication, and service authorization remain distinct functions.

## LDAP (Lightweight Directory Access Protocol)

LDAP clients search and modify entries such as users, groups, and devices. Server-side access controls determine which directory operations are allowed after authentication.

**Bind methods:** A name/password simple bind is one authentication mechanism. LDAP also supports anonymous/unauthenticated states and SASL mechanisms, including enterprise integrations with Kerberos. Do not describe every LDAP deployment as sending a plaintext password.

For a password-based application flow:

1. Establish TLS through a supported LDAPS connection or a required, successful StartTLS upgrade. Validate the server identity and chain before transmitting credentials.
2. Resolve the user's entry through a least-privileged search if necessary. Build the search with library-supported safe filters; filter and DN escaping rules differ.
3. Perform the user bind with the resolved distinguished name (DN), the entry's directory identifier, and supplied password. Reject empty passwords rather than interpreting an unauthenticated bind as a successful login.
4. Check the bind result and enforce the application's authorization policy. Do not reuse a connection under an unintended identity in a connection pool.

**Failure example:** StartTLS fails or the server certificate names another host. Abort; falling back to an unprotected simple bind would disclose credentials.

| Threat | Primary controls |
|---|---|
| LDAP injection | Use safe filter construction and context-specific escaping; do not concatenate untrusted filter syntax |
| Credential interception | Require an authenticated confidentiality-protecting channel for password binds; do not silently downgrade |
| Excessive directory permissions | Restrict bind-account search/modify rights and query scope |
| Authentication-state confusion | Check bind outcome, reject empty passwords, and manage connection identity explicitly |

!!! info "External Resources"
    - [LDAP Authentication Methods and StartTLS](https://www.rfc-editor.org/rfc/rfc4513.html) (IETF)
    - [LDAP Injection Prevention](https://cheatsheetseries.owasp.org/cheatsheets/LDAP_Injection_Prevention_Cheat_Sheet.html) (OWASP)

## Kerberos

Kerberos uses a Key Distribution Center (KDC), comprising an Authentication Service (AS) and Ticket Granting Service (TGS). A **principal** names a user or service; a service principal name identifies the intended service in an Active Directory deployment.

In a password-based Kerberos V5 exchange, pre-authentication methods and key types depend on the deployment.

1. **AS exchange:** The client requests a Ticket Granting Ticket (TGT), providing pre-authentication evidence as required. The response contains a TGT encrypted for the TGS and a client/TGS session key protected for the client.
2. **TGS exchange:** The client cannot decrypt the TGT. It presents it with an authenticator protected by the client/TGS session key to request a ticket for a specific service.
3. **Service ticket:** The TGS returns a ticket encrypted for that service and a client/service session key protected for the client.
4. **Application exchange:** The client presents the service ticket and a fresh authenticator. The service validates them, including freshness/replay checks. Mutual authentication can include a service response.
5. **Authorization:** The service applies its access policy to the authenticated identity. SSO does not grant access to every resource.

A copied ticket alone is not the complete authentication proof; stolen credential caches may include both tickets and usable session keys. Protect caches, service keys, and privileged accounts. Time synchronization supports freshness checks, but does not replace replay handling.

!!! info "External Resources"
    - [Kerberos V5 Message Exchanges](https://www.rfc-editor.org/rfc/rfc4120.html#section-3) (IETF)

## Active Directory Attack and Recovery Considerations

| Threat | Mechanism | Controls and limitations |
|---|---|---|
| Kerberoasting | Offline password guessing against service-ticket material encrypted under a service account key | Use managed service accounts where applicable, long random credentials, least privilege, and supported strong encryption types |
| AS-REP roasting | Offline guessing against suitable AS reply material when required pre-authentication is disabled | Require pre-authentication and review exceptional account settings |
| Golden Ticket | Forged TGTs after compromise of the domain's `krbtgt` key | Remove attacker persistence, investigate domain compromise, and execute a controlled key-reset recovery process |

**Why offline guessing is possible:** A legitimate requester can obtain a service ticket containing material encrypted under the service account's long-term key. An attacker can test password-derived candidate keys locally against that captured material without repeatedly attempting a service login. Practical resistance depends on the account's key/credential entropy and encryption characteristics, which is why long random managed credentials matter.

Active Directory retains the current and previous `krbtgt` keys. Microsoft's recovery procedure uses two resets with appropriate replication and ticket-lifetime planning. A single reset or a generic instruction to rotate regularly is not a complete compromise-recovery procedure. Follow the runbook and account for domain topology rather than running ad hoc resets.

**Example:** A user obtains a valid ticket for a file service but lacks read permission on a folder. The service should deny the read after authenticating the ticket; ticket issuance does not override folder authorization.

!!! info "External Resources"
    - [Reset the krbtgt Password During AD Recovery](https://learn.microsoft.com/en-us/windows-server/identity/ad-ds/manage/forest-recovery-guide/ad-forest-recovery-reset-the-krbtgt-password) (Microsoft)
    - [Kerberos Authorization](https://www.rfc-editor.org/rfc/rfc4120.html#section-1.4) (IETF)
