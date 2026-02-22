# Authentication and Directory Services

## Local Authentication

### Basics and Core Concepts

**Pluggable Authentication Modules (PAM)** provide a dynamic authorization mechanism for applications and services in Linux environments. PAM decouples the task of authentication from the application, allowing administrators to configure authentication policies without modifying application source code. Configurations are typically stored in `/etc/pam.d/` and define a stack of modules managed by four management groups: `auth` (identity verification), `account` (authorization/validity), `password` (credential updates), and `session` (environment setup/teardown). Control flags (e.g., `required`, `requisite`, `sufficient`) determine how the failure or success of a module affects the overall result.

**Password Storage and Hashing** is a critical defense against offline attacks and credential dumping. Passwords must never be stored in cleartext; instead, they should be hashed using slow, iterative algorithms designed to resist GPU-accelerated brute-force and rainbow table attacks. Modern standards mandate the use of algorithms such as Argon2id, bcrypt, or scrypt, combined with a unique, random salt per user. Deprecated algorithms like MD5, SHA-1, or standard SHA-256 (without aggressive iteration counts) are considered insufficient for password storage due to their speed, which benefits attackers.

**Multi-Factor Authentication (MFA)** augments security by requiring at least two of the three authentication factors: something you know (password), something you have (hardware token, smartphone), or something you are (biometrics). Implementation methods vary from Time-based One-Time Passwords (TOTP) and HMAC-based One-Time Passwords (HOTP) to stronger cryptographic proofs like FIDO2/WebAuthn. While SMS and email-based MFA provide a layer of defense, they are susceptible to SIM swapping and interception; hardware keys and FIDO2 are preferred for high-security environments due to their resistance to phishing and man-in-the-middle (MitM) attacks.

**Key-based Authentication**, predominantly used in SSH (Secure Shell), relies on asymmetric cryptography (public-private key pairs) rather than shared secrets like passwords. The public key is placed in the target system's `~/.ssh/authorized_keys` file, while the private key remains secured on the client. This method mitigates brute-force attacks against passwords and enables automated, non-interactive logins. Security best practices for key-based auth include disabling root login, strictly limiting file permissions on key files, protecting private keys with passphrases, and disabling password authentication entirely in the `sshd_config`.

## Directory Integration

### Basics and Core Concepts

**LDAP (Lightweight Directory Access Protocol)** is the industry standard for accessing and maintaining distributed directory information services. For a detailed breakdown of LDAP security and threats, see **[Directory Services](../authentication/directory-services.md)**.

**Kerberos** is a network authentication protocol using "tickets" to prove identity. It requires strict time synchronization (NTP) and supports mutual authentication. For a deep dive into Kerberos flows and attacks (like Kerberoasting), see **[Directory Services](../authentication/directory-services.md)**.

**Active Directory (AD)** is Microsoft's directory service utilizing LDAP, Kerberos, and DNS. AD security relies on managing Group Policy Objects (GPO) and monitoring for "Pass-the-Hash" or "Golden Ticket" attacks.

**Identity Federation** enables Single Sign-On (SSO) across distinct security domains using standards like **[SAML](../authentication/saml.md)**, **[OIDC](../authentication/oauth2-oidc.md)**, and OAuth 2.0. Security relies on the trust relationship (PKI/Certificate exchange) between the Identity Provider (IdP) and Service Provider (SP).

### Summary Cheatsheet

| Concept | Protocol/Port | Key Component | Primary Security Risk |
| :--- | :--- | :--- | :--- |
| **[LDAP](../authentication/directory-services.md)** | TCP 389 / 636 | Directory Tree (DIT) | Cleartext transmission, Injection |
| **[Kerberos](../authentication/directory-services.md)** | TCP/UDP 88 | KDC (AS + TGS) | Time drift, Golden Tickets |
| **SSH Keys** | TCP 22 | Pub/Priv Key Pair | Private key theft, weak permissions |
| **[SAML](../authentication/saml.md)** | HTTPS 443 | XML Assertions | XML Signature Wrapping, XXE |
| **[OIDC](../authentication/oauth2-oidc.md)** | HTTPS 443 | JSON Tokens (JWT) | Token leakage, weak signing |
| **PAM** | N/A (Local Lib) | `libpam`, Stack Config | Flag misuse (e.g., `sufficient`) |
| **Hashing** | N/A (Compute) | Salt + Iterations | Fast hashes (MD5, SHA-1) |

!!! info "External Resources for Deep Dive"

    * **NIST Special Publication 800-63B** - *Digital Identity Guidelines: Authentication and Lifecycle Management.* The authoritative standard for MFA, password requirements, and identity proofing.
        [https://pages.nist.gov/800-63-3/sp800-63b.html](https://pages.nist.gov/800-63-3/sp800-63b.html)

    * **Red Hat Enterprise Linux Security Guide: Using PAM** - *Official Documentation.* detailed breakdown of PAM configuration syntax, module types, and control flags.
        [https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/8/html/configuring_and_managing_authentication_services/pluggable-authentication-modules-pam_configuring-and-managing-authentication-services](https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/8/html/configuring_and_managing_authentication_services/pluggable-authentication-modules-pam_configuring-and-managing-authentication-services)

    * **OWASP Password Storage Cheat Sheet** - *Practical Guide.* Best practices for choosing hashing algorithms, salting, and work factors.
        [https://cheatsheetseries.owasp.org/cheatsheets/Password_Storage_Cheat_Sheet.html](https://cheatsheetseries.owasp.org/cheatsheets/Password_Storage_Cheat_Sheet.html)

    * **Explain Like I'm 5: Kerberos** - *Conceptual Deep Dive.* A highly regarded technical explanation of the Kerberos handshake and ticket exchange process.
        [https://www.roguelynn.com/words/explain-like-im-5-kerberos/](https://www.roguelynn.com/words/explain-like-im-5-kerberos/)

    * **Auth0: SAML vs. OIDC** - *Comparison Guide.* A technical comparison of federation protocols, helpful for understanding when to use XML-based vs. JSON-based assertions.
        [https://auth0.com/intro-to-iam/saml-vs-openid-connect-oidc/](https://auth0.com/intro-to-iam/saml-vs-openid-connect-oidc/)