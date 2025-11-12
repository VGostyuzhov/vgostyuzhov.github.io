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

### Discretionary Access Control (DAC)

- Traditional Unix permissions (rwx)
- Access Control Lists (ACLs)
- File ownership and groups
- SUID/SGID/sticky bits

### Mandatory Access Control (MAC)

- SELinux architecture and policies
- AppArmor profiles and enforcement
- RBAC and MLS/MCS models
- Policy development and debugging

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