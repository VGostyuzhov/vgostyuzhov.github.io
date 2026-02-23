# Plan: Create All Missing Documentation

## Context

The site has 71 markdown files total. 41 are completely empty stubs, 2 are thin (functional but minimal), and 28 are populated. Eight entire sections have zero content. The `notes.md` file (734 lines) contains detailed topic outlines from the original "Security Engineering at Google" study notes that map directly to the empty sections and serve as a content blueprint. The `interview-prep/questions.md` reveals additional topics (cloud IAM, secrets management, detection strategy) that should be covered.

---

## Step 1: Audit - Missing Topics and New Files

### Current empty stubs (41 files)

| Section | Empty files |
|---------|-------------|
| `compliance/` | `index.md`, `frameworks.md`, `governance.md` |
| `crypto-identity/` | `fundamentals.md`, `identity.md` |
| `detection-monitoring/` | `index.md`, `detection-systems.md`, `monitoring.md` |
| `emerging-tech/` | `index.md`, `devsecops.md`, `modern-attacks.md`, `zero-trust.md` |
| `forensics/` | `index.md`, `fundamentals.md`, `disciplines.md` |
| `incident-response/` | `index.md`, `management.md` |
| `infrastructure-cloud/` | `index.md`, `cloud-security.md`, `containers.md`, `virtualization.md` |
| `malware-threats/` | `index.md`, `analysis.md`, `attack-vectors.md`, `lifecycle.md` |
| `networking/` | `core-networking.md`, `protocols.md`, `security-tools.md` |
| `programming/` | `index.md`, `fundamentals.md`, `challenges.md` |
| `threat-modeling/` | `index.md`, `frameworks.md`, `risk-management.md` |
| `web-security/` | `index.md`, `tools.md`, `vulnerabilities.md` |

### Thin files needing expansion (2 files)

- `networking/index.md` (8 lines - just a title and 4 bullet links)
- `crypto-identity/index.md` (14 lines - brief intro, no structure like linux/index.md)

### Suggested new files (topics missing from current structure)

Based on `notes.md` content and `questions.md` topics that have no home:

| New file | Rationale | Nav placement |
|----------|-----------|---------------|
| `infrastructure-cloud/iam.md` | Cloud IAM is a major interview topic (questions 11-12, 14-16). Currently no dedicated page. notes.md covers service accounts, lateral movement, privilege escalation in cloud. | Under "Infrastructure & Cloud", after "Cloud Security" |
| `infrastructure-cloud/secrets-management.md` | Secrets management appears in questions 15, 24. No current page. Critical for cloud security interviews. | Under "Infrastructure & Cloud", after "IAM" |
| `networking/attacks.md` | notes.md has substantial content on network attacks (ARP spoofing, DNS exfil, PitM, CAM overflow). No current page covers network-layer attacks specifically. `security-tools.md` is about tools, not attacks. | Under "Networking & Protocols", after "Security Tools" |

### Files NOT to add

- **Windows security** - notes.md mentions AD/Kerberos/SMB but the site is scoped to "Infrastructure & Cloud Security", not Windows administration. Windows topics are covered where relevant (e.g., Kerberos in `authentication/directory-services.md`).
- **Dedicated OWASP top 10 page** - already covered across `web-security/fundamentals.md` and `web-security/vulnerabilities.md`.

### Action items for Step 1

1. Create 3 new empty files: `infrastructure-cloud/iam.md`, `infrastructure-cloud/secrets-management.md`, `networking/attacks.md`
2. Update `mkdocs.yml` nav to include the 3 new files
3. Run `mkdocs build` to verify

---

## Step 2: Review and Improve File Organization

### Current issues

1. **`web-security/fundamentals.md` is 691 lines** - It contains Browser Security Model, CORS & CSP, Secure Cookies & Storage, Common Web Vulnerabilities, XSS Variants, Injection Flaws, SSRF & Deserialization, API & Microservice Security, Secure Development Practices, Dependency Management, Code Review & Linters. This is 12+ topics crammed into one file.

