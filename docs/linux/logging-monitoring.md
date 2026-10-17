# Linux Logging and Monitoring

Detection depends on what events are produced, captured, retained, and correlated. No local log or one monitoring tool is a complete proof that a privileged attacker could not act.

## journald, Forwarding, and Retention

**systemd-journald** collects structured events from supported services/kernel sources. **rsyslog** and other agents can route logs to files and collectors. Configuration determines persistent versus volatile storage, rate limits, queueing, retention, and which events are actually forwarded.

```bash
# Read-only Linux/systemd checks; permissions determine visible records.
journalctl --list-boots
journalctl -u app.service --since '30 minutes ago' --no-pager
journalctl --disk-usage
# A visible old boot supports retention; absence can mean retention/access/configuration,
# not proof that a reboot or incident did not occur.
```

Remote collection makes already received records harder for a host-only attacker to erase when collector credentials/retention are protected. It does not guarantee that a compromised host emitted truthful/complete events or that a failing agent delivered them. Authenticate/encrypt transport, monitor queue loss/backpressure, isolate collector authority, and define timestamp/clock-skew handling.

!!! info "External Resources"
    - [Journal Storage and Rate Limits](https://manpages.debian.org/bookworm/systemd/journald.conf.5.en.html) (systemd manual, Debian distribution)
    - [Journal Query and Visibility](https://manpages.debian.org/bookworm/systemd/journalctl.1.en.html) (systemd manual, Debian distribution)

## Linux Audit Events

The kernel audit subsystem evaluates configured event rules; auditd collects the resulting stream. A successful operation can be logged, not only a denied one. Coverage depends on the rule's syscall/path/architecture filters, mounts, backlog behavior, and storage. Multiple records with the same audit event identifier may need correlation for subject, path, syscall, and result.

```bash
# Read-only inspection on an authorized audit-enabled Linux host.
sudo auditctl -s  # Inspect enabled state/backlog/lost events under actual version.
sudo auditctl -l  # Effective rules, not merely files awaiting a loader/reboot.
sudo ausearch -k study_identity -ts recent -i
# Restrict access to logs: command arguments/paths can include sensitive data.
```

For an authorized disposable lab, first create a harmless fixture in a private temporary directory. An administrator can add a narrow rule for its actual absolute path, generate a controlled write, search the resulting event, and remove exactly that rule afterward. Do not modify `/etc/passwd` to demonstrate logging, and do not clear unrelated audit rules.

```text
# Illustrative audit-rule fragment; replace the path with an existing lab fixture.
-a always,exit -F arch=b64 -F path=/srv/study-lab/identity.txt -F perm=wa -k study_identity
# Permission filters select relevant syscall/open-flag operations, not every
# individual write(2) on an already-open descriptor; this is not a read watch.
# Select the actual ABI and required compatibility coverage; b64 is not every ABI.
# Persistent loading and immutable audit-mode changes require separate administration.
```

Audit immutable configuration can prevent runtime rule changes until reboot under its mode; that does not make the host's entire evidence stream cryptographically immutable. Alert on collection gaps and resource exhaustion instead of assuming zero lost events.

!!! info "External Resources"
    - [Audit Rule Matching, Status, and Failure Policy](https://man7.org/linux/man-pages/man8/auditctl.8.html) (Linux Audit)
    - [Audit Rules and Architecture Scope](https://man7.org/linux/man-pages/man7/audit.rules.7.html) (Linux Audit)

## Integrity and Container Evidence

**File integrity monitoring (FIM)** compares current metadata/content with a baseline. Protect the baseline independently and reconcile approved package/deployment changes; automatically approving every change after an alert defeats the purpose. An authorized change and an attacker modification can both produce a hash difference.

Container stdout/stderr, node journals, Kubernetes audit logs, and runtime/kernel telemetry observe different layers. A restarted pod can lose local writable-layer evidence; a node agent may miss events while disconnected. Correlate workload UID/container identity, node identity, process lineage, and synchronized timestamps, with controlled retention. No controlling terminal, high load, or a denied syscall is an investigation clue, not a malware verdict.

!!! info "External Resources"
    - [AIDE Baselines and Integrity Checks](https://aide.github.io/) (AIDE)
    - [Kubernetes Logging Architecture](https://kubernetes.io/docs/concepts/cluster-administration/logging/) (Kubernetes)
