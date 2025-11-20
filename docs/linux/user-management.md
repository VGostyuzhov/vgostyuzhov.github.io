# Linux User and Group Management

Proper user and group management is fundamental to Linux security. It ensures that users and processes have only the permissions they need to perform their tasks, adhering to the principle of least privilege. This section covers the core concepts and practices for managing user accounts, privileges, and authentication.

## User and Group Fundamentals

Linux is a multi-user system. Every user has a unique **User ID (UID)** and is a member of at least one **Group ID (GID)**.

- **UID `0`**: The `root` user, which has unlimited privileges.
- **UIDs `1-999`**: Typically reserved for system accounts and services.
- **UIDs `1000+`**: Assigned to regular user accounts.

### Key Files for User Management

- **/etc/passwd**: Stores user account information, including username, UID, GID, home directory, and default shell. It is world-readable but should not contain password hashes.
- **/etc/shadow**: Contains the encrypted password hashes for user accounts, along with password aging policies. It is only readable by the `root` user.
- **/etc/group**: Defines groups and their members.
- **/etc/gshadow**: Contains group password information (rarely used).
- **/etc/sudoers**: Manages which users and groups have `sudo` (superuser) privileges. This file should always be edited with the `visudo` command to prevent syntax errors.

## User Account Management

### Creating, Modifying, and Deleting Users

- **`useradd`**: Creates a new user account.
  ```bash
  # Add a new user with a home directory and default shell
  useradd -m -s /bin/bash newuser
  ```
- **`usermod`**: Modifies an existing user account (e.g., changing their shell, groups, or UID).
  ```bash
  # Add the user 'newuser' to the 'sudo' group
  usermod -aG sudo newuser
  ```
- **`userdel`**: Deletes a user account.
  ```bash
  # Remove the user and their home directory
  userdel -r newuser
  ```

### Password Policies and Aging

Enforcing strong password policies is critical for preventing unauthorized access. This includes setting:

- **Password Complexity**: Enforced using Pluggable Authentication Modules (PAM) via the `pam_pwquality` module. Configuration is in `/etc/security/pwquality.conf`.
- **Password Aging**: Forces users to change their passwords periodically. The `chage` command is used to manage these settings.
  ```bash
  # Set the maximum number of days a password is valid to 90
  chage -M 90 newuser

  # Expire a user's password immediately, forcing them to change it on next login
  chage -d 0 newuser
  ```

### Account Lockout Mechanisms

To mitigate brute-force attacks, accounts should be locked after a certain number of failed login attempts. This is configured using PAM via the `pam_tally2` or `pam_faillock` module.

Example configuration in `/etc/pam.d/system-auth`:
```
auth        required      pam_faillock.so preauth silent audit deny=5 unlock_time=900
auth        [default=die] pam_faillock.so authfail audit deny=5 unlock_time=900
account     required      pam_faillock.so
```
This locks an account for 900 seconds (15 minutes) after 5 failed login attempts.

## Service Accounts vs. User Accounts

- **User Accounts**: Used by interactive human users to log in and perform tasks. They typically have a home directory and a login shell.
- **Service Accounts**: Used by applications and services to run in a restricted environment. They should not be used for interactive logins.
  - **Best Practices for Service Accounts**:
    - Assign a non-interactive shell (e.g., `/sbin/nologin` or `/bin/false`).
    - Create them with a specific purpose and limited privileges.
    - Do not reuse service accounts across different services.
    - Lock the account's password to prevent logins (`passwd -l service_account`).

## Privilege Management with `sudo`

`sudo` allows permitted users to execute commands as another user, typically `root`. It provides a more secure and auditable alternative to sharing the `root` password.

### `sudo` Configuration

The `/etc/sudoers` file defines who can run what commands. The `visudo` command should always be used to edit it.

**Example `sudoers` rules:**

```
# Allow members of the 'wheel' group to run any command
%wheel ALL=(ALL) ALL

# Allow the 'devops' group to restart the webserver
%devops ALL=(ALL) /usr/sbin/service httpd restart
```

### Auditing `sudo` Usage

All `sudo` commands are logged by default to `/var/log/secure` (RHEL/CentOS) or `/var/log/auth.log` (Debian/Ubuntu), providing a clear audit trail of privileged operations.

By implementing these user and group management practices, you can significantly enhance the security posture of any Linux system.