2. **Content in `fundamentals.md` belongs in other empty pages:**
   - Common Web Vulnerabilities, XSS Variants, Injection Flaws, SSRF & Deserialization -> `web-security/vulnerabilities.md`
   - API & Microservice Security (REST vs GraphQL, AuthN/AuthZ Patterns, Rate Limiting) -> should stay in `fundamentals.md` OR move to `web-security/tools.md` (tools is misnamed)
   - SDLC & Threat Modeling, Dependency Management, Code Review -> `emerging-tech/devsecops.md` (better fit than web-security)

3. **`web-security/tools.md` is misnamed** - The nav calls it "Web Technologies & Tools" but the actual populated content in `fundamentals.md` about API security, rate limiting patterns, and development practices would fit better here. Rename to reflect actual content scope: web application architecture and tooling.

### Proposed reorganization

**Move content out of `web-security/fundamentals.md`:**

| Content block (current H2/H3 in fundamentals.md) | Move to | Rationale |
|--------------------------------------------------|---------|-----------|
| Common Web Vulnerabilities (H2) + XSS Variants + Injection Flaws + SSRF & Deserialization | `web-security/vulnerabilities.md` | That's literally what the file is for |
| API & Microservice Security (H2) + REST vs GraphQL + AuthN & AuthZ Patterns + Rate Limiting | `web-security/tools.md` | API architecture and security patterns |
| Secure Development Practices (H2) + SDLC & Threat Modeling + Dependency Management + Code Review & Linters | `emerging-tech/devsecops.md` | DevSecOps is the right home |

**Keep in `web-security/fundamentals.md`:**
- Browser Security Model (Same-Origin Policy)
- CORS & CSP
- Secure Cookies & Storage

These are genuinely "web security fundamentals" - the browser security model.

**Thin index pages** - expand `networking/index.md` and `crypto-identity/index.md` to match the pattern in `linux/index.md` and `authentication/index.md` (structured topic list with learning objectives).

### Action items for Step 2

1. Move vulnerability content from `fundamentals.md` to `vulnerabilities.md`
2. Move API/microservice content from `fundamentals.md` to `tools.md`
3. Move SDLC/dependency/code-review content from `fundamentals.md` to `devsecops.md`
4. Keep browser security model, CORS/CSP, cookies in `fundamentals.md`
5. Expand `networking/index.md` and `crypto-identity/index.md`
6. Run `mkdocs build` to verify no broken links

---

## Step 3: Create Table of Contents for Each Empty Page

Every empty page gets a structured TOC (H1 + H2 headings) based on `notes.md` content, `questions.md` topics, and standard security interview coverage. Content style recommendation noted for each.

### networking/

**`core-networking.md`** (concise style)
```
# Core Networking
## OSI Model & TCP/IP Stack
## IP Addressing & Subnetting
## TCP vs UDP
## Routing & Switching Fundamentals
## NAT & DHCP
## HTTP/HTTPS
## Common Service Ports
```

**`protocols.md`** (concise style)
```
# Network Protocols
## Transport Layer Protocols
## Application Layer Protocols
## Email Protocols (SMTP, IMAP, POP3)
## File Transfer Protocols (FTP, SFTP, SCP)
## Remote Access Protocols (SSH, Telnet, RDP)
## Directory & Authentication Protocols (LDAP, Kerberos, RADIUS)
## SSL/TLS
```

**`security-tools.md`** (concise style)
```
# Network Security & Tools
## Packet Capture & Analysis
## Network Scanning
## Traffic Monitoring
## VPN & Tunneling
## Proxy & Interception Tools
## Wireless Security Tools
```

**`attacks.md`** (detailed style) [NEW]
```
# Network Attacks
## Person-in-the-Middle (PitM) Attacks
## ARP Spoofing & Cache Poisoning
## DNS Attacks (Exfiltration, Poisoning, Hijacking)
## BGP Hijacking
## DDoS & Amplification Attacks
## Port Scanning & Reconnaissance
## CAM Table Overflow
```

**`networking/index.md`** - expand to match linux/index.md pattern with categorized links and learning objectives.

### web-security/

**`index.md`** (index page)
```
# Web Application Security
## Core Topics
## Key Learning Objectives
```

**`vulnerabilities.md`** (detailed style) - will receive moved content from fundamentals.md, no new TOC needed since content exists.

**`tools.md`** (detailed style) - will receive moved API/microservice content from fundamentals.md, no new TOC needed since content exists.

