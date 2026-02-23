# Incident Management

## Incident Classification

Proper classification of incidents determines the response process, stakeholder involvement, legal obligations, and communication requirements. The two primary categories - security incidents and privacy incidents - have distinct characteristics and may trigger different regulatory obligations.

**Security Incident:**

A security incident is any event that compromises the confidentiality, integrity, or availability of information systems or data. Examples include unauthorized access, malware infection, denial of service, data exfiltration, and insider threat activity. Security incidents are primarily handled by security operations and incident response teams, with escalation to management based on severity.

**Privacy Incident:**

A privacy incident specifically involves the unauthorized access, disclosure, or loss of personally identifiable information (PII) or protected data categories. Privacy incidents carry additional regulatory obligations: GDPR requires notification to the supervisory authority within 72 hours; HIPAA mandates notification to affected individuals, HHS, and potentially media outlets depending on the number of individuals affected. Privacy incidents require immediate involvement of legal counsel and the Data Protection Officer (if applicable).

**Key Distinction:**

Not all security incidents are privacy incidents, and not all privacy incidents originate from security failures. A database breach exposing customer records is both. An employee accidentally emailing PII to the wrong recipient is a privacy incident that may not involve a security compromise. A DDoS attack is a security incident that may not involve any personal data.

**Severity Classification:**

| Severity | Criteria | Response Time | Escalation |
|:---|:---|:---|:---|
| Critical (P1) | Active data breach, widespread system compromise, business-critical systems down | Immediate | Executive leadership, legal, PR |
| High (P2) | Confirmed compromise with limited scope, significant vulnerability exploited | Within 1 hour | Security management, affected system owners |
| Medium (P3) | Suspicious activity under investigation, contained malware | Within 4 hours | Security operations lead |
| Low (P4) | Policy violation, unsuccessful attack attempt, false positive investigation | Within 24 hours | Analyst on duty |

