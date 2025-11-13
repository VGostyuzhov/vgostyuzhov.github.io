# *nix Systems Security

Unix-like systems (Linux, Unix, BSD) form the backbone of modern infrastructure. Understanding their security model is essential for infrastructure and cloud security roles.

## System Architecture & Security Model

### Kernel vs Userspace
- Kernel space privileges and syscalls
- User space isolation and boundaries
- Process rings and protection levels
- System call interface security

### File System Security

#### File Permissions and Ownership

File permissions in UNIX-based systems are a form of Discretionary Access Control (DAC). Every file and directory has an owner, an associated group, and permissions are defined for three distinct classes: the User (owner), the Group, and Other (all other users).

**Core Permission Types:**

- **Read `(r)`**: View file contents or list directory contents
- **Write `(w)`**: Modify/delete files or create/delete files within directory
- **Execute `(x)`**: Run file as program or traverse into directory

**Permission Management:**

- `chmod` symbolic mode: `chmod u+x file` (adds execute for user)
- `chmod` octal mode: `chmod 755 file` (rwx for user, r-x for group/other)
- `chown` and `chgrp` control ownership
- `umask` sets default permissions for new files

!!! info "External Resources"
    - [Linux File Ownership and Permissions Guide](https://www.freecodecamp.org/news/file-ownership-permissions-rhel/) (freeCodeCamp)
    - [Understanding Chown and Chmod](https://www.pythian.com/blog/technical-track/an-overview-of-understanding-chown-and-chmod-in-linux) (Pythian)
    - [UNIX File Permissions](https://help.rc.unc.edu/how-to-use-unix-and-linux-file-permissions/) (UNC Research Computing)

#### Special File Types and Attributes

**Special Permission Bits:**

- **SUID**: Process runs with file owner's privileges (denoted by `s`)
- **SGID**: Process runs with file group's privileges or inherits directory group
- **Sticky Bit**: Only file owner can delete files in directory (e.g., `/tmp`)

**Special File Types:**

- **Symbolic Link `l`**: Pointer to another file's path
- **Block Device `(b)`**: Device that moves data in blocks (e.g., `/dev/sda`)
- **Character Device `(c)`**: Streams data character by character (e.g., `/dev/tty`)
- **Socket `(s)`**: Inter-process communication file
- **Named Pipe/FIFO `(p)`**: One-way IPC between processes

**Extended Attributes:**
- **Immutable `(i)`**: `chattr +i` makes file unchangeable, even by root
- **Append-only `(a)`**: `chattr +a` allows only data appending

!!! info "External Resources"
    - [Setuid, Setgid, and Sticky Bit](https://www.cbtnuggets.com/blog/technology/system-admin/linux-file-permissions-understanding-setuid-setgid-and-the-sticky-bit) (CBT Nuggets)
    - [chattr and lsattr commands](https://www.geeksforgeeks.org/linux-unix/chattr-command-in-linux-with-examples/) (GeeksforGeeks)
    - [Special Files in Linux](https://dev.to/naval_upadhyay/special-files-in-linux-the-hidden-power-behind-everything-is-a-file-34j7) (DEV Community)

#### Mount Points and Filesystem Options

Mount points attach additional filesystems to the directory hierarchy. Configuration is stored in `/etc/fstab` with critical security mount options:

**Security Mount Options:**

- **nosuid**: Prevents SUID/SGID bits from taking effect
- **nodev**: Prevents device file interpretation
- **noexec**: Prevents binary execution
- **ro**: Read-only mounting

Common secure mounting strategies:

- `/tmp` and `/var/tmp`: `nosuid,nodev,noexec`
- `/home`: `nosuid,nodev`
- `/boot`: `ro`

!!! info "External Resources"
    - [fstab Documentation](https://wiki.archlinux.org/title/Fstab) (ArchWiki)
    - [Mount Options Explanation](https://serverfault.com/) (Server Fault)
    - [Mount Points in Linux](https://technogeeks.org/) (Technogeeks)

#### File System Hardening Techniques

Filesystem hardening enforces least privilege and reduces attack surface through:

**Partitioning Strategy:**

- Separate partitions for `/home`, `/tmp`, `/var`, `/var/log`
- Apply restrictive mount options per partition
- Isolate from root (`/`) filesystem

**Hardening Policy:**

- **Secure Mounts**: Apply security options to appropriate partitions
- **Strict Permissions**: Restrictive umask (e.g., 027), audit critical files
- **Attribute Use**: Immutable bit on static config files
- **Integrity Monitoring**: FIM tools like AIDE or Tripwire

**Compliance Framework:**

- Follow CIS Benchmarks for comprehensive hardening
- Regular security audits and file integrity checks

!!! info "External Resources"
    - [CIS Benchmarks FAQ](https://www.cisecurity.org/) (Center for Internet Security)
    - [Linux Server Hardening Guide](https://www.msp360.com/) (MSP360)
    - [AIDE Tutorial](https://docs.oracle.com/) (Oracle Linux Docs)

## Access Control Systems

---

## Discretionary Access Control (DAC)

### Basics and Core Concepts

Discretionary Access Control (DAC) is an access control model where the **owner** of an object (e.g., a file or directory) has the discretion to determine the access permissions for other subjects (e.g., users or groups). This model is identity-centric, meaning access rights are bound to the identity of the subject. The creator of an object is its initial owner, and this owner can grant or revoke privileges to others.

Traditional Unix permissions are a classic implementation of DAC. Every file and directory has three sets of permissions defined for three classes of users: the **owner** (u), the **group** (g), and **others** (o). Each set defines three permissions: **read (r)**, **write (w)**, and **execute (x)**. These are commonly represented by a 9-character string (e.g., `rwxr-xr--`) or an octal number (e.g., `754`). The `chmod` command modifies these permissions, while `chown` and `chgrp` change the user and group ownership.

Special permission bits modify the standard behavior.

* The **SUID (Set User ID)** bit (`s` in the owner's execute field) on an executable file causes it to run with the privileges of the file's owner, not the user who executed it (e.g., the `passwd` command running as root). 
* The **SGID (Set Group ID)** bit (`s` in the group's execute field) on an executable makes it run with the file's group privileges. On a directory, SGID causes new files and subdirectories created within it to inherit the directory's group ownership, which is useful for shared project folders. 
* The **Sticky Bit** (`t` in the others' execute field), when set on a directory (like `/tmp`), allows users to delete or rename files *only* if they are the owner of the file, the owner of the directory, or root, even if they have write permission to the directory.

**Access Control Lists (ACLs)** extend the traditional Unix permission model. While standard permissions only define access for one owner, one group, and everyone else, ACLs provide a more granular mechanism. They allow the owner to grant or deny permissions for specific, additional users and groups. The `getfacl` command is used to view ACLs, and `setfacl` is used to modify them. A file with an ACL applied is often indicated by a `+` (plus sign) at the end of its permission string in the `ls -l` output.

| Component | Symbolic | Octal | Description |
| :--- | :--- | :--- | :--- |
| **Permissions** | | | |
| Read | `r` | `4` | View contents of a file; list contents of a directory. |
| Write | `w` | `2` | Modify or delete a file; create or delete files in a directory. |
| Execute | `x` | `1` | Run a file as a script/program; `cd` into a directory. |
| No Permission | `-` | `0` | No access. |
| **Special Bits** | | | |
| SUID | `---s--` | `4000` | (Set User ID) Executable runs with file owner's privileges. |
| SGID | `------s-` | `2000` | (Set Group ID) Executable runs with file group's privileges. / Files in dir inherit dir's group. |
| Sticky Bit | `--------t` | `1000` | On directories, only the owner (or root) can delete/rename files. |
| **Core Commands** | | | |
| `chmod` | `chmod [u/g/o/a][+/-][r/w/x]` | `chmod [octal]` | Change permissions. (e.g., `chmod 755 file`, `chmod g+w file`) |
| `chown` | `chown [user]:[group]` | | Change owner and/or group. (e.g., `chown user:group file`) |
| `chgrp` | `chgrp [group]` | | Change group ownership. |
| **ACL Commands** | | | |
| `getfacl` | `getfacl [file]` | | View the Access Control List for a file. |
| `setfacl` | `setfacl -m u:[user]:[rwx]` | | Modify (set) the ACL for a file. (e.Same syntax for `g:[group]`) |

!!! info "External Resources"
    - **Official Documentation/Guides:**
        * [Access Control Lists in Linux](https://documentation.suse.com/sles/12-SP5/html/SLES-all/cha-security-acls.html) (SUSE Documentation)
        * [What You Need To Know To Set Linux Permissions and Access Control Lists](https://www.comptia.org/en-us/blog/what-you-need-to-know-to-set-linux-permissions-and-access-control-lists) (CompTIA)
    - **In-Depth Blog Posts:**
        * [Linux File Permissions: Understanding Setuid, Setgid, and the Sticky Bit](https://www.cbtnuggets.com/blog/technology/system-admin/linux-file-permissions-understanding-setuid-setgid-and-the-sticky-bit) (CBT Nuggets)
    - **Video Tutorials:**
        * [YouTube - Linux ACLs Explained](https://www.youtube.com/watch?v=LnKoncbQBsM)

---

# Mandatory Access Control (MAC)

## Basics and Core Concepts

Mandatory Access Control (MAC) is an access control model where the operating system constrains the ability of a subject (e.g., a process) to access or perform operations on an object (e.g., a file). Unlike Discretionary Access Control (DAC), where owners can set permissions, MAC policies are centrally managed by an administrator and enforced by the kernel. Access decisions are made by a **Reference Monitor** (like the Linux Security Modules - LSM) by comparing security labels of the subject and object against a system-wide policy.

SELinux (Security-Enhanced Linux) is a label-based MAC implementation. Every subject and object on the system is assigned a **security context** (e.g., `user:role:type:level`). The core of SELinux policy is **Type Enforcement (TE)**, which defines allowed interactions between types (e.g., `allow httpd_t var_www_t:file { read }`). Its policy is monolithic, loaded into the kernel, and can operate in `enforcing`, `permissive`, or `disabled` modes. AppArmor, by contrast, is a path-based MAC system. It uses **profiles** (plain text files) that are bound to specific executable paths (e.g., `/usr/sbin/sshd`) and define what file system paths and capabilities that application can access.

The SELinux context often includes components for **Role-Based Access Control (RBAC)** and **Multi-Level/Category Security (MLS/MCS)**. RBAC limits user privileges based on their assigned role (e.g., `web_admin_r`). MLS provides a hierarchical security model (e.g., `Unclassified` < `Secret`) based on the Bell-LaPadula model, enforcing "no read up" and "no write down" rules. MCS is a non-hierarchical extension that uses categories (e.g., `s0:c1,c2`) to provide compartmentalization, which is widely used to isolate containers.

Policy development and debugging are critical skills. For SELinux, policy is written in TE or CIL (Common Intermediate Language). Debugging relies on parsing `type=AVC` (Access Vector Cache) denial messages from `/var/log/audit/audit.log` using `ausearch`. The `audit2allow` utility can read these denials and suggest new `allow` rules. For AppArmor, profiles are developed manually or with tools like `aa-genprof`. Debugging is done in `complain` mode, with denials logged to `kern.log` or `auditd`, and the `aa-logprof` tool is used to interactively review and approve or deny behaviors.

## Cheatsheet: MAC Concepts & Implementations

| Concept | SELinux (Label-Based) | AppArmor (Path-Based) |
| :--- | :--- | :--- |
| **Core Unit** | Security Context (`user:role:type:level`) | Profile (per-executable path) |
| **Policy Logic** | Type Enforcement (TE). Rules on *types*. | Path-based rules. `allow /var/www/ r`. |
| **Enforcement** | Kernel (LSM). Enforces on all objects. | Kernel (LSM). Enforces on profiled apps. |
| **Modes** | `enforcing`, `permissive`, `disabled` | `enforce`, `complain`, `disabled` |
| **Denial Log** | `/var/log/audit/audit.log` (`type=AVC`) | `/var/log/kern.log` or `auditd` |
| **Debug Tool** | `ausearch`, `audit2allow`, `sealert` | `aa-logprof`, `aa-complain` |
| **Models** | Supports TE, RBAC, MLS, MCS | Focuses on application confinement |

## External Resources for Deep Dive

!!! info "External Resources for Deep Dive"
    * **Official Documentation & Wikis:**
        * [SELinux Project Wiki](https://selinuxproject.org/page/Main_Page): The central hub for SELinux documentation, architecture explanations, and news.
        * [AppArmor Wiki](https://gitlab.com/apparmor/apparmor/-/wikis/home): The official GitLab wiki for AppArmor, containing documentation on profile creation, kernel interfaces, and utilities.
        * [Red Hat Enterprise Linux (RHEL) SELinux Guide](https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/8/html/using_selinux): A comprehensive and practical guide covering SELinux policy, modes, and troubleshooting.

    * **In-Depth Articles & Tutorials:**
        * [Red Hat Blog: MCS and Containers](https://www.redhat.com/en/blog/multi-category-security-mcs-selinux-and-containers): A detailed technical blog post explaining the practical application of MCS with SELinux for container isolation.
        * [Ubuntu Tutorial: AppArmor Profile Development](https://ubuntu.com/tutorials/apparmor-profile-development): A hands-on tutorial for creating and debugging AppArmor profiles from scratch.

## Critical System Directories

### Essential System Paths

- `/proc` - Process and kernel information
- `/sys` - Kernel and hardware interface
- `/dev` - Device files and special files
- `/tmp` - Temporary files security concerns

### Security-Critical Files

- `/etc/passwd` and `/etc/shadow`
- `/etc/sudoers` and sudo configuration
- System service configurations
- Log file locations and permissions

## User and Process Management

### User Account Security

- User creation and management
- Password policies and aging
- Account lockout mechanisms
- Service accounts vs user accounts

### Process Security

- Process isolation and sandboxing
- Resource limits and quotas
- Process monitoring and auditing
- Signal handling and IPC security

## Network Security

### Network Stack Security

- Firewall configuration (iptables/netfilter)
- Network namespaces and isolation
- Port binding and service exposure
- Network monitoring and logging

### Service Hardening
- Service configuration best practices
- Daemon security and privileges
- Network service isolation
- SSL/TLS configuration

## Authentication and Directory Services

### Local Authentication
- PAM (Pluggable Authentication Modules)
- Password storage and hashing
- Multi-factor authentication
- Key-based authentication

### Directory Integration
- LDAP integration and security
- Kerberos authentication
- Active Directory integration
- Identity federation

## Logging and Monitoring

### System Logging
- Syslog architecture and configuration
- Log rotation and retention
- Centralized logging solutions
- Log analysis and SIEM integration

### Security Monitoring
- File integrity monitoring
- Process and network monitoring
- Intrusion detection systems
- Performance monitoring for security

## Container and Virtualization Security

### Container Security
- Docker security best practices
- Container runtime security
- Image scanning and validation
- Container isolation mechanisms

### Virtualization Security
- Hypervisor security considerations
- VM escape techniques and prevention
- Resource isolation and allocation
- Virtual network security

## Hardening and Compliance

### System Hardening
- Security benchmarks (CIS, STIG)
- Service minimization
- Kernel hardening options
- Security modules and frameworks

### Compliance Requirements
- Audit trail requirements
- Access logging and monitoring
- Configuration management
- Vulnerability management

## Common Attack Vectors

### Privilege Escalation
- Local privilege escalation techniques
- Kernel exploits and mitigations
- SUID/SGID abuse
- Container escape techniques

### Persistence Mechanisms
- Init system abuse
- Cron job manipulation
- Library injection
- Kernel module insertion

## Security Tools and Utilities

### Built-in Security Tools
- `sudo` and privilege management
- `chroot` and process isolation
- `iptables` and network filtering
- Audit frameworks (`auditd`)

### Third-party Security Tools
- Configuration management tools
- Vulnerability scanners
- Intrusion detection systems
- Security monitoring solutions