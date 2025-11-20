# Authentication and Directory Services

## Local Authentication

### Basics and Core Concepts

**Pluggable Authentication Modules (PAM)** provide a dynamic authorization mechanism for applications and services in Linux environments. PAM decouples the task of authentication from the application, allowing administrators to configure authentication policies without modifying application source code. Configurations are typically stored in `/etc/pam.d/` and define a stack of modules managed by four management groups: `auth` (identity verification), `account` (authorization/validity), `password` (credential updates), and `session` (environment setup/teardown). Control flags (e.g., `required`, `requisite`, `sufficient`) determine how the failure or success of a module affects the overall result.

**Password Storage and Hashing** is a critical defense against offline attacks and credential dumping. Passwords must never be stored in cleartext; instead, they should be hashed using slow, iterative algorithms designed to resist GPU-accelerated brute-force and rainbow table attacks. Modern standards mandate the use of algorithms such as Argon2id, bcrypt, or scrypt, combined with a unique, random salt per user. Deprecated algorithms like MD5, SHA-1, or standard SHA-256 (without aggressive iteration counts) are considered insufficient for password storage due to their speed, which benefits attackers.

**Multi-Factor Authentication (MFA)** augments security by requiring at least two of the three authentication factors: something you know (password), something you have (hardware token, smartphone), or something you are (biometrics). Implementation methods vary from Time-based One-Time Passwords (TOTP) and HMAC-based One-Time Passwords (HOTP) to stronger cryptographic proofs like FIDO2/WebAuthn. While SMS and email-based MFA provide a layer of defense, they are susceptible to SIM swapping and interception; hardware keys and FIDO2 are preferred for high-security environments due to their resistance to phishing and man-in-the-middle (MitM) attacks.

**Key-based Authentication**, predominantly used in SSH (Secure Shell), relies on asymmetric cryptography (public-private key pairs) rather than shared secrets like passwords. The public key is placed in the target system's `~/.ssh/authorized_keys` file, while the private key remains secured on the client. This method mitigates brute-force attacks against passwords and enables automated, non-interactive logins. Security best practices for key-based auth include disabling root login, strictly limiting file permissions on key files, protecting private keys with passphrases, and disabling password authentication entirely in the `sshd_config`.

## Directory Integration

### Basics and Core Concepts

**LDAP (Lightweight Directory Access Protocol)** is the industry standard for accessing and maintaining distributed directory information services over an IP network. It organizes data in a hierarchical tree structure (Directory Information Tree) using Distinguished Names (DNs) to uniquely identify entries. Standard LDAP traffic (port 389) transmits data in cleartext; therefore, security engineering requires the enforcement of LDAPS (port 636) or StartTLS to encrypt the channel. Common security concerns include LDAP injection attacks and anonymous binding, which should be disabled to prevent unauthorized directory enumeration.



**Kerberos** is a network authentication protocol designed to provide strong authentication for client/server applications by using secret-key cryptography. It relies on a trusted third party, the Key Distribution Center (KDC), which consists of an Authentication Server (AS) and a Ticket Granting Server (TGS). The protocol uses "tickets" to allow nodes communicating over a non-secure network to prove their identity to one another in a secure manner. Crucially, Kerberos requires strict time synchronization (NTP) between clients and servers to prevent replay attacks, and it supports mutual authentication, ensuring both the user and the server verify each other's identity.



**Active Directory (AD)** is Microsoft's proprietary directory service that utilizes LDAP, Kerberos, and DNS. It organizes network resources into domains, trees, and forests, serving as a central authority for network security and identity management. AD security relies heavily on minimizing the attack surface of Domain Controllers, managing Group Policy Objects (GPO) to enforce security baselines, and monitoring for "Pass-the-Hash" or "Golden Ticket" attacks. Legacy protocol support (such as NTLMv1/v2) is a significant vulnerability in AD environments and should be deprecated in favor of Kerberos enforcement.

**Identity Federation** enables the portability of identity information across distinct security domains, allowing users to use one set of credentials to access data across multiple organizations or applications (Single Sign-On). Common standards include SAML (Security Assertion Markup Language), OIDC (OpenID Connect), and OAuth 2.0. In this model, an Identity Provider (IdP) authenticates the user and issues a token (assertion), which is then consumed and trusted by the Service Provider (SP). Security in federation relies on the trust relationship (PKI/Certificate exchange) between the IdP and SP and the secure handling of assertion tokens to prevent hijacking.



[Image of SAML authentication flow]


### Summary Cheatsheet

| Concept | Protocol/Port | Key Component | Primary Security Risk |
| :--- | :--- | :--- | :--- |
| **LDAP** | TCP 389 / 636 (SSL) | Directory Information Tree (DIT) | Cleartext transmission (Sniffing), Injection |
| **Kerberos** | TCP/UDP 88 | KDC (AS + TGS) | Time synchronization drift, Golden Tickets |
| **SSH Keys** | TCP 22 | Pub/Priv Key Pair | Private key theft, weak file permissions |
| **SAML** | HTTPS 443 | XML Assertions | XML External Entity (XXE), Token replay |
| **OIDC** | HTTPS 443 | JSON Web Tokens (JWT) | Token leakage, weak signature algorithms |
| **PAM** | N/A (Local Lib) | `libpam`, Stack Config | Misconfiguration (e.g., `sufficient` flag misuse) |
| **Hashing** | N/A (Compute) | Salt + Iterations | Insufficient work factor (fast hashes like MD5) |

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