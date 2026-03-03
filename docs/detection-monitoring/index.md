# Detection Systems

## Intrusion Detection Systems (IDS/IPS)

**IDS (Intrusion Detection System)** monitors network traffic or host activity and generates alerts on suspicious behaviour. **IPS (Intrusion Prevention System)** actively blocks detected threats.

| Property | IDS | IPS |
|----------|-----|-----|
| Mode | Passive (monitors copy of traffic) | Inline (traffic passes through) |
| Action | Alert only | Alert + block/drop |
| Risk | No impact on traffic flow | False positives block legitimate traffic |
| Deployment | SPAN port, TAP, or host agent | Inline between network segments |

**IDS types:**

| Type | Deployment | Monitors | Examples |
|------|-----------|----------|---------|
| **NIDS** (Network) | Network tap/SPAN port | Network packets | Snort, Suricata, Zeek |
| **HIDS** (Host) | Agent on endpoint | Files, processes, logs, syscalls | OSSEC, Wazuh, Samhain |
| **Wireless IDS** | Wireless sensor | 802.11 frames | Kismet, AirMagnet |

**Deployment considerations:**

- Place NIDS at network choke points (perimeter, between segments, in front of critical services)
- HIDS complements NIDS by detecting host-level attacks that network monitoring misses
- Encrypted traffic (TLS) limits NIDS visibility; consider TLS inspection or endpoint-level detection
- Tune rules aggressively to reduce false positives; untriaged alerts are worthless

