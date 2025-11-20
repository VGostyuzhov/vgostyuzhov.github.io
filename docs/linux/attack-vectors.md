# Linux Attack Vectors: A Study Guide

This guide provides a structured overview of common attack vectors used against Linux systems. Understanding these techniques is fundamental for designing and implementing robust security controls. The content is organized to support study for security engineering roles, focusing on core concepts and providing resources for further learning.

## Basics and Core Concepts

The process of compromising a Linux system typically follows a sequence of phases: initial access, privilege escalation, and establishing persistence. Each phase employs distinct techniques that exploit vulnerabilities in system configuration, software, or human behavior. A security engineer must be able to identify and mitigate threats at every stage of this attack lifecycle.

**Initial Access** is the first step, where an attacker gains a foothold. This is commonly achieved by exploiting exposed network services with known vulnerabilities, such as an unpatched web server or an open SSH port. Attackers also rely on weak or default credentials, which are susceptible to automated brute-force attacks. Other vectors include phishing, where a user is tricked into executing malicious code, and supply chain attacks, which involve compromising legitimate software packages or container images before they are installed.

**Privilege Escalation** follows initial access. Once on a system with limited privileges, an attacker's primary objective is to gain administrative (`root`) access. This is often accomplished by exploiting misconfigured `sudo` rules that grant excessive permissions, or by abusing SUID/SGID executables that run with the privileges of their owner (e.g., `root`). Kernel exploits, though complex, are another powerful method, leveraging vulnerabilities in the Linux kernel itself to elevate privileges.

**Persistence** is the final phase, ensuring the attacker can maintain access across system reboots or if their initial entry point is discovered. Common methods include installing backdoors like web shells or adding an SSH public key to the `authorized_keys` file. Attackers also create malicious `systemd` services or `cron` jobs to execute their code at boot or on a schedule. Advanced techniques involve using `LD_PRELOAD` to inject malicious libraries into legitimate processes or installing a Loadable Kernel Module (LKM) rootkit, which can hide the attacker's presence from system administrators.

### Common Attack Vectors Cheat Sheet

| Attack Phase | Technique | Description & Keywords | Mitigation |
| :--- | :--- | :--- | :--- |
| **Initial Access** | Exposed Network Services | Exploiting vulnerabilities in services like SSH, HTTP/S, SMB. | Firewall rules, vulnerability scanning, patching. |
| | Brute-Force Attacks | Guessing weak or default credentials for user accounts. | Strong password policies, `fail2ban`, key-based auth. |
| | Supply Chain Attack | Compromising a software package or container image. | Image scanning, software signing, trusted registries. |
| **Privilege Escalation** | `sudo` Misconfiguration | Overly permissive rules allowing command execution. | Principle of Least Privilege (PoLP), specific command rules. |
| | SUID/SGID Abuse | Exploiting vulnerable executables with elevated permissions. | Remove SUID/SGID bits where not needed, monitor file permissions. |
| | Kernel Exploit | Leveraging a bug in the kernel for root access. | Regular kernel patching, `unattended-upgrades`. |
| **Persistence** | SSH Key Addition | Adding attacker's public key to `authorized_keys`. | File Integrity Monitoring (FIM) on `.ssh` directory. |
| | `systemd`/`cron` Jobs | Scheduling malicious scripts to run at boot or intervals. | Monitor `systemd` unit paths and `cron` directories. |
| | `LD_PRELOAD` Hijacking | Forcing a process to load a malicious shared library. | Secure environment variables, use SELinux/AppArmor. |
| | LKM Rootkit | Malicious code loaded into the kernel to hide activity. | Kernel hardening, `rkhunter`, `chkrootkit`, secure boot. |

!!! info "External Resources for Deep Dive"
    *   **Payloads All The Things - Linux Privilege Escalation:** [https://github.com/swisskyrepo/PayloadsAllTheThings/blob/master/Methodology%20and%20Resources/Linux%20-%20Privilege%20Escalation.md](https://github.com/swisskyrepo/PayloadsAllTheThings/blob/master/Methodology%20and%20Resources/Linux%20-%20Privilege%20Escalation.md) (A comprehensive repository of Linux privilege escalation techniques and commands).
    *   **GTFOBins:** [https://gtfobins.github.io/](https://gtfobins.github.io/) (A curated list of Unix binaries that can be used to bypass local security restrictions in misconfigured systems).
    *   **Linux Kernel Exploitation Series by 0xax:** [https://0xax.gitbooks.io/linux-insides/content/Misc/linux-kernel-exploits.html](https://0xax.gitbooks.io/linux-insides/content/Misc/linux-kernel-exploits.html) (An in-depth, technical resource for understanding the mechanics of kernel-level exploits).