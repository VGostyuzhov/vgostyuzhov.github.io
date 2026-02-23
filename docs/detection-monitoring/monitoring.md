# Monitoring & Analysis

## SIEM Architecture & Tooling

A SIEM (Security Information and Event Management) system aggregates logs from diverse sources, correlates events, and enables security analysis and alerting.

**SIEM architecture components:**

| Component | Function |
|-----------|---------|
| **Log collection** | Agents, syslog, API integrations, beat shippers |
| **Parsing/normalisation** | Convert diverse log formats to common schema |
| **Storage** | Indexed storage for search and retention |
| **Correlation engine** | Match events across sources using rules and patterns |
| **Alerting** | Trigger notifications based on rule matches or thresholds |
| **Dashboard/Visualization** | Real-time and historical views of security posture |
| **Case management** | Track investigations from alert to resolution |

**SIEM platforms:**

| Platform | Type | Notes |
|----------|------|-------|
| **Splunk** | Commercial | Mature, powerful SPL query language, expensive at scale |
| **Elastic Security** | Open source / Commercial | ELK stack (Elasticsearch, Logstash, Kibana) + security features |
| **Chronicle (Google)** | Cloud-native | Backed by Google infrastructure; flat-rate pricing model |
| **Microsoft Sentinel** | Cloud-native | Azure-integrated; KQL query language; SOAR built-in |
| **Wazuh** | Open source | HIDS + SIEM + vulnerability detection |
| **Graylog** | Open source / Commercial | Log management with security analytics |

**SIEM deployment models:**

- **On-premises** - full control, high operational burden
- **Cloud-native** - scalable, managed, provider lock-in risk
- **Hybrid** - sensitive logs on-prem, cloud logs in cloud SIEM

