# Linux Attack Vectors: A Study Guide

This guide provides a structured overview of common attack vectors used against Linux systems. Understanding these techniques is fundamental for designing and implementing robust security controls. The content is organized to support study for security engineering roles, focusing on core concepts and providing resources for further learning.

## Basics and Core Concepts

The process of compromising a Linux system typically follows a sequence of phases: initial access, privilege escalation, and establishing persistence. Each phase employs distinct techniques that exploit vulnerabilities in system configuration, software, or human behavior. A security engineer must be able to identify and mitigate threats at every stage of this attack lifecycle.

**Initial Access** is the first step, where an attacker gains a foothold. This is commonly achieved by exploiting exposed network services with known vulnerabilities, such as a vulnerable exposed web server or an SSH service with exploitable authentication/configuration. An open port alone is exposure, not proof of a vulnerability. Attackers also rely on weak or default credentials, which are susceptible to automated brute-force attacks. Other vectors include phishing, where a user is tricked into executing malicious code, and supply chain attacks, which involve compromising legitimate software packages or container images before they are installed.

**Privilege Escalation** follows initial access. Root escalation is one objective; an attacker may already steal application data, act using workload/cloud credentials, or disrupt service without becoming root. This is often accomplished by exploiting misconfigured `sudo` rules that grant excessive permissions, or by abusing SUID/SGID executables that run with the privileges of their owner (e.g., `root`). Kernel exploits, though complex, are another powerful method, leveraging vulnerabilities in the Linux kernel itself to elevate privileges.

**Persistence** maintains access rather than being a mandatory final phase, ensuring the attacker can maintain access across system reboots or if their initial entry point is discovered. Common methods include installing backdoors like web shells or adding an SSH public key to the `authorized_keys` file. Attackers also create malicious `systemd` services or `cron` jobs to execute their code at boot or on a schedule. Library-loading attacks depend on the program's loader and execution context: secure-execution mode restricts variables such as `LD_PRELOAD`, so it is not a universal setuid bypass. Loading a malicious kernel module requires appropriate authority and accepted module policy; a compromised kernel can falsify local observations.

### Common Attack Vectors Cheat Sheet

| Attack Phase | Technique | Description & Keywords | Mitigation |
| :--- | :--- | :--- | :--- |
| **Initial Access** | Exposed Network Services | Exploiting vulnerabilities in services like SSH, HTTP/S, SMB. | Firewall rules, vulnerability scanning, patching. |
| | Brute-Force Attacks | Guessing weak or default credentials for user accounts. | Strong password policies, `fail2ban`, key-based auth. |
| | Supply Chain Attack | Compromising a software package or container image. | Image scanning, software signing, trusted registries. |
| **Privilege Escalation** | `sudo` Misconfiguration | Overly permissive rules allowing command execution. | Principle of Least Privilege (PoLP), specific command rules. |
| | SUID/SGID Abuse | Exploiting vulnerable executables with elevated permissions. | Remove SUID/SGID bits where not needed, monitor file permissions. |
| | Kernel Exploit | Leveraging a bug in the kernel for root access. | Vendor fixes plus running-kernel activation/live-patch verification. |
| **Persistence** | SSH Key Addition | Adding attacker's public key to `authorized_keys`. | File Integrity Monitoring (FIM) on `.ssh` directory. |
| | `systemd`/`cron` Jobs | Scheduling malicious scripts to run at boot or intervals. | Monitor `systemd` unit paths and `cron` directories. |
| | `LD_PRELOAD` Hijacking | Forcing a process to load a malicious shared library. | Secure environment variables, use SELinux/AppArmor. |
| | LKM Rootkit | Malicious code loaded into the kernel to hide activity. | Restrict module-loading/signing authority and protect boot/update trust; scanners supply investigative signals, not proof of a clean kernel. |

!!! info "External Resources for Deep Dive"
    *   **Payloads All The Things - Linux Privilege Escalation:** [https://github.com/swisskyrepo/PayloadsAllTheThings/blob/master/Methodology%20and%20Resources/Linux%20-%20Privilege%20Escalation.md](https://github.com/swisskyrepo/PayloadsAllTheThings/blob/master/Methodology%20and%20Resources/Linux%20-%20Privilege%20Escalation.md) (A comprehensive repository of Linux privilege escalation techniques and commands).
    *   **GTFOBins:** [https://gtfobins.github.io/](https://gtfobins.github.io/) (A curated list of Unix binaries that can be used to bypass local security restrictions in misconfigured systems).
    *   **Linux Kernel Exploitation Series by 0xax:** [https://0xax.gitbooks.io/linux-insides/content/Misc/linux-kernel-exploits.html](https://0xax.gitbooks.io/linux-insides/content/Misc/linux-kernel-exploits.html) (An in-depth, technical resource for understanding the mechanics of kernel-level exploits).
## Trace Authority Instead of Collecting Tool Names

```text
Web service compromised as appsvc
 -> appsvc can edit /srv/tasks/report.sh
 -> root timer executes that script
 -> attacker-controlled content runs under root authority
# Root-owned timer unit alone does not make its writable dependencies trustworthy.
```

Protect executable/configuration/parent paths used by privileged consumers, review broad sudo/capability/socket access, and reduce exposed secrets. On compromise, separate containment from evidence collection and known-good recovery; killing one process or running a rootkit scanner does not establish that persistence is gone. Rebuild or restore trust according to the incident's scope, including external credentials.

!!! info "External Resources"
    - [Dynamic Loader Secure-Execution Rules](https://man7.org/linux/man-pages/man8/ld.so.8.html) (Linux man-pages)
    - [Systemd Unit Search Paths and Drop-ins](https://manpages.debian.org/bookworm/systemd/systemd.unit.5.en.html) (systemd manual)