!!! info "External Resources"
    - [NIST SP 800-61r2 - Computer Security Incident Handling Guide](https://csrc.nist.gov/publications/detail/sp/800-61/rev-2/final) (NIST)
    - [GDPR Breach Notification Requirements](https://gdpr-info.eu/art-33-gdpr/) (GDPR Info)
    - [FIRST CSIRT Services Framework](https://www.first.org/standards/frameworks/csirts/csirt_services_framework_v2.1) (FIRST)

## Response Models

### PICERL

PICERL is the foundational incident response lifecycle model, derived from NIST SP 800-61. It defines six sequential (though iterative) phases:

**1. Preparation:**

- Establish the incident response team, define roles, and document contact lists
- Develop and maintain incident response playbooks for common incident types
- Deploy and tune detection capabilities (SIEM, EDR, IDS/IPS)
- Conduct tabletop exercises and red team engagements
- Ensure legal agreements, communication templates, and escalation paths are in place

**2. Identification:**

- Detect and confirm that an incident has occurred
- Determine scope: which systems, data, and users are affected
- Assign severity and classify the incident type
- Begin documentation and evidence preservation immediately
- Alert types that feed identification: SIEM correlation rules, EDR detections, user reports, threat intelligence matches, anomaly detection

**3. Containment:**

- **Short-term containment**: Isolate affected systems from the network while preserving evidence (e.g., VLAN isolation, firewall blocks)
- **Long-term containment**: Apply temporary fixes that allow business operations to continue while investigation proceeds (e.g., patching, credential resets)
- Risk assessment: weigh the risk of tipping off the attacker against the risk of continued compromise
- Document all containment actions and their timestamps

**4. Eradication:**

- Remove the root cause of the incident from the environment
- Eliminate malware, close unauthorized access paths, patch exploited vulnerabilities
- Distinguish between the symptom (the alert that was triggered) and the cause (the underlying vulnerability or misconfiguration)
- Verify eradication by scanning affected and adjacent systems

**5. Recovery:**

- Restore affected systems to normal operation from known-good backups or rebuilt images
- Monitor restored systems for signs of re-compromise
- Gradually restore services with increased logging and monitoring
- Validate that business functions operate correctly

**6. Lessons Learned:**

- Conduct a post-incident review (blameless postmortem) within 1-2 weeks of incident closure
- Document root cause analysis, timeline of events, actions taken, and outcomes
- Identify process improvements, detection gaps, and prevention opportunities
- Update playbooks, detection rules, and training based on findings
- Track remediation actions to completion

### IMAG (Incident Management At Google)

IMAG adapts principles from Google's Site Reliability Engineering (SRE) practices to security incident management. Key principles include:

- **Structured role assignment**: Incident Commander (IC), Communications Lead, Operations Lead, and subject matter experts
- **Clear escalation criteria**: Severity-based triggers for involving additional teams and leadership
- **Blameless postmortems**: Focus on systemic improvements rather than individual fault
- **Documentation-first approach**: All actions, decisions, and findings are recorded in real time
- **Iterative response**: Continuously reassess scope and severity as new information emerges

!!! info "External Resources"
    - [NIST SP 800-61r2 - Incident Handling Guide](https://csrc.nist.gov/publications/detail/sp/800-61/rev-2/final) (NIST)
    - [Google SRE Book - Managing Incidents](https://sre.google/sre-book/managing-incidents/) (Google)
    - [SANS Incident Handler's Handbook](https://www.sans.org/white-papers/33901/) (SANS Institute)

## Incident Roles & Communication

Effective incident response requires clearly defined roles with explicit authority and communication protocols. Ambiguity in roles leads to duplicated effort, missed tasks, and delayed response.

**Core Incident Roles:**

- **Incident Commander (IC)**: Has overall authority and decision-making responsibility. Delegates tasks, manages resources, and serves as the single point of coordination. The IC does not perform technical analysis directly.
- **Communications Lead**: Manages all internal and external communications. Determines what information is shared with whom and when. Drafts status updates for stakeholders.
- **Operations Lead**: Directs the technical response. Coordinates containment, eradication, and recovery activities. Assigns tasks to technical responders.
- **Scribe/Documenter**: Maintains the real-time incident log. Records all decisions, actions, findings, and timestamps.
- **Subject Matter Experts (SMEs)**: Brought in as needed based on affected systems (e.g., cloud engineer, database administrator, network engineer).

**Communication Management:**

When to involve specific stakeholders:

- **Legal counsel**: Immediately for any incident involving personal data, potential regulatory violations, or law enforcement interaction. Legal should advise on evidence preservation obligations and disclosure requirements.
- **Affected users**: Only after legal review and when notification obligations are clear. Premature notification can cause unnecessary alarm; delayed notification can violate regulations.
- **Direct managers**: When their team's systems or personnel are involved. They need to reallocate resources and manage team workload.
- **Directors / Executive leadership**: For P1/P2 incidents, when business impact exceeds operational thresholds, or when external communication (press, regulators) is required.

**Managing Expectations with Upper Management:**

- Provide regular, scheduled updates rather than ad-hoc reports - establish a cadence (e.g., every 2 hours for P1)
- Be transparent about what is known, what is unknown, and what is being done to close gaps
- Avoid speculation - state facts and clearly label hypotheses
- Provide estimated timelines with explicit caveats about uncertainty
- Assume good intent from all parties, including the individuals whose systems were compromised

**Delegation Principles:**

- The IC should delegate all technical and communication tasks, retaining only coordination and decision authority
- No single individual should be both executing technical tasks and making strategic decisions
- If the incident exceeds the IC's expertise, escalate the IC role to a more experienced individual rather than having the IC attempt unfamiliar technical work

!!! info "External Resources"
    - [Incident Command System Overview](https://www.fema.gov/emergency-managers/nims/components) (FEMA)
    - [PagerDuty Incident Response Guide](https://response.pagerduty.com/) (PagerDuty)
    - [Google SRE - Incident Response Roles](https://sre.google/sre-book/managing-incidents/) (Google)

## Investigation Process

The investigation process transforms raw alerts and artifacts into an evidence-based understanding of what occurred, how, and what the attacker's objectives were.

**First Principles vs In-Depth Knowledge:**

- Investigators must balance first-principles reasoning (working from fundamental truths about how systems operate) with domain-specific expertise
- When encountering unfamiliar systems, first-principles thinking enables progress: how does authentication work on this platform, what gets logged, where are configuration files stored
- In-depth knowledge accelerates investigation: knowing that NTFS `$UsnJrnl` records file operations even when event logs are cleared, or that AWS CloudTrail logs API calls by default

**Building Timelines:**

- Construct a chronological timeline of attacker and defender actions from all available evidence sources
- Correlate timestamps across systems (account for clock skew and timezone differences)
- Use tools like plaso/log2timeline for automated super-timeline generation
- Identify gaps in the timeline - these may indicate evidence destruction or collection gaps
- Map timeline events to the Cyber Kill Chain or MITRE ATT&CK framework

**Cyber Kill Chain Application:**

1. **Reconnaissance**: What did the attacker learn about the target? (OSINT, scanning)
2. **Weaponization**: What tools or exploits were prepared?
3. **Delivery**: How was the payload delivered? (phishing, watering hole, direct exploit)
4. **Exploitation**: What vulnerability was exploited?
5. **Installation**: How did the attacker establish persistence?
6. **Command & Control (C2)**: How does the attacker communicate with compromised systems?
7. **Actions on Objectives**: What was the attacker's goal? (exfiltration, destruction, espionage)

**Symptom vs Cause:**

- The alert that initiates an investigation is frequently a symptom, not the root cause
- Example: A malware detection alert (symptom) may trace back to a phishing email exploiting a missing email gateway rule (cause)
- Root cause analysis must trace the chain of events backward to the initial failure point
- Multiple root causes may exist: a technical vulnerability, a process failure, and a human factor may all contribute

**When to Stop an Active Attack:**

- Immediate containment is warranted when: data is actively being exfiltrated, systems are being destroyed, or the attacker is spreading laterally to critical systems
- Delayed containment may be strategic when: the attacker's full scope is unknown, and premature action would alert the attacker to monitoring, causing them to destroy evidence or switch to backup access methods
- This decision must be made by the IC with input from legal and executive stakeholders

!!! info "External Resources"
    - [Lockheed Martin Cyber Kill Chain](https://www.lockheedmartin.com/en-us/capabilities/cyber/cyber-kill-chain.html) (Lockheed Martin)
    - [MITRE ATT&CK Framework](https://attack.mitre.org/) (MITRE)
    - [SANS Investigation Methodology](https://www.sans.org/white-papers/36327/) (SANS Institute)

## Containment & Eradication

Containment halts the spread of an incident while preserving evidence. Eradication removes the threat from the environment. These phases are tightly coupled - inadequate containment leads to reinfection after eradication.

**Containment Strategies:**

- **Network isolation**: Move affected hosts to a quarantine VLAN or apply firewall rules to block lateral movement while maintaining connectivity to forensic collection infrastructure
- **Account suspension**: Disable compromised accounts and revoke active sessions and tokens
- **DNS sinkholing**: Redirect C2 domains to controlled infrastructure to identify additional compromised hosts
- **Endpoint isolation**: Use EDR tools to isolate hosts at the agent level, blocking all network communication except management channel
- **Preserve before contain**: Capture memory and volatile data before isolating a system if possible

**Risk of Alerting the Attacker:**

- Containment actions visible to the attacker (e.g., killing their C2 process, resetting passwords they monitor) may trigger destructive actions
- Sophisticated attackers maintain multiple access paths - removing one may cause them to activate backup persistence mechanisms
- Coordinate containment actions to execute simultaneously across all known compromised assets
- Consider what the attacker can observe: network traffic changes, authentication failures, process termination

**Attacker Cleanup and Hiding:**

- Attackers who detect investigation activity may clear logs, destroy artifacts, and remove their tools
- They may deploy additional backdoors as insurance against discovered access being revoked
- Some attackers operate "low and slow" - maintaining minimal activity to avoid detection while retaining access

**Eradication Steps:**

1. Identify all indicators of compromise (IOCs) from the investigation
2. Scan the entire environment for the identified IOCs, not just known-affected systems
3. Remove malware, unauthorized accounts, persistence mechanisms, and backdoors
4. Patch the exploited vulnerability across all vulnerable systems
5. Reset credentials for all accounts with any level of access to compromised systems
6. Verify eradication through targeted scanning and monitoring

!!! info "External Resources"
    - [CISA Incident Response Playbooks](https://www.cisa.gov/sites/default/files/publications/Federal_Government_Cybersecurity_Incident_and_Vulnerability_Response_Playbooks_508C.pdf) (CISA)
    - [NIST SP 800-83 - Guide to Malware Incident Prevention and Handling](https://csrc.nist.gov/publications/detail/sp/800-83/rev-1/final) (NIST)
    - [Containment Strategies - SANS DFIR](https://www.sans.org/white-papers/36372/) (SANS Institute)

## Recovery & Lessons Learned

### Recovery

Recovery restores business operations to a normal state following an incident. The process must be methodical - hasty restoration without verification risks reintroducing the compromise.

**Recovery Process:**

- Rebuild affected systems from known-good images or verified backups taken before the compromise window
- Do not restore from backups that may contain the attacker's persistence mechanisms
- Restore services in priority order based on business criticality
- Apply all patches and hardening measures before returning systems to production
- Implement enhanced monitoring on recovered systems for a defined observation period (typically 30-90 days)
- Validate that all business functions operate correctly before declaring recovery complete

**Preventing Future Incidents:**

- Address root causes identified during investigation, not just symptoms
- Implement or improve detective controls that would catch the attack earlier in the kill chain
- Evaluate whether preventive controls (network segmentation, MFA, application whitelisting) would have blocked the attack path
- Update asset inventory and access control based on findings

### Lessons Learned

The lessons learned phase is the most critical and most frequently neglected phase of incident response. Without it, organizations are condemned to repeat the same failures.

**Post-Incident Review (Blameless Postmortem):**

- Conduct within 1-2 weeks while events are fresh, but after the immediate stress of response has subsided
- Include all participants in the response, plus stakeholders from affected business units
- Focus on systemic failures (process, tooling, visibility gaps) rather than individual mistakes
- Assume good intent - people made the best decisions they could with the information available at the time

**Root Cause Analysis Methods:**

- **5 Whys**: Iteratively ask "why" to trace from symptom to root cause
- **Fishbone (Ishikawa) diagram**: Categorize contributing factors (people, process, technology, environment)
- **Timeline analysis**: Walk through the chronological sequence of events to identify where the response could have been faster or more effective

**Deliverables:**

- Written incident report documenting timeline, root cause, impact, and response actions
- Action items with owners, deadlines, and tracking mechanisms
- Updated playbooks reflecting lessons from this incident
- Updated detection rules and monitoring coverage
- Training recommendations if skill gaps were identified

!!! info "External Resources"
    - [Etsy Blameless Postmortem Culture](https://www.etsy.com/codeascraft/blameless-postmortems/) (Etsy Engineering)
    - [Google SRE - Postmortem Culture](https://sre.google/sre-book/postmortem-culture/) (Google)
    - [NIST Lessons Learned in Incident Response](https://csrc.nist.gov/publications/detail/sp/800-61/rev-2/final) (NIST)

## Metrics & Prioritization

Metrics enable objective assessment of incident response effectiveness and guide resource allocation. Without measurement, improvement is anecdotal.

**Key Incident Response Metrics:**

| Metric | Definition | Target |
|:---|:---|:---|
| MTTD (Mean Time to Detect) | Time from incident occurrence to detection | Minimize |
| MTTR (Mean Time to Respond) | Time from detection to containment | Minimize |
| MTTE (Mean Time to Eradicate) | Time from containment to full eradication | Minimize |
| Dwell time | Total time attacker was present before detection | Minimize |
| False positive rate | Percentage of alerts that are not true incidents | Reduce to improve analyst efficiency |
| Incidents per analyst | Workload distribution metric | Balance to prevent burnout |
| Playbook coverage | Percentage of incident types with documented playbooks | Maximize |

**Prioritization Framework:**

Incident priority is determined by the intersection of impact and urgency:

- **Impact**: Number of affected users/systems, sensitivity of compromised data, business criticality of affected services, regulatory implications
- **Urgency**: Whether the attack is active and ongoing, whether containment has been achieved, time sensitivity of evidence collection

**Playbooks:**

- Pre-written, step-by-step response procedures for common incident types
- Cover alert triage, initial investigation steps, containment actions, escalation criteria, and communication templates
- Must be regularly tested through tabletop exercises and updated based on lessons learned
- Common playbook categories: phishing, malware infection, unauthorized access, data exfiltration, DDoS, insider threat, ransomware

**Alert Classification:**

- **True positive**: Legitimate security event requiring response
- **False positive**: Alert triggered by benign activity - requires tuning
- **True negative**: No alert when no event occurred - correct behavior
- **False negative**: No alert when a real event occurred - detection gap requiring immediate remediation

!!! info "External Resources"
    - [SANS Incident Response Metrics](https://www.sans.org/white-papers/36337/) (SANS Institute)
    - [VERIS Framework for Incident Classification](http://veriscommunity.net/) (Verizon VERIS)
    - [Incident Response Playbook Best Practices](https://www.cisa.gov/sites/default/files/publications/Federal_Government_Cybersecurity_Incident_and_Vulnerability_Response_Playbooks_508C.pdf) (CISA)
