# *nix Systems Security

Unix-like systems (Linux, Unix, BSD) form the backbone of modern infrastructure. Understanding their security model is essential for infrastructure and cloud security roles.

## System Architecture & Security Model

### Kernel vs Userspace
- Kernel space privileges and syscalls
- User space isolation and boundaries
- Process rings and protection levels
- System call interface security

### File System Security
- File permissions and ownership
- Special file types and attributes
- Mount points and filesystem options
- File system hardening techniques

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