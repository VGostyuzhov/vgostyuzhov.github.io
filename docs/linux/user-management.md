# User and Group Management

Linux access is attached to numeric identities, process credentials, and policy, not only account names. NSS (Name Service Switch) resolves identities through configured local/directory sources; the same name can resolve differently across hosts or namespaces.

## Accounts and Running Credentials

`/etc/passwd` stores local account attributes, not local password hashes. `/etc/shadow` holds password-verifier/account-aging data with restricted access (often root and a designated group, depending on the distribution). It does not store reversibly encrypted passwords. UID ranges and account allocation conventions are distribution/configuration dependent.

```bash
# Read-only identity lookup; getent follows configured NSS, not only local files.
getent passwd appsvc
id appsvc
# Inspect an authorized process's actual credentials, which may be older.
target_pid=1234
sed -n '/^Uid:/p; /^Gid:/p; /^Groups:/p' /proc/$target_pid/status
```

Changing group membership in the directory/local database does not rewrite existing processes' supplementary group lists. New login/service sessions may receive new credentials while old workers continue using their previous ones. Offboarding needs an explicit boundary for new login, current sessions, running jobs, SSH keys, API credentials, and external workload roles.

!!! info "External Resources"
    - [Process Credentials](https://man7.org/linux/man-pages/man7/credentials.7.html) (Linux man-pages)
    - [Shadow Password Data](https://man7.org/linux/man-pages/man5/shadow.5.html) (shadow-utils)

## Service Accounts and Delegation

A dedicated service identity separates one daemon's data/permissions from another's. Locking its password and selecting `nologin` constrain particular login paths; they do not prevent the daemon from executing code after exploitation, nor necessarily block every SSH/PAM/key-based path. A service process can operate without an interactive login shell.

**sudo** delegates specified command execution under another identity. Its safety depends on the executable, arguments, environment, writable configuration/plugins, shell escapes, and files it can modify. Allowing a general interpreter/editor as root is often much broader than the command name suggests. Edit policy with `visudo` and validate the effective result; sudo logs do not automatically record every action inside a launched root shell.

```bash
# Authorized Linux host: inspect rather than modify delegation.
sudo -l
# Validate the installed sudoers configuration without changing it.
sudo visudo -c
# Review indirect inputs too: a root-run helper reading a user-writable script
# can delegate root authority even if its own executable is not writable.
```

Use narrow, maintained administrative interfaces with controlled inputs and auditable actions; do not invent an unsafe wrapper as a substitute for reviewing the operation. Removing sudo permission does not kill a root shell already started through it.

!!! info "External Resources"
    - [sudoers Command, Environment, and Logging Rules](https://www.sudo.ws/docs/man/sudoers.man/) (sudo)
    - [nologin Scope](https://man7.org/linux/man-pages/man8/nologin.8.html) (util-linux)

## Provisioning and Removal

Use platform-managed account provisioning, unique identities, controlled group assignment, and recorded ownership. Before account deletion or UID reuse, identify retained files, services, jobs, and credentials; a new account receiving an old numeric UID can acquire access to old UID-owned data. `userdel -r` is a destructive removal operation, not an investigation command.

Choose authentication throttling, recovery, and MFA for the actual service. Account lockout can also enable denial of service; do not impose one universal failure count. See [Linux authentication](authentication.md) for PAM and SSH enforcement.