### infrastructure-cloud/

**`index.md`** (index page)
```
# Infrastructure & Cloud Security
## Core Topics
## Key Learning Objectives
```

**`virtualization.md`** (concise style)
```
# Virtualization & Cloud Fundamentals
## Hypervisors (Type 1 vs Type 2)
## Virtual Machine Security
## VM Escape & Side-Channel Attacks
## Cloud Service Models (IaaS, PaaS, SaaS)
## Shared Responsibility Model
## Cloud vs On-Prem Security Differences
```

**`cloud-security.md`** (detailed style)
```
# Cloud Security
## Cloud Networking & Segmentation
## Object Storage Security (S3, GCS, Blob)
## Cloud Logging & Monitoring
## Cloud Threat Modeling
## Serverless & PaaS Security
## Multi-Cloud & Hybrid Considerations
```

**`containers.md`** (detailed style)
```
# Container Security
## Container Architecture & Isolation
## Image Security & Supply Chain
## Container Runtime Security
## Kubernetes Security Fundamentals
## Network Policies & Service Mesh
## Container Escape Techniques
## Container Hardening Best Practices
```

**`iam.md`** (detailed style) [NEW]
```
# Cloud Identity & Access Management
## IAM Fundamentals & Principles
## Common IAM Misconfigurations
## Service Accounts & Workload Identity
## Role Design & Least Privilege
## Cross-Account & Federation
## IAM for Microservices
## Privilege Escalation in Cloud
```

**`secrets-management.md`** (concise style) [NEW]
```
# Secrets Management
## Why Secrets Management Matters
## Secrets Management Tools (Vault, AWS SM, GCP SM)
## Secret Rotation & Lifecycle
## Secrets in CI/CD Pipelines
## Application-Level Secret Injection
## Emergency Revocation & Incident Response
```

### crypto-identity/

**`fundamentals.md`** (detailed style)
```
# Cryptographic Fundamentals
## Encryption vs Encoding vs Hashing vs Signing
## Symmetric Encryption (AES, ChaCha20)
## Asymmetric Encryption (RSA, ECC/Ed25519)
## Block vs Stream Ciphers
## Hashing Functions (SHA-256, BLAKE, MD5)
## Message Authentication Codes (MAC, HMAC)
## Digital Signatures
## Key Exchange & Perfect Forward Secrecy
## Entropy & Random Number Generation
## Common Cryptographic Attacks
```

**`identity.md`** (detailed style)
```
# Identity Management
## Identity Lifecycle (Provisioning, Management, Deprovisioning)
## Access Control Models (RBAC, ABAC, ReBAC)
## Service Accounts vs User Accounts
## Privileged Access Management (PAM)
## Federated Identity
## Identity Governance & Compliance
```

**`crypto-identity/index.md`** - expand to match linux/index.md pattern.

### malware-threats/

**`index.md`** (index page)
```
# Malware & Threats
## Core Topics
## Key Learning Objectives
```

**`analysis.md`** (detailed style)
```
# Malware Analysis
## Static vs Dynamic Analysis
## Decompilation & Reverse Engineering Tools
## Sandboxing & Behavioral Analysis
## Malware Feature Analysis
## Evasion Techniques (Anti-sandbox, Polymorphism)
## Analysis Platforms & Resources
```

**`attack-vectors.md`** (detailed style)
```
# Attack Vectors
## Social Engineering (Phishing, Spear Phishing, Baiting)
## Physical Attack Vectors
## Network Attack Vectors
## Exploit Kits & Drive-by Downloads
## Supply Chain Attacks
## Remote Code Execution & Shells
## Spoofing Techniques (IP, MAC, Email, ARP, Biometric)
```

**`lifecycle.md`** (detailed style)
```
# Attack Lifecycle
## Reconnaissance & OSINT
## Resource Development
## Initial Access
## Execution & Persistence
## Privilege Escalation
## Defense Evasion
## Credential Access
## Lateral Movement & Discovery
## Collection & Exfiltration
## Command and Control (C2)
## Impact
```

### threat-modeling/

**`index.md`** (index page)
```
# Threat Modeling
## Core Topics
## Key Learning Objectives
```

