# Linux Hardening and Compliance: A Study Guide

This guide details the practice of system hardening and compliance for Linux environments. Hardening involves reducing a system's vulnerability by minimizing its attack surface, while compliance ensures the system adheres to established security standards.

## Basics and Core Concepts

System hardening is a proactive security measure guided by fundamental principles like the **Principle of Least Privilege**, which dictates that any user, program, or process should have only the bare minimum privileges necessary to perform its function. Another key principle is **service minimization**, which involves disabling or uninstalling all non-essential services and applications to reduce potential entry points for an attacker. These principles work together to create a more resilient and defensible system.

Rather than relying on ad-hoc configurations, hardening should follow established industry frameworks. The most recognized standards are the **CIS (Center for Internet Security) Benchmarks** and the **DISA STIGs (Security Technical Implementation Guides)** from the U.S. Department of Defense. These benchmarks provide detailed, prescriptive checklists for securing various Linux distributions. Automated tools like **OpenSCAP** can be used to scan a system's configuration and report on its compliance with these standards, identifying deviations that need to be remediated.

Key areas for hardening include the filesystem, kernel, and access controls. Securing the filesystem involves creating separate partitions for critical directories (`/tmp`, `/home`, `/var`) and applying restrictive mount options (`nodev`, `nosuid`, `noexec`) in `/etc/fstab` to limit the potential for privilege escalation. The kernel's behavior can be tuned for security using `sysctl.conf` to mitigate network-based attacks by, for example, enabling SYN cookies to counter SYN floods or randomizing memory layout (`kernel.randomize_va_space`) to make buffer overflow attacks more difficult.

Finally, a critical component of a hardened system is a **Mandatory Access Control (MAC)** framework like **SELinux** or **AppArmor**. Unlike traditional Discretionary Access Control (DAC), MAC systems enforce a system-wide policy that confines every process, including `root`, to a limited set of actions and resources. SELinux (used in RHEL-based systems) and AppArmor (used in Debian/Ubuntu) provide an essential layer of defense against privilege escalation and are a cornerstone of modern Linux security.

### Linux Hardening Cheat Sheet

| Hardening Area | Technique | Description & Keywords | Key Tools/Files |
| :--- | :--- | :--- | :--- |
| **Filesystem** | Separate Partitions | Isolate key directories like `/tmp`, `/var` to prevent system-wide impact. | `/etc/fstab` |
| | Restrictive Mounts | Use `nodev`, `nosuid`, `noexec` options to limit risks. | `/etc/fstab` |
| **Services** | Minimize Attack Surface | Disable or uninstall all unnecessary services (e.g., `cups`). | `systemctl` |
| **Kernel Tuning** | `sysctl` Hardening | Mitigate network attacks and improve memory security. | `/etc/sysctl.conf` |
| **Access Control**| Mandatory Access Control | Enforce strict confinement of processes, even `root`. | SELinux, AppArmor |
| **Compliance** | Automated Scanning | Scan system against a security baseline (e.g., CIS). | OpenSCAP |

!!! info "External Resources for Deep Dive"
    *   **CIS Benchmarks:** [https://www.cisecurity.org/cis-benchmarks/](https://www.cisecurity.org/cis-benchmarks/) (Industry-standard configuration guidelines for hardening a wide variety of systems).
    *   **DISA STIGs:** [https://public.cyber.mil/stigs/](https://public.cyber.mil/stigs/) (Security Technical Implementation Guides from the U.S. Department of Defense).
    *   **OpenSCAP Project:** [https://www.open-scap.org/](https://www.open-scap.org/) (An open-source framework for automated vulnerability scanning and compliance checking).