!!! info "External Resources"
    - [NIST SP 800-94 - Guide to IDS/IPS](https://csrc.nist.gov/publications/detail/sp/800-94/final) (NIST)
    - [Snort Documentation](https://www.snort.org/documents) (Snort)
    - [Suricata Documentation](https://docs.suricata.io/) (Suricata)

## Signature-Based vs Behavior-Based Detection

| Property | Signature-Based | Behavior-Based |
|----------|----------------|---------------|
| **How it works** | Matches traffic/activity against known patterns | Learns "normal" baseline; alerts on deviations |
| **Strengths** | Low false positives for known threats; precise | Detects unknown/zero-day threats; adapts |
| **Weaknesses** | Cannot detect novel attacks; requires signature updates | Higher false positive rate; requires tuning and baseline period |
| **Examples** | Snort/Suricata rules, YARA, antivirus signatures | ML-based anomaly detection, UEBA, Zeek behavioural scripts |

**Signature-based detection signals:**

- Host-based: registry changes, files created/modified, known malware strings in binaries (antivirus)
- Network-based: DNS queries to known C2 domains, specific byte patterns in packet payloads, known exploit signatures

**Behavior-based detection signals:**

- User-specific anomalies: unusual login times, unusual file access patterns, impossible travel
- Process anomalies: web server spawning shell, unusual parent-child process relationships
- Network anomalies: unusual outbound destinations, beaconing patterns, DNS query volume spikes
- System anomalies: unexpected service starts, privilege escalation patterns, `HISTFILE` manipulation

**Practical approach:** Layer both. Signatures catch known threats quickly and cheaply. Behaviour-based detection catches what signatures miss. Combine with threat intelligence for enrichment.

!!! info "External Resources"
    - [Sigma Rules - Generic Signature Format](https://github.com/SigmaHQ/sigma) (Sigma)
    - [Elastic Detection Rules](https://github.com/elastic/detection-rules) (Elastic)
    - [MITRE ATT&CK Detection](https://attack.mitre.org/resources/getting-started/) (MITRE)

## Rule Writing (Snort, Suricata, YARA)

**Snort/Suricata rule format:**

```
action protocol source_ip source_port -> dest_ip dest_port (options)
```

**Example rules:**

```
# Alert on DNS query to known malicious domain
alert dns any any -> any any (msg:"Malicious DNS query"; dns.query; content:"evil.com"; nocase; sid:1000001; rev:1;)

# Alert on HTTP request with SQL injection pattern
alert http any any -> any any (msg:"SQL injection attempt"; http.uri; content:"' OR 1=1"; nocase; sid:1000002; rev:1;)

# Alert on outbound connection to C2 IP
alert ip any any -> 203.0.113.66 any (msg:"C2 communication"; sid:1000003; rev:1;)
```

**YARA rule format (file/memory scanning):**

```
rule Detect_WebShell {
    meta:
        description = "Detect common PHP web shell"
        severity = "high"
    strings:
        $eval = "eval(" ascii
        $base64 = "base64_decode(" ascii
        $system = "system(" ascii
        $exec = "exec(" ascii
    condition:
        filesize < 50KB and 2 of them
}
```

**Rule writing best practices:**

- Start with specific, high-confidence rules; expand coverage iteratively
- Test rules against representative traffic before deploying to production
- Include metadata: description, severity, MITRE ATT&CK mapping, author, date
- Version rules and track changes in source control
- Monitor rule performance - expensive rules can cause packet drops under load
- Use Sigma as a vendor-neutral format; convert to Snort/Suricata/Splunk as needed

!!! info "External Resources"
    - [Snort Rule Writing Guide](https://www.snort.org/faq/what-is-a-snort-rule) (Snort)
    - [Suricata Rule Management](https://docs.suricata.io/en/latest/rules/) (Suricata)
    - [YARA Documentation](https://yara.readthedocs.io/) (VirusTotal)

## Honeypots & Canary Tokens

**Honeypots** are decoy systems designed to attract attackers, detect intrusion, and gather intelligence about attacker behaviour.

| Type | Fidelity | Use Case |
|------|----------|----------|
| **Low interaction** | Emulates services superficially | Detection at scale; easy to deploy (Honeyd, Dionaea) |
| **Medium interaction** | Emulates protocols more completely | Capture exploit payloads; malware collection |
| **High interaction** | Full operating system/application | Deep attacker behavioural analysis; research |

**Canary tokens:**

- Tripwire-style alerts triggered when an attacker accesses or uses a planted resource
- No infrastructure required; generate tokens at [canarytokens.org](https://canarytokens.org)

| Token Type | Trigger | Example |
|-----------|---------|---------|
| **DNS canary** | DNS resolution of unique hostname | Planted in config files, documentation |
| **Web bug** | HTTP request to unique URL | Embedded in documents, spreadsheets |
| **AWS key** | Use of fake AWS access key | Planted in code repos, config files |
| **Email** | Email opened or forwarded | Sent to unused/monitoring mailbox |
| **File open** | Document opened (Word, PDF) | Placed on shared drives, honeypot servers |

**Deployment strategy:**

- Deploy honeypots on unused IP addresses in each network segment
- Place canary tokens in high-value locations (credential files, admin directories, cloud configs)
- Any interaction with a honeypot/canary is by definition suspicious (no legitimate use)
- Integrate alerts with SIEM for correlation with other indicators

!!! info "External Resources"
    - [Canarytokens](https://canarytokens.org/) (Thinkst)
    - [T-Pot Honeypot Platform](https://github.com/telekom-security/tpotce) (T-Systems)
    - [MITRE ATT&CK - Honeypots](https://attack.mitre.org/techniques/T1583/) (MITRE)

## Indicators of Compromise (IOCs)

IOCs are specific, observable artifacts that indicate a system has been compromised or an attack is underway.

**IOC types:**

| IOC Type | Examples | Sharing Format |
|----------|---------|---------------|
| **Network** | IP addresses, domains, URLs, JA3 hashes | STIX/TAXII, OpenIOC |
| **Host** | File hashes (MD5, SHA256), file paths, registry keys | YARA rules, STIX |
| **Email** | Sender addresses, subject lines, attachment hashes | STIX, email gateway rules |
| **Behavioral** | Process command lines, execution patterns | Sigma rules, MITRE ATT&CK |

**IOC lifecycle:**

1. **Collection** - gather from threat intelligence feeds, incident investigations, community sharing
2. **Enrichment** - add context (threat actor, campaign, severity, confidence)
3. **Distribution** - push to detection tools (IDS, SIEM, EDR, firewall)
4. **Detection** - match against live traffic and host telemetry
5. **Expiration** - age out stale IOCs (IP addresses rotate; domains expire)

**Sharing platforms:**

- MISP (Malware Information Sharing Platform) - open source threat intelligence platform
- VirusTotal - malware and URL analysis with community intelligence
- AlienVault OTX - open threat exchange
- Anomali ThreatStream - commercial threat intelligence

**Pyramid of Pain (David Bianco):**

Ranks IOC types by how much pain they cause the attacker when blocked:

1. Hash values (trivial to change)
2. IP addresses (easy to change)
3. Domain names (more effort to change)
4. Network/host artifacts (moderate effort)
5. Tools (significant effort to replace)
6. TTPs (hardest to change - requires new tradecraft)

Focus detection on higher levels (TTPs, tools) for more durable defences.

!!! info "External Resources"
    - [Pyramid of Pain - David Bianco](https://detect-respond.blogspot.com/2013/03/the-pyramid-of-pain.html) (Detect-Respond)
    - [MISP Project](https://www.misp-project.org/) (MISP)
    - [STIX/TAXII - OASIS](https://oasis-open.github.io/cti-documentation/) (OASIS)

## Detection Strategy Design

Detection strategy defines what to detect, how to detect it, and how to measure detection effectiveness.

**Detection engineering process:**

1. **Identify threats** - use threat intelligence and ATT&CK to determine relevant adversary techniques
2. **Assess visibility** - what data sources are available (logs, network, endpoint telemetry)
3. **Map coverage** - which ATT&CK techniques have detection rules vs coverage gaps
4. **Write detections** - create rules/queries for priority techniques
5. **Test detections** - validate with red team exercises, atomic tests, or replayed attack data
6. **Tune and maintain** - reduce false positives, update for new attack variants, retire stale rules

**Detection maturity model:**

| Level | Capability |
|-------|-----------|
| 0 | Minimal - no active detection |
| 1 | Reactive - respond to alerts from vendor tools |
| 2 | Proactive - custom detection rules for key threats |
| 3 | Hunting - proactively search for threats not covered by rules |
| 4 | Automated - detection-as-code, CI/CD for rules, ML-assisted |

**Data sources for detection:**

- Endpoint: process creation, file modification, registry changes, network connections
- Network: flow data, DNS queries, proxy logs, IDS alerts
- Cloud: control plane logs, data plane logs, VPC flow logs
- Identity: authentication events, privilege changes, token usage
- Application: access logs, error logs, business logic events

**Increasing log verbosity selectively:**

- If initial investigation suggests suspicious activity, increase logging for that user/host
- Windows: enable PowerShell script block logging, Sysmon
- Linux: enable auditd rules for suspicious commands, process execution
- Cloud: enable data event logging for specific resources

!!! info "External Resources"
    - [Detection Engineering with ATT&CK](https://www.mitre.org/sites/default/files/publications/pr-18-0944-11-mitre-attack-design-and-philosophy.pdf) (MITRE)
    - [Atomic Red Team](https://github.com/redcanaryco/atomic-red-team) (Red Canary)
    - [Palantir Alerting and Detection Strategy](https://github.com/palantir/alerting-detection-strategy-framework) (Palantir)