**`frameworks.md`** (detailed style)
```
# Threat Modeling Frameworks
## STRIDE
## PASTA (Process for Attack Simulation and Threat Analysis)
## MITRE ATT&CK Framework
## Attack Trees
## DREAD (Deprecated but Referenced)
## Choosing a Framework
```

**`risk-management.md`** (detailed style)
```
# Risk Management
## Risk Assessment Fundamentals
## Risk Prioritization (Likelihood x Impact)
## Risk Treatment Options (Accept, Mitigate, Transfer, Avoid)
## Blast Radius & Containment
## Communicating Risk to Stakeholders
## Risk Registers & Tracking
```

### detection-monitoring/

**`index.md`** (index page)
```
# Detection & Monitoring
## Core Topics
## Key Learning Objectives
```

**`detection-systems.md`** (detailed style)
```
# Detection Systems
## Intrusion Detection Systems (IDS/IPS)
## Signature-Based vs Behavior-Based Detection
## Rule Writing (Snort, Suricata, YARA)
## Honeypots & Canary Tokens
## Indicators of Compromise (IOCs)
## Detection Strategy Design
```

**`monitoring.md`** (detailed style)
```
# Monitoring & Analysis
## SIEM Architecture & Tooling
## Log Sources & Collection
## Alert Triage & Fatigue Reduction
## Anomaly & Behavior-Based Detection
## Network Traffic Analysis
## Security Metrics & KPIs
```

### forensics/

**`index.md`** (index page)
```
# Digital Forensics
## Core Topics
## Key Learning Objectives
```

**`fundamentals.md`** (detailed style)
```
# Forensic Fundamentals
## Evidence Volatility & Order of Collection
## Chain of Custody
## Forensic Imaging & Preservation
## Anti-Forensics Techniques
## Legal Considerations
```

**`disciplines.md`** (concise style)
```
# Forensic Disciplines
## Network Forensics (DNS Logs, Netflow, Packet Capture)
## Disk Forensics (Imaging, Filesystems, Data Carving)
## Memory Forensics (Acquisition, Structures, Volatility)
## Mobile Forensics (Android vs iOS, Jailbreaking)
## Cloud Forensics
## Forensic Tools Reference
```

### incident-response/

**`index.md`** (index page)
```
# Incident Response
## Core Topics
## Key Learning Objectives
```

**`management.md`** (detailed style)
```
# Incident Management
## Incident Classification (Security vs Privacy)
## Response Models (PICERL, IMAG)
## Incident Roles & Communication
## Investigation Process
## Containment & Eradication
## Recovery & Lessons Learned
## Metrics & Prioritization
```

### compliance/

**`index.md`** (index page)
```
# Compliance & Governance
## Core Topics
## Key Learning Objectives
```

**`frameworks.md`** (concise style)
```
# Regulatory Frameworks
## SOC 2 (Type I vs Type II)
## ISO 27001/27002
## PCI DSS
## GDPR
## HIPAA
## NIST Cybersecurity Framework
## FedRAMP
## Framework Comparison Matrix
```

**`governance.md`** (detailed style)
```
# Security Governance
## Security Policies & Standards
## Security Organization Structure
## Audit & Assessment Programs
## Vendor & Third-Party Risk Management
## Security Awareness Training
## Governance Metrics & Reporting
```

### emerging-tech/

**`index.md`** (index page)
```
# Emerging Technologies & Practices
## Core Topics
## Key Learning Objectives
```

**`devsecops.md`** (detailed style) - will receive moved content from web-security/fundamentals.md (SDLC, Dependency Management, Code Review). Additional TOC:
```
# DevSecOps
## Secure SDLC & Threat Modeling (already written)
## Dependency Management (already written)
## Code Review & Linters (already written)
## Infrastructure as Code Security
## Container & Image Scanning in CI/CD
## Policy as Code (OPA, Sentinel)
```

**`modern-attacks.md`** (detailed style)
```
# Modern Attack Techniques
## Supply Chain Attacks (SolarWinds, Log4j, Codecov)
## Cloud-Native Attacks
## API Abuse & Business Logic Attacks
## AI/ML Security Risks
## Side-Channel Attacks (Spectre, Meltdown)
## Ransomware Evolution
```

