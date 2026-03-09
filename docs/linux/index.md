# Linux OS Security

This section covers the security principles and hardening techniques for the Linux operating system. Linux is the foundation of most cloud infrastructure, containers, and high-security environments. Understanding its internal security model is critical for any security engineer.

## Core Topics

- **[File System Security](file-system.md)** - Permissions (DAC), special bits (SUID/SGID), and secure mounting.
- **[Kernel & Userspace](kernel-userspace.md)** - Memory protection, system calls, and the boundary between kernel and user mode.
- **[Access Control Systems](access-control-systems.md)** - Deep dive into ACLs, and Mandatory Access Control (SELinux, AppArmor).
- **[User Management](user-management.md)** - UID/GID, sudo configuration, and the principle of least privilege.
- **[Process Security](process-security.md)** - Process isolation, signals, and monitoring running services.
- **[Hardening & Compliance](hardening-compliance.md)** - CIS Benchmarks, automated auditing, and system-wide security policies.
- **[VM & Container Security](vm-container-sec.md)** - Namespaces, cgroups, hypervisors, and isolation technology.
- **[Network Security](network-security.md)** - Linux network stack hardening, sysctl tuning, and secure protocols.
- **[Firewalls](firewalls.md)** - Netfilter, iptables, nftables, and frontend tools like UFW/Firewalld.
- **[Logging & Monitoring](logging-monitoring.md)** - Syslog, journald, auditd, and centralized log analysis.

## Key Learning Objectives

1.  **Understand the Linux Security Model**: Master the concepts of DAC vs. MAC and how the kernel enforces boundaries.
2.  **Privilege Management**: Learn how to effectively use `sudo`, capabilities, and user namespaces to minimize risk.
3.  **System Hardening**: Apply industry-standard benchmarks (CIS, STIG) to secure a Linux distribution from the ground up.
4.  **Isolation Technologies**: Deep dive into how containers and VMs work at the kernel level to provide security boundaries.
5.  **Auditing and Visibility**: Configure comprehensive logging and auditing to detect and investigate security incidents.
6.  **Secure Communication**: Harden the network stack and manage firewall rules to protect services from external threats.

---

*For interview preparation related to Linux security, see the [Security Questions](../interview-prep/questions.md) section.*