!!! info "External Resources"
    - [Splunk Security Essentials](https://www.splunk.com/en_us/products/security-essentials.html) (Splunk)
    - [Elastic Security](https://www.elastic.co/security) (Elastic)
    - [NIST SP 800-92 - Log Management Guide](https://csrc.nist.gov/publications/detail/sp/800-92/final) (NIST)

## Log Sources & Collection

Effective monitoring depends on collecting the right logs from the right sources.

**Critical log sources:**

| Source | What it provides | Priority |
|--------|-----------------|----------|
| **Authentication logs** | Login success/failure, MFA events, account lockouts | Critical |
| **Cloud control plane** | IAM changes, resource creation/deletion, policy changes | Critical |
| **DNS logs** | Domain queries, resolution results, tunneling indicators | High |
| **Firewall/proxy logs** | Allowed/denied connections, URLs accessed | High |
| **Endpoint (EDR)** | Process creation, file changes, network connections | High |
| **VPC/network flow logs** | Source/dest IP, ports, bytes, connection metadata | High |
| **Web server access logs** | HTTP requests, status codes, user agents | Medium |
| **Application logs** | Business logic events, errors, user actions | Medium |
| **Database audit logs** | Queries executed, schema changes, access patterns | Medium |
| **Email gateway logs** | Inbound/outbound email, spam/phishing detections | Medium |

**Log collection methods:**

| Method | Use Case |
|--------|----------|
| **Syslog (514/TCP/UDP)** | Network devices, Linux servers |
| **Windows Event Forwarding** | Windows endpoints and servers |
| **Agents (Beats, Fluentd, Vector)** | Structured collection from endpoints and applications |
| **API polling** | Cloud services, SaaS applications |
| **Webhook/push** | Real-time event streaming |

**Log standardisation:**

- Normalise timestamps to UTC
- Use common field names across sources (Elastic Common Schema, OCSF)
- Enrich logs with context (geo-IP, asset ownership, user department)
- Tag logs with source and classification for efficient querying

!!! info "External Resources"
    - [Elastic Common Schema](https://www.elastic.co/guide/en/ecs/current/index.html) (Elastic)
    - [OCSF - Open Cybersecurity Schema Framework](https://schema.ocsf.io/) (OCSF)
    - [SANS - Critical Log Review Checklist](https://www.sans.org/blog/list-of-resource-links-from-the-sans-poster-critical-log-review-checklist-for-security-incidents/) (SANS)

## Alert Triage & Fatigue Reduction

Alert fatigue occurs when analysts are overwhelmed by volume, leading to missed real threats. The goal is maximum signal with minimum noise.

**Causes of alert fatigue:**

- Too many low-fidelity rules generating false positives
- Duplicate alerts for the same event from multiple tools
- No prioritisation - all alerts treated equally
- Lack of context in alert data (raw log line without enrichment)
- Insufficient tuning after initial rule deployment

**Fatigue reduction strategies:**

| Strategy | How it helps |
|----------|-------------|
| **Tiered alerting** | P1 (immediate response), P2 (investigate within hours), P3 (review in batch) |
| **Alert enrichment** | Add context (asset value, user role, threat intelligence match) to each alert |
| **Correlation rules** | Combine multiple weak signals into one high-confidence alert |
| **Suppression** | Suppress known benign patterns (maintenance windows, expected scanners) |
| **Deduplication** | Group identical alerts within a time window into a single incident |
| **Automation (SOAR)** | Auto-triage known patterns; enrich and close obvious false positives |
| **Regular tuning** | Weekly review of false positive sources; adjust or retire noisy rules |

**Triage workflow:**

1. Alert fires -> auto-enriched with asset, user, and threat intel context
2. Analyst reviews enriched alert -> decides: true positive, false positive, or needs investigation
3. True positive -> create incident, begin investigation
4. False positive -> document reason, tune rule to prevent recurrence
5. Track metrics: false positive rate per rule, triage time, escalation rate

!!! info "External Resources"
    - [SANS - Alert Fatigue Reduction](https://www.sans.org/blog/alert-fatigue/) (SANS)
    - [Phantom/Splunk SOAR](https://www.splunk.com/en_us/products/splunk-security-orchestration-and-automation.html) (Splunk)
    - [TheHive - Incident Response Platform](https://thehive-project.org/) (TheHive)

## Anomaly & Behavior-Based Detection

Anomaly detection identifies deviations from established baselines of normal behaviour.

**UEBA (User and Entity Behavior Analytics):**

- Build behavioural profiles for users and entities (hosts, services, applications)
- Detect anomalies: unusual login times, abnormal data access volume, impossible travel
- Risk scoring: accumulate risk points for anomalous actions; alert when threshold exceeded
- Requires sufficient baseline period (typically 2-4 weeks)

**Behavioural indicators:**

| Indicator | Normal Baseline | Anomaly Example |
|-----------|----------------|----------------|
| Login time | 9AM-6PM weekdays | 3AM Sunday login from new country |
| Data volume | 50MB/day download | 5GB download in one hour |
| Process execution | Known application set | PowerShell with encoded commands |
| DNS queries | Consistent query volume | 10,000 queries to single domain |
| Privileged operations | Rare, during change windows | Bulk IAM changes outside window |
| File access | Regular project files | Access to HR records by engineering user |

**Specific behavioural detections to implement:**

- If action could be suspicious, increase log verbosity for that user/host
- Monitor for `HISTFILE` clearing/redirection (attacker covering tracks)
- Watch for access to `/proc` from unusual processes
- Detect brute force via login failure counts per source
- Identify port scanning by TCP SYN packets without completed handshakes

**Machine learning approaches:**

- Supervised: trained on labelled data (known attacks vs normal); requires labelled dataset
- Unsupervised: clustering and outlier detection; no labels needed but more false positives
- Practical: start with rule-based anomaly detection; add ML where rules are insufficient

!!! info "External Resources"
    - [Elastic Machine Learning](https://www.elastic.co/what-is/elastic-machine-learning) (Elastic)
    - [MITRE ATT&CK Analytics](https://car.mitre.org/) (MITRE)
    - [SANS - Behavior Analytics](https://www.sans.org/blog/user-behavior-analytics/) (SANS)

## Network Traffic Analysis

Network traffic analysis (NTA) inspects network communications to detect threats, investigate incidents, and monitor for policy violations.

**Key analysis techniques:**

| Technique | Data Source | Use Case |
|-----------|-----------|----------|
| **Full packet capture** | Raw packets (PCAP) | Deep inspection, payload analysis, forensics |
| **Flow analysis** | NetFlow/IPFIX/sFlow metadata | Volume monitoring, connection patterns, anomaly detection |
| **Protocol analysis** | Zeek/Bro connection logs | Application-layer visibility without full capture |
| **DNS analysis** | DNS query/response logs | Tunneling detection, DGA identification, C2 communication |
| **TLS inspection** | JA3/JA3S fingerprints, certificate metadata | Identify malicious TLS clients/servers without decryption |

**Network-based detection use cases:**

- Detecting C2 beaconing (regular interval connections to external hosts)
- DNS exfiltration (high entropy subdomains, unusual query volume)
- Lateral movement (unexpected internal connections, SMB/RDP from unusual sources)
- Data exfiltration (large outbound transfers, connections to cloud storage)
- Reconnaissance (port scanning patterns, service enumeration)

**Firewall rule-based detection:**

- Alert on brute force (repeated login failures from single source)
- Detect port scanning (TCP SYN without SYN-ACK, half-open connections)
- Block known malicious IPs via threat intelligence feeds
- Log and alert on denied connection attempts (reconnaissance indicators)

**Tools:** Zeek, Arkime (full PCAP indexing), Suricata (IDS + protocol logging), ntopng (flow analysis)

!!! info "External Resources"
    - [Zeek Documentation](https://docs.zeek.org/) (Zeek)
    - [JA3 - TLS Fingerprinting](https://github.com/salesforce/ja3) (Salesforce)
    - [SANS - Network Forensics](https://www.sans.org/blog/network-forensics-and-traffic-analysis/) (SANS)

## Security Metrics & KPIs

Metrics quantify detection and response effectiveness, justify security investment, and identify areas for improvement.

**Detection metrics:**

| Metric | Definition | Target |
|--------|-----------|--------|
| **MTTD** (Mean Time to Detect) | Average time from compromise to detection | Hours to days (depends on threat type) |
| **Detection coverage** | Percentage of ATT&CK techniques with active detection rules | Increasing over time |
| **True positive rate** | Percentage of alerts that are genuine threats | Target > 50% for automated alerts |
| **False positive rate** | Percentage of alerts that are benign | Reduce continuously; < 20% is good |

**Response metrics:**

| Metric | Definition | Target |
|--------|-----------|--------|
| **MTTR** (Mean Time to Respond) | Average time from detection to containment | Minutes for critical; hours for high |
| **MTTC** (Mean Time to Contain) | Average time from detection to preventing further spread | As fast as possible |
| **Dwell time** | Total time attacker is present before detection + eradication | Reduce year over year |
| **Escalation rate** | Percentage of alerts escalated to incidents | Track for staffing needs |

**Operational metrics:**

| Metric | Definition |
|--------|-----------|
| **Alert volume** | Total alerts per day/week/month by severity |
| **Analyst throughput** | Alerts triaged per analyst per shift |
| **Rule coverage** | Percentage of critical assets with monitoring |
| **Log ingestion health** | Gaps in log collection, source availability |

**Reporting:**

- Track metrics over time; trend matters more than absolute values
- Compare MTTD/MTTR to industry benchmarks
- Use metrics to justify investment (reducing MTTD by X hours saves $Y in breach cost)
- Present to leadership in business terms, not technical jargon

!!! info "External Resources"
    - [SANS - Security Metrics](https://www.sans.org/blog/security-metrics/) (SANS)
    - [Verizon DBIR](https://www.verizon.com/business/resources/reports/dbir/) (Verizon)
    - [CIS Controls - Metrics](https://www.cisecurity.org/controls/metrics) (CIS)
