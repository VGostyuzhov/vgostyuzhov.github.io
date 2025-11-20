# Linux Logging and Monitoring: A Study Guide

This guide covers the essential components of logging and security monitoring on Linux systems. Effective logging provides the visibility required to detect malicious activity, perform forensic analysis, and meet compliance obligations.

## Basics and Core Concepts

Modern Linux logging is primarily handled by two systems: **`systemd-journald`** and **`rsyslog`**. `systemd-journald` (the "journal") is the default on most current distributions. It captures log data from the kernel, services, and other sources into a structured, indexed binary format. This allows for powerful and efficient querying with the `journalctl` utility. For enterprise environments, it is critical to configure the journal for persistent storage so that logs survive a reboot, which is a common tactic used by attackers to clear volatile data.

**`rsyslog`** is a highly configurable, rule-based log routing daemon that has long been the standard in the Linux world. It can collect logs from various sources and forward them to different destinations—such as local files or, more importantly, a remote server—based on rules defined by facility and priority. In any production environment, logs from all systems should be forwarded to a **centralized logging server** (e.g., an ELK stack or Graylog instance). This practice is fundamental to security as it ensures log integrity by preventing attackers from tampering with local files to hide their activity.

Beyond basic system logging, security monitoring involves actively looking for signs of compromise. **File Integrity Monitoring (FIM)** is a critical process in this domain. FIM tools like **AIDE** or **Tripwire** create a cryptographic baseline of important system files and periodically check for any unauthorized modifications, which could signal a rootkit installation or other malicious changes. This serves as a foundational component of a Host-based Intrusion Detection System (HIDS).

The **Linux Audit Framework**, accessed via the `auditd` daemon, provides the most granular level of monitoring available. It can be configured to create a detailed audit trail of security-relevant events by hooking directly into the kernel to log specific system calls. For example, `auditd` can be configured to log every attempt to read, write, or execute a sensitive file (e.g., `/etc/passwd`), track all commands run by a specific user, or monitor for changes to system configuration. This detailed, low-level logging is invaluable for forensic investigations and is a requirement for many compliance standards.

### Linux Monitoring & Logging Cheat Sheet

| Technology | Type | Purpose | Key Tool(s) |
| :--- | :--- | :--- | :--- |
| **`systemd-journald`** | Logging | Centralized log collection from system services. | `journalctl` |
| **`rsyslog`** | Logging | Rule-based log forwarding, especially to remote servers. | `rsyslogd` |
| **`logrotate`** | Logging | Manages log file rotation, compression, and deletion. | `logrotate` |
| **AIDE / Tripwire** | FIM / HIDS | Monitors for unauthorized changes to critical system files. | `aide` |
| **`auditd`** | Auditing / HIDS | Detailed, kernel-level logging of security events (syscalls, file access). | `auditctl`, `ausearch` |
| **`ss` / `netstat`**| Network Monitoring | Display active network connections and listening ports. | `ss`, `netstat` |

!!! info "External Resources for Deep Dive"
    *   **The Linux Audit Framework Project:** [https://people.redhat.com/sgrubb/audit/](https://people.redhat.com/sgrubb/audit/) (The official documentation and resources for `auditd`).
    *   **AIDE (Advanced Intrusion Detection Environment):** [https://aide.github.io/](https://aide.github.io/) (Official site for the AIDE file integrity monitoring tool).
    *   **Logstash and the ELK Stack:** [https://www.elastic.co/what-is/elk-stack](https://www.elastic.co/what-is/elk-stack) (An overview of the popular open-source centralized logging solution).