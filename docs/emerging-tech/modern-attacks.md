# Modern Attacks

## Supply Chain Attacks

Supply chain attacks compromise a trusted upstream component to reach downstream targets. The attacker gains access to many organisations through a single point of compromise.

**Notable supply chain attacks:**

| Incident | Year | Attack vector | Impact |
|----------|------|--------------|--------|
| **SolarWinds (SUNBURST)** | 2020 | Trojanised build of Orion IT monitoring platform | ~18,000 organisations received backdoored update; multiple US government agencies compromised |
| **Codecov** | 2021 | Modified Bash Uploader script in CI/CD pipeline | Exfiltrated environment variables (secrets, tokens) from thousands of CI builds |
| **Log4Shell (Log4j)** | 2021 | RCE via JNDI lookup in ubiquitous logging library (CVE-2021-44228) | Affected virtually every Java application using Log4j 2.x; mass exploitation within hours |
| **Kaseya VSA** | 2021 | Zero-day in managed service provider (MSP) software | REvil ransomware deployed to ~1,500 downstream businesses |
| **3CX** | 2023 | Compromised desktop application distributed via official channels | Supply-chain-within-a-supply-chain: prior Trading Technologies compromise led to 3CX compromise |
| **xz Utils** | 2024 | Social engineering of open-source maintainer; backdoor in compression library | Near-miss: backdoor in sshd dependency caught before widespread deployment |
| **polyfill.io** | 2024 | Domain/CDN acquisition; injected malicious JavaScript via widely-used polyfill service | Affected 100K+ websites loading scripts from the compromised CDN |

**Supply chain attack categories:**

| Category | Description | Example |
|----------|-------------|---------|
| **Build system compromise** | Inject malicious code during CI/CD build process | SolarWinds - build pipeline modified to inject SUNBURST |
| **Dependency confusion** | Upload malicious package with same name as internal package to public registry | Dependency confusion attacks against npm, PyPI |
| **Typosquatting** | Publish malicious packages with names similar to popular ones | `crossenv` vs `cross-env` on npm |
| **Compromised maintainer** | Gain control of legitimate package through social engineering or account takeover | xz Utils, event-stream npm package |
| **CDN/distribution hijack** | Compromise the delivery mechanism rather than the source | polyfill.io CDN takeover |
| **Hardware/firmware implant** | Insert backdoors at the hardware or firmware level | Theoretical supply chain implants in network equipment |

**Defences:**

- Generate and verify Software Bill of Materials (SBOM) - SPDX or CycloneDX formats
- Pin dependencies to specific versions and verify checksums/signatures
- Use package lock files and audit dependencies regularly
- Implement binary authorisation and code signing (Sigstore/Cosign)
- Monitor for unexpected dependency changes in CI/CD
- Apply SLSA (Supply chain Levels for Software Artifacts) framework
- Segment build environments with least-privilege access
- Review and limit CDN/third-party script inclusions (use SRI - Subresource Integrity)

!!! info "External Resources"
    - [SLSA Framework](https://slsa.dev/) (OpenSSF)
    - [NIST SP 800-161 Rev 1 - C-SCRM](https://csrc.nist.gov/publications/detail/sp/800-161/rev-1/final) (NIST)
    - [OpenSSF Scorecard](https://securityscorecards.dev/) (OpenSSF)

---

## Cloud-Native Attacks

Cloud-native attacks exploit the unique properties of cloud infrastructure - APIs, metadata services, identity federation, and ephemeral compute.

**Common cloud-native attack patterns:**

| Attack | Technique | Impact |
|--------|-----------|--------|
| **IMDS exploitation** | Query instance metadata service (169.254.169.254) via SSRF | Steal IAM credentials, access cloud APIs |
| **Credential harvesting from CI/CD** | Extract environment variables or secrets from build logs | Lateral movement across cloud accounts |
| **Privilege escalation via IAM misconfiguration** | Exploit overly permissive IAM policies (iam:PassRole, sts:AssumeRole) | Full account takeover |
| **Storage bucket exposure** | Misconfigured S3/GCS/Blob ACLs or policies | Data exfiltration at scale |
| **Container escape** | Exploit kernel vulnerability or misconfigured capabilities | Host-level access from within container |
| **Serverless event injection** | Inject malicious payloads via event sources (S3, SNS, API Gateway) | Code execution in Lambda/Cloud Functions |
| **Cross-account role assumption** | Exploit trust relationships between accounts | Pivot between cloud accounts |
| **Resource hijacking** | Compromise cloud resources for cryptomining | Financial impact, resource exhaustion |

**IMDS v1 vs v2 (AWS):**

| Property | IMDSv1 | IMDSv2 |
|----------|--------|--------|
| **Request method** | Simple GET request | Requires PUT to get session token, then GET with token header |
| **SSRF vulnerability** | Easily exploitable via SSRF | PUT + custom header requirement blocks most SSRF attacks |
| **Best practice** | Disable where possible | Enforce IMDSv2-only via instance metadata options |

**Cloud-specific defences:**

- Enforce IMDSv2 and limit metadata service access
- Apply least-privilege IAM with service control policies (SCPs) as guardrails
- Enable CloudTrail/Cloud Audit Logs with alerting on sensitive API calls
- Use VPC endpoints to keep traffic off the public internet
- Implement CSPM (Cloud Security Posture Management) tools
- Enforce encryption at rest and in transit for all storage services
- Use short-lived credentials and avoid long-lived access keys

---

## API Abuse and Business Logic Attacks

API attacks target the logic and access controls of APIs rather than traditional web vulnerabilities.

**OWASP API Security Top 10 (2023):**

| # | Risk | Description |
|---|------|-------------|
| 1 | **Broken Object Level Authorisation (BOLA)** | API endpoints expose object IDs; attacker manipulates IDs to access other users' data |
| 2 | **Broken Authentication** | Weak or missing authentication mechanisms in APIs |
| 3 | **Broken Object Property Level Authorisation** | Excessive data exposure or mass assignment via API properties |
| 4 | **Unrestricted Resource Consumption** | No rate limiting; allows DoS or brute force |
| 5 | **Broken Function Level Authorisation** | Admin functions accessible to regular users |
| 6 | **Unrestricted Access to Sensitive Business Flows** | Automated abuse of business features (scalping, credential stuffing) |
| 7 | **Server-Side Request Forgery (SSRF)** | API fetches user-supplied URLs without validation |
| 8 | **Security Misconfiguration** | Missing security headers, verbose errors, unnecessary HTTP methods |
| 9 | **Improper Inventory Management** | Exposed old/debug/unpatched API versions |
| 10 | **Unsafe Consumption of APIs** | Trusting third-party API responses without validation |

**Business logic attack examples:**

- Price manipulation by modifying cart values in API requests
- Race conditions in payment processing (double-spend)
- Coupon/reward abuse through replay or parameter tampering
- Account enumeration via differential response timing or messages
- Bypassing workflow steps by calling APIs out of sequence

---

## AI/ML Security Risks

**Threat categories:**

| Category | Description | Example |
|----------|-------------|---------|
| **Adversarial inputs** | Crafted inputs that cause misclassification | Adversarial patches on stop signs fool object detection |
| **Data poisoning** | Corrupting training data to influence model behaviour | Injecting biased samples to shift model output |
| **Model theft/extraction** | Querying model API to reconstruct the model | Using prediction APIs to clone proprietary models |
| **Prompt injection** | Manipulating LLM behaviour via crafted prompts | Overriding system instructions to exfiltrate data |
| **Training data extraction** | Extracting memorised training data from models | Recovering PII or secrets embedded in training data |
| **Model supply chain** | Distributing backdoored pre-trained models | Trojanised models on public model registries |

**Defences:**

- Input validation and anomaly detection on model inputs
- Adversarial training and robustness testing
- Data provenance tracking and training data validation
- Rate limiting and monitoring of model API usage
- Output filtering and guardrails for generative AI
- Model watermarking and fingerprinting

!!! info "External Resources"
    - [OWASP API Security Top 10](https://owasp.org/API-Security/) (OWASP)
    - [MITRE ATLAS (Adversarial Threat Landscape for AI)](https://atlas.mitre.org/) (MITRE)
    - [NIST AI Risk Management Framework](https://www.nist.gov/artificial-intelligence/ai-risk-management-framework) (NIST)

---

## Ransomware Evolution

Modern ransomware has evolved from simple file encryption to multi-layered extortion operations.

**Ransomware evolution timeline:**

| Generation | Characteristics |
|------------|----------------|
| **Early (pre-2016)** | Simple encryption, fixed Bitcoin ransom, spray-and-pray distribution |
| **Big Game Hunting (2016-2019)** | Targeted attacks on large organisations, manual lateral movement, higher ransoms |
| **Double extortion (2019+)** | Data exfiltration before encryption; threaten to publish stolen data |
| **Triple extortion (2020+)** | Add DDoS threats or contact victims' customers/partners directly |
| **Ransomware-as-a-Service (RaaS)** | Developers provide platform; affiliates conduct attacks for revenue share |
| **Intermittent encryption (2022+)** | Encrypt portions of files to speed up encryption and evade detection |

**Modern ransomware tactics (mapped to MITRE ATT&CK):**

| Phase | Techniques |
|-------|-----------|
| **Initial access** | Phishing, RDP brute force, VPN vulnerability exploitation, access broker purchase |
| **Execution** | PowerShell, WMI, legitimate admin tools (LOLBins) |
| **Persistence** | Scheduled tasks, registry run keys, service creation |
| **Privilege escalation** | Token impersonation, exploiting unpatched local vulns, credential dumping |
| **Defence evasion** | Disable AV/EDR, clear event logs, use signed binaries |
| **Lateral movement** | PsExec, RDP, WMI, SMB, Group Policy modification |
| **Exfiltration** | Rclone to cloud storage, custom exfil tools, encrypted channels |
| **Impact** | Volume shadow copy deletion, file encryption, ransom note deployment |

**Defences:**

- Offline, tested backups with immutable storage (3-2-1 rule)
- Network segmentation limiting lateral movement
- EDR with behavioural detection (not just signature-based)
- Restrict RDP access and enforce MFA everywhere
- Patch management with focus on known exploited vulnerabilities (CISA KEV)
- Disable macros and restrict PowerShell via AppLocker/WDAC
- Incident response playbook specific to ransomware

!!! info "External Resources"
    - [CISA Stop Ransomware](https://www.cisa.gov/stopransomware) (CISA)
    - [MITRE ATT&CK - Ransomware](https://attack.mitre.org/software/) (MITRE)
    - [No More Ransom Project](https://www.nomoreransom.org/) (Europol)
