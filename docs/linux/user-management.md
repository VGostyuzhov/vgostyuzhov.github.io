# User and Group Management: A Study Guide

This guide focuses on the principles of user and group management in Linux, a cornerstone of system security. Proper configuration ensures that access is granted according to the principle of least privilege, providing accountability and preventing unauthorized actions.

## Basics and Core Concepts

Linux is an inherently multi-user operating system where every user is assigned a unique **User ID (UID)** and belongs to one or more groups, each with a **Group ID (GID)**. The system distinguishes between the all-powerful `root` user (UID 0), system accounts used for running services (typically low UIDs), and regular human user accounts (typically UIDs 1000 and above). This user and group information is primarily stored in the `/etc/passwd` and `/etc/group` files, respectively. Crucially, encrypted password hashes are stored separately in the `/etc/shadow` file, which is readable only by the `root` user, protecting them from offline cracking attempts.

Privilege escalation is managed almost exclusively through the **`sudo`** utility. Instead of sharing the `root` password, which is a major security risk, administrators grant specific users or groups the ability to run some or all commands as `root`. These permissions are defined in the `/etc/sudoers` file (which must always be edited with `visudo` to prevent syntax errors). This approach provides both granular control and a clear audit trail, as every command executed via `sudo` is logged.

A key security distinction is made between interactive user accounts and **service accounts**. User accounts are for humans and have a login shell. Service accounts are for running applications and daemons. To adhere to the principle of least privilege, service accounts should be created with no login shell (e.g., `/sbin/nologin`), have their password locked, and possess only the minimal permissions required for the service to function. This prevents them from being used as a foothold for an interactive session if compromised.

To defend against brute-force attacks, robust password policies are essential. This is managed through **Pluggable Authentication Modules (PAM)**. PAM allows for the configuration of password complexity rules (e.g., length, character types via `pam_pwquality`) and account lockout policies (e.g., locking an account after multiple failed login attempts via `pam_faillock`). Enforcing strong password hygiene and automated lockout mechanisms is a fundamental step in securing user access.

### User Management Cheat Sheet

| Task | Command / File | Description | Security Goal |
| :--- | :--- | :--- | :--- |
| **Create User** | `useradd` | Creates a new user account. | Provisioning Access |
| **Modify User** | `usermod` | Modifies user attributes (e.g., add to a group). | Adjusting Privileges |
| **Delete User** | `userdel -r` | Removes a user and their home directory. | Deprovisioning Access |
| **Manage Passwords** | `passwd`, `chage` | Set passwords and configure password aging policies. | Password Hygiene |
| **Grant Privileges** | `/etc/sudoers` (edit with `visudo`) | Defines which users/groups can run commands as root. | Least Privilege |
| **View Users** | `/etc/passwd` | Contains user account information (UID, GID, shell). | Auditing |
| **View Passwords** | `/etc/shadow` | Contains encrypted password hashes. Readable only by root. | Credential Protection |
| **Lockout Policy** | PAM (`pam_faillock`) | Configure account lockout after failed login attempts. | Brute-Force Mitigation |

!!! info "External Resources for Deep Dive"
    *   **`sudoers` Manual:** [https://www.sudo.ws/docs/man/sudoers.man/](https://www.sudo.ws/docs/man/sudoers.man/) (The official, detailed manual for the `sudoers` file syntax).
    *   **Pluggable Authentication Modules (PAM) Administrator's Guide:** [https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/8/html/security_hardening/using-pam_security-hardening](https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/8/html/security_hardening/using-pam_security-hardening) (A guide to understanding and configuring PAM).
    *   **Arch Wiki - Security/Account security:** [https://wiki.archlinux.org/title/Security#Account_security](https://wiki.archlinux.org/title/Security#Account_security) (A concise community-maintained checklist for user account security).