# Linux Logging and Monitoring

Effective logging and monitoring are critical for security, providing the visibility needed to detect threats, investigate incidents, and ensure compliance. This section covers the core components of the Linux logging architecture and key security monitoring techniques.

## System Logging

Most modern Linux distributions use a logging system based on `rsyslog` or `systemd-journald`.

### `systemd-journald`

`systemd-journald` is a centralized logging service that collects and stores log data in a structured, indexed binary format. It captures a wide range of information, including:
- Standard output and error from services.
- Kernel log messages (`dmesg`).
- Syslog messages.
- Audit records.

Log data can be queried using the `journalctl` command.

**Key `journalctl` commands:**
```bash
# View all logs in real-time
journalctl -f

# View logs for a specific service
journalctl -u sshd.service

# View logs since a specific time
journalctl --since "1 hour ago"

# View kernel messages
journalctl -k
```

By default, `journald` stores logs in memory or at `/run/log/journal`. To enable persistent logging across reboots, you must set `Storage=persistent` in `/etc/systemd/journald.conf` and create the directory `/var/log/journal`.

### `rsyslog`

`rsyslog` is an advanced, rule-based logging system that has been the standard for many years. It collects logs from various sources and routes them to different destinations based on a set of rules.

- **Configuration**: `/etc/rsyslog.conf` and files in `/etc/rsyslog.d/`.
- **Log Destinations**: Local files (e.g., `/var/log/messages`, `/var/log/auth.log`), remote syslog servers, or databases.

An `rsyslog` rule consists of a **selector** (facility and priority) and an **action** (where to send the log).
```
# Log all mail-related messages to /var/log/maillog
mail.*    /var/log/maillog

# Send all logs to a remote syslog server
*.*    @remote-server.example.com
```

### Centralized Logging

For any production environment, logs should be forwarded to a **centralized logging server**. This ensures:
- **Log Integrity**: Prevents an attacker from tampering with local log files to cover their tracks.
- **Correlation**: Allows security analysts to correlate events from multiple systems.
- **Long-term Retention**: Central servers can be configured with storage for long-term log retention, as required by compliance frameworks.

Popular open-source centralized logging solutions include the **ELK Stack** (Elasticsearch, Logstash, Kibana) and **Graylog**.

### Log Rotation

Log files can grow very large over time. `logrotate` is a utility that automates the rotation, compression, and deletion of log files.

- **Configuration**: `/etc/logrotate.conf` and files in `/etc/logrotate.d/`.
- **Actions**: `logrotate` can be configured to rotate logs daily, weekly, or when they reach a certain size.

## Security Monitoring

### File Integrity Monitoring (FIM)

FIM tools monitor critical system files and alert administrators when they are created, modified, or deleted. This is essential for detecting unauthorized changes that could indicate a compromise.

- **AIDE (Advanced Intrusion Detection Environment)**: A popular FIM tool. It creates a baseline database of file checksums and other attributes. It can then be run periodically to compare the current state of the system against the baseline.
  ```bash
  # Initialize the AIDE database
  aide --init

  # Check the system against the database
  aide --check
  ```
- **Tripwire**: Another widely used FIM tool with similar functionality.

### Process and Network Monitoring

- **Process Accounting**: Tools like `psacct` or `acct` can be used to log every command run by users on the system, including the user who ran it and the time.
- **Network Monitoring**:
  - `ss` or `netstat`: Display active network connections, listening ports, and socket statistics. Essential for identifying unauthorized listening services or suspicious connections.
  - `tcpdump` and `wireshark`: Packet sniffers used to capture and analyze network traffic for signs of malicious activity.

### Intrusion Detection Systems (IDS)

An IDS monitors a system or network for malicious activity or policy violations.
- **Host-based IDS (HIDS)**: Runs on an individual host and monitors its activity. Examples include **OSSEC** and **Wazuh**. They analyze log files, check file integrity, and monitor for rootkits.
- **Network-based IDS (NIDS)**: Deployed on the network to monitor traffic for suspicious patterns. **Snort** and **Suricata** are popular examples.

### The Linux Audit Framework (`auditd`)

`auditd` is the userspace component of the Linux kernel's audit framework. It provides a highly configurable and detailed logging system for security-relevant events. It can track:
- **System calls**: Monitor for sensitive syscalls (e.g., `open`, `execve`).
- **File access**: Log every time a specific file or directory is read, written to, or executed.
- **User actions**: Track commands run by specific users.

**Example `auditd` rule (in `/etc/audit/rules.d/audit.rules`):**
```
# Watch for writes and attribute changes to /etc/passwd
-w /etc/passwd -p wa -k passwd_changes
```
Audit logs are stored in `/var/log/audit/audit.log` and can be queried with the `ausearch` and `aureport` commands.