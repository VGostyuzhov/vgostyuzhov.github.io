# Linux Hardening and Compliance

System hardening is the process of securing a system by reducing its surface of vulnerability. For Linux, this involves applying security best practices, configuring the OS to be more resilient to attack, and ensuring it complies with industry or organizational standards.

## System Hardening Principles

The goal of hardening is to eliminate as many security risks as possible. This is guided by a few core principles:

- **Principle of Least Privilege**: Users, services, and applications should have only the minimum permissions required to perform their function.
- **Service Minimization**: Disable or uninstall any unnecessary services, applications, and network ports to reduce the attack surface.
- **Defense in Depth**: Implement multiple layers of security controls, so that if one control fails, others are in place to stop an attack.

## Hardening Benchmarks

Instead of manually deciding what to secure, it is best to follow established industry benchmarks. These provide a detailed, prescriptive checklist of security configurations.

- **CIS (Center for Internet Security) Benchmarks**: The CIS Benchmarks are globally recognized as a standard for securing IT systems. They provide detailed, step-by-step guidance for a wide range of operating systems, including various Linux distributions.
- **DISA STIGs (Defense Information Systems Agency - Security Technical Implementation Guides)**: The STIGs are configuration standards for U.S. Department of Defense (DoD) systems. They are more stringent than the CIS Benchmarks and are mandatory for military systems.

Automated tools like **OpenSCAP** can be used to scan a system for compliance with these benchmarks and report on any deviations.

## Key Hardening Areas

### 1. Filesystem and Partitions

- **Separate Partitions**: Use separate partitions for key directories like `/home`, `/tmp`, `/var`, and `/var/log`. This prevents a runaway process in one partition (e.g., filling up `/tmp`) from crashing the entire system.
- **Mount Options**: Apply restrictive mount options in `/etc/fstab`.
  - `nodev`: Do not interpret character or block special devices on the filesystem.
  - `nosuid`: Do not allow SUID/SGID bits to take effect.
  - `noexec`: Do not allow execution of any binaries on the filesystem.
  
  ```
  # Example fstab entry for /tmp
  /dev/sdb1   /tmp    ext4    defaults,nodev,nosuid,noexec    0 0
  ```

### 2. Service and Network Minimization

- **Disable Unused Services**: Use `systemctl` to identify and disable any services that are not required.
  ```bash
  # List all running services
  systemctl --type=service --state=running
  
  # Disable the CUPS printing service if not needed
  systemctl disable --now cups.service
  ```
- **Firewall Configuration**: Implement a strict firewall policy that denies all incoming traffic by default and only allows traffic to specific, required services (e.g., SSH on port 22). Use `firewalld`, `iptables`, or `ufw`.

### 3. Kernel Hardening

The kernel's behavior can be tuned for security via `/etc/sysctl.conf`. These settings can help mitigate common network-based attacks.

**Example `sysctl.conf` settings:**
```
# Disable IP forwarding (unless the system is a router)
net.ipv4.ip_forward = 0

# Enable SYN cookies to mitigate SYN flood attacks
net.ipv4.tcp_syncookies = 1

# Ignore ICMP broadcast requests
net.ipv4.icmp_echo_ignore_broadcasts = 1

# Log malformed packets
net.ipv4.conf.all.log_martians = 1

# Randomize memory space to make buffer overflow attacks harder
kernel.randomize_va_space = 2
```

### 4. Mandatory Access Control (MAC)

Standard Linux Discretionary Access Control (DAC) is not enough to stop a compromised root user. **Mandatory Access Control (MAC)** systems like **SELinux** (Security-Enhanced Linux) and **AppArmor** provide a second layer of defense.

- **SELinux**: Enforces a strict policy on what every user, process, and file can do. Even the `root` user is constrained by the SELinux policy. It is enabled by default on RHEL-based systems.
- **AppArmor**: A profile-based MAC system that confines individual applications to a specific set of files and capabilities. It is used by default on Debian and Ubuntu.

Enabling and configuring a MAC framework is a critical step in hardening a modern Linux system.

## Compliance and Auditing

Ensuring a system is hardened is not a one-time task. It must be continuously monitored to ensure it remains compliant with security policies.

- **Audit Trails**: The Linux `auditd` framework is essential for compliance. It must be configured to log all security-relevant events, such as logins, use of `sudo`, and changes to critical files. PCI DSS, for example, has strict requirements for audit logging and retention.
- **Configuration Management**: Tools like **Ansible**, **Puppet**, or **SaltStack** should be used to enforce and maintain a secure baseline configuration across all systems. This ensures that any configuration drift is automatically corrected.
- **Vulnerability Management**: Regularly scan the system for known vulnerabilities using tools like **OpenVAS** or commercial scanners, and apply security patches in a timely manner.