**`zero-trust.md`** (detailed style)
```
# Zero Trust Architecture
## Zero Trust Principles
## BeyondCorp Model
## Identity-Centric Security
## Microsegmentation
## Continuous Verification
## Zero Trust Implementation Roadmap
```

### programming/

**`index.md`** (index page)
```
# Programming & Technical
## Core Topics
## Key Learning Objectives
```

**`fundamentals.md`** (concise style)
```
# Coding Fundamentals
## Data Structures for Security Engineers
## Algorithms & Complexity (Big O)
## Regular Expressions
## Python Essentials
## Scripting for Security Automation
```

**`challenges.md`** (concise style)
```
# Security Coding Challenges
## Cipher Implementation
## Log Parsing & Analysis
## Port Scanner
## Web Scraper
## Password Brute-Forcer
## Malware Signature Scanner
## Metadata Extraction
## Deleted File Recovery
```

---

## Step 4: Add Content

### Content sources and approach

1. **Moved content** (from web-security/fundamentals.md) - copy as-is, clean up formatting to match AGENT.md guidelines
2. **notes.md as blueprint** - use the topic outlines as a skeleton, expand each bullet into proper study guide content
3. **questions.md as depth guide** - topics that appear in interview questions get more detailed treatment
4. **New content** - written following AGENT.md style guidelines (professional, dry, technical, no emojis, no metaphors)

### Execution order

Content will be created in dependency order. Index pages first (they reference child pages), then content pages grouped by section.

**Batch 1: Reorganize existing content**
1. Move vulnerability content from `web-security/fundamentals.md` to `vulnerabilities.md`
2. Move API/microservice content from `web-security/fundamentals.md` to `tools.md`
3. Move SDLC/dependency/code-review content from `web-security/fundamentals.md` to `devsecops.md`
4. Create 3 new files, update `mkdocs.yml`
5. Expand `networking/index.md` and `crypto-identity/index.md`
6. Build and verify

**Batch 2: Index pages (10 empty section landing pages)**
- All section `index.md` files following the `linux/index.md` pattern
- Quick to write, unblocks navigation

**Batch 3: Networking section** (3 empty + 1 new)
- `core-networking.md`, `protocols.md`, `security-tools.md`, `attacks.md`
- notes.md has extensive networking content to draw from

**Batch 4: Infrastructure & Cloud section** (4 empty + 2 new)
- `virtualization.md`, `cloud-security.md`, `containers.md`, `iam.md`, `secrets-management.md`
- High interview relevance per questions.md

**Batch 5: Crypto & Identity** (2 empty)
- `fundamentals.md`, `identity.md`
- notes.md has detailed crypto/auth/identity outlines

**Batch 6: Threat Intelligence sections** (9 empty)
- `malware-threats/`: `analysis.md`, `attack-vectors.md`, `lifecycle.md`
- `threat-modeling/`: `frameworks.md`, `risk-management.md`
- `detection-monitoring/`: `detection-systems.md`, `monitoring.md`

**Batch 7: Response & Compliance** (5 empty)
- `forensics/`: `fundamentals.md`, `disciplines.md`
- `incident-response/`: `management.md`
- `compliance/`: `frameworks.md`, `governance.md`

**Batch 8: Remaining sections** (5 empty)
- `emerging-tech/`: `modern-attacks.md`, `zero-trust.md` (devsecops.md done in batch 1)
- `programming/`: `fundamentals.md`, `challenges.md`
- `web-security/index.md`

### Content rules per page

- Follow AGENT.md guidelines strictly
- Each H2 section gets an `!!! info "External Resources"` block (min 3 links)
- Use concise or detailed style as noted in Step 3
- Target ~50-150 lines per page (concise) or ~100-300 lines (detailed)
- No duplication of content already in other pages - cross-reference instead

---

## Verification

After each batch:
1. Run `mkdocs build` - verify no errors or broken links
2. Run `mkdocs serve` - spot-check rendered pages
3. Verify all new/moved pages appear correctly in navigation
4. Check cross-references between pages still work

Final verification:
- Confirm zero empty stubs remain
- Confirm all index pages have structured content
- Confirm `mkdocs build` is clean
- Count total content pages and confirm coverage
