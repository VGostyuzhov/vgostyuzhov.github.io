# Authentication and Directory Services

Linux login depends on the application and configured identity/authentication sources. Finding a user, verifying a credential, deciding account eligibility, opening a session, and authorizing a particular operation are separate steps.

## PAM and Local Authentication

**Pluggable Authentication Modules (PAM)** let participating applications invoke configurable authentication/account/session functions. It is not a universal packet or syscall authorization layer. A service that does not use PAM does not inherit its stack merely because the files exist.

| PAM group | Decision | Example distinction |
|---|---|---|
| `auth` | Verify authentication evidence | Password or configured second factor |
| `account` | Account eligibility | Expiry/access conditions, not every file's authorization |
| `password` | Credential changes | Hashing and password-update checks |
| `session` | Session setup/cleanup | Limits, audit context, environment under configured modules |

A `required` module failure is remembered while processing continues; `requisite` failure returns early; `sufficient` success can end that stack successfully if earlier required failures do not prevent it. Includes and extended controls change the full path. A multi-factor authentication (MFA) module after an earlier short-circuit success may never run. Validate the actual service's complete stack with permitted success/failure/recovery cases before changing a login path.

```text
# Conceptual flow, not a PAM configuration to paste:
auth module A: sufficient + success -> may return success before later auth module B
later MFA module B                 -> cannot protect a path that never invokes it
# Required account/session processing still depends on application calls/configuration.
```

Linux password-verifier algorithms depend on the distribution's PAM/libcrypt support and configured parameters; yescrypt and SHA-512-crypt are examples, not equivalent to a single fast SHA-512 digest. Do not prescribe Argon2/bcrypt as a universal `pam_unix` setting based on web-application password advice. A salt prevents shared precomputation; cost parameters increase guessing cost. Protect shadow data and recovery even with a suitable verifier.

!!! info "External Resources"
    - [PAM Stack and Control Syntax](https://man7.org/linux/man-pages/man5/pam.d.5.html) (Linux-PAM)
    - [pam_unix Algorithms and Account Checks](https://man7.org/linux/man-pages/man8/pam_unix.8.html) (Linux-PAM)

## SSH User Authentication and Host Identity

An SSH client verifies the server's **host key**; the server separately verifies the user's key/certificate or other configured evidence. Public-key login proves control of a private-key capability but does not automatically require two factors. Hardware-backed keys with appropriate verification can supply stronger properties; mere use of a public key does not imply phishing resistance for every session/recovery path.

```bash
# Read-only validation on an authorized host; sshd may need privileges/config access.
sshd -t  # Syntax check: does not prove Match blocks enforce the intended user policy.
sshd -T -C user=alice,host=client.example,addr=192.0.2.8
# Inspect authenticationmethods, pubkeyauthentication, passwordauthentication,
# kbdinteractiveauthentication, usepam, permitrootlogin, and applicable Match effects.
```

Disabling `PasswordAuthentication` alone does not establish that keyboard-interactive/PAM cannot accept a password. To require a key plus a second method, configure supported `AuthenticationMethods` and the matching backend, then test both missing-factor cases and a successful full flow. Keep a verified recovery route while rolling out changes. Restrict forwarded-agent use, key scope, and authorized-key options where appropriate.

Locking a Unix password does not necessarily revoke SSH keys, existing sessions, sudo shells, directory tickets, or cloud credentials. See [account lifecycle](user-management.md#accounts-and-running-credentials) and [session lifecycle](../authentication/sessions.md).

!!! info "External Resources"
    - [sshd_config Authentication Methods and Match Rules](https://man.openbsd.org/sshd_config) (OpenSSH)
    - [SSH Host-Key Checking](https://man.openbsd.org/ssh_config#StrictHostKeyChecking) (OpenSSH)

## Directory Integration

NSS resolves identities; authentication can involve PAM/SSSD, LDAP, or Kerberos under the deployment's configuration. Cached directory information and offline authentication can affect how quickly removal takes effect. Identify the application's lookup/cache/session behavior instead of promising immediate offboarding from a directory update alone.

Use [Directory Services](../authentication/directory-services.md) for LDAP binds and Kerberos tickets, and [OIDC](../authentication/oauth2-oidc.md) or [SAML](../authentication/saml.md) for application federation. Federation trust requires recipient/issuer/request validation and lifecycle policy, not merely exchanging certificates.
