# Critical System Directories and Security Implications

## Basics and Core Concepts

**Configuration and Authentication Targets (`/etc`)**
The `/etc` directory houses host-specific system-wide configuration files and is the primary target for attackers seeking persistence or privilege escalation. Critical files include `/etc/passwd` (user attributes), `/etc/shadow` (hashed passwords), `/etc/sudoers` (privilege delegation), and `/etc/ssh/sshd_config` (remote access policies). A compromise of write permissions in this directory allows adversaries to create rogue users, modify service configurations, or alter path variables. Furthermore, strict permission settings (typically 0600 or 0640 owned by root) on `/etc/shadow` and `/etc/sudoers` are mandatory to prevent unauthorized reads leading to offline password cracking or unauthorized writes leading to immediate root access.

**Binaries and SUID Risks (`/bin`, `/sbin`, `/usr`)**
System executables reside in `/bin` (essential user binaries) and `/sbin` (essential system binaries), with non-essential counterparts in `/usr/bin` and `/usr/sbin`. From a security perspective, the integrity of these directories is paramount; unauthorized modification constitutes a "Trojan" attack, replacing legitimate tools (like `ls` or `ps`) to hide malicious activity. Engineers must monitor these directories for the presence of SUID (Set User ID) and SGID (Set Group ID) binaries. Improperly coded SUID binaries found here are frequent vectors for local privilege escalation, as they execute with the file owner's permissions (usually root) rather than the user's.

**Variable Data, Logging, and Web Roots (`/var`)**
The `/var` directory contains variable data files, including logs (`/var/log`), mail, printer spools, and often web server content (`/var/www`). Security engineering focuses heavily on `/var/log` for incident response; if an attacker gains write access here, they can sanitize logs to remove evidence of intrusion (`timestomping` or deletion). Additionally, `/var` should ideally be mounted on a separate partition to prevent a Denial of Service (DoS) attack where a flooded log directory consumes all available disk space, causing critical system services to crash.

**Runtime Information and Temporary Storage (`/proc`, `/sys`, `/tmp`)**
`/tmp` provides temporary storage and must be world-writable, making it a hotspot for race condition attacks and insecure file creation. It requires the "sticky bit" (chmod +t) to ensure users can only delete files they own, and should be mounted with `noexec` and `nosuid` to prevent malware execution. Meanwhile, `/proc` and `/sys` are pseudo-filesystems interacting directly with the kernel; they expose sensitive process information, kernel memory pointers, and hardware configuration. While generally strictly controlled, information leaks in `/proc` (such as ASLR offsets) can facilitate buffer overflow exploits.

---

## Practical Examples

**1. Securing Temporary Storage**
To mitigate risks associated with execution of malicious scripts dropped in `/tmp` or `/var/tmp`, these directories should be mounted on separate partitions with restrictive flags.
* **Configuration:** Edit `/etc/fstab`:
    ```bash
    # Example /etc/fstab entry
    /dev/vg0/tmp  /tmp  ext4  defaults,noexec,nosuid,nodev  0 0
    ```
    * `noexec`: Prevents binary execution.
    * `nosuid`: Ignores SUID bits.
    * `nodev`: Prevents character or block special devices from being interpreted.

**2. Auditing SUID Binaries**
Security engineers must regularly audit the filesystem for unauthorized SUID binaries that could lead to privilege escalation.
* **Command:**
    ```bash
    # Find all files with SUID bit set, owned by root
    find / -user root -perm -4000 -print 2>/dev/null
    ```
* **Implication:** If an unknown binary appears in this list, or if a standard utility (like `vim` or `cp`) appears here unexpectedly, it indicates a severe misconfiguration or a backdoor.

**3. Immutable Files**
For critical configuration files that should rarely change, the immutable attribute adds a layer of protection beyond standard DAC (Discretionary Access Control). Even the root user cannot delete or modify the file until the attribute is removed.
* **Command:**
    ```bash
    # Set immutable flag on /etc/shadow
    chattr +i /etc/shadow

    # Verify attributes
    lsattr /etc/shadow
    # Output: ----i---------e---- /etc/shadow
    ```

---

## Quick Reference Cheatsheet

| Directory | Primary Purpose | Critical Security Risks | Hardening / Mitigation |
| :--- | :--- | :--- | :--- |
| **`/etc`** | System configuration & Auth | Privilege escalation (modifying `sudoers`), Credential theft (`shadow`). | File integrity monitoring (FIM), strict permissions (0600/0640), `chattr +i`. |
| **`/bin`, `/sbin`** | Essential Binaries | Trojaned binaries, Vulnerable SUID executables. | FIM, audit SUID binaries regularly, mount read-only if possible. |
| **`/var/log`** | System Logs | Log tampering (covering tracks), Disk exhaustion DoS. | Remote logging (syslog-ng/rsyslog), separate partition. |
| **`/tmp`** | Temporary Files | TOCTOU (Time-of-check to Time-of-use) race conditions, malware execution. | Sticky bit (`+t`), mount with `noexec`, `nodev`, `nosuid`. |
| **`/boot`** | Boot loader files | Kernel tampering, insecure boot parameters. | Secure Boot, restrictive permissions, encrypt `/boot` if needed. |
| **`/dev`** | Device files | Direct hardware access, unmanaged block devices. | Restrict access to `/dev/mem` and `/dev/kmem`, mount with `noexec`. |



---

## External Resources for Deep Dive

!!! info "Recommended Study Materials"
    * **Filesystem Hierarchy Standard (FHS) Specification**
        * [Official FHS Documentation (refspecs.linuxfoundation.org)](https://refspecs.linuxfoundation.org/FHS_3.0/fhs-3.0.html)
        * *Context:* The authoritative source on directory layout and requirements. Essential for understanding standard compliance.

    * **CIS Benchmark for Linux (Center for Internet Security)**
        * [CIS Benchmarks (cisecurity.org)](https://www.cisecurity.org/cis-benchmarks/)
        * *Context:* Industry-standard hardening guidelines. Sections regarding "Filesystem Configuration" detail specific mount options (`nodev`, `nosuid`) for critical directories.

    * **Red Hat Enterprise Linux Security Guide**
        * [RHEL Security Hardening - File System Security](https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/8/html/security_hardening/scanning-the-system-for-security-compliance-and-vulnerabilities_security-hardening)
        * *Context:* Practical, enterprise-grade documentation on securing filesystems, setting ACLs, and managing permissions.

***

Would you like to review this material in more detail, or are you ready to proceed to the next security engineering topic?