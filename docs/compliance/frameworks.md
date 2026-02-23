# Regulatory Frameworks

## Framework Comparison Matrix

| Framework | Scope | Mandatory? | Certification/Audit | Primary Focus |
|-----------|-------|------------|---------------------|---------------|
| **SOC 2** | Service organisations handling customer data | Voluntary (but often contractually required) | Type I (point-in-time) or Type II (over a period) | Trust Service Criteria: security, availability, processing integrity, confidentiality, privacy |
| **ISO 27001** | Any organisation, any industry | Voluntary | Third-party certification audit (3-year cycle) | Information Security Management System (ISMS) |
| **PCI DSS** | Any entity that stores, processes, or transmits cardholder data | Mandatory for card processors | Self-Assessment Questionnaire (SAQ) or QSA audit | Cardholder data protection |
| **GDPR** | Organisations processing data of EU residents | Mandatory (EU law) | No formal certification; supervisory authority enforcement | Data protection and privacy rights |
| **HIPAA** | Covered entities and business associates in US healthcare | Mandatory (US law) | OCR audits and breach investigations | Protected health information (PHI) |
| **NIST CSF** | US critical infrastructure; widely adopted globally | Voluntary (mandatory for US federal agencies via EO) | Self-assessment; no formal certification | Risk-based cybersecurity program |
| **FedRAMP** | Cloud service providers serving US federal agencies | Mandatory for federal cloud services | Third-party assessment organisation (3PAO) audit | Cloud service security authorisation |

---

## SOC 2

SOC 2 reports are issued by independent auditors under the AICPA Trust Services Criteria. They evaluate controls relevant to one or more of five categories: Security (required), Availability, Processing Integrity, Confidentiality, and Privacy.

**Type I vs Type II:**

| Property | Type I | Type II |
|----------|--------|---------|
| **Assessment scope** | Design of controls at a point in time | Design and operating effectiveness over a period (typically 6-12 months) |
| **Evidence required** | Control descriptions, walkthroughs | Control descriptions + evidence of consistent operation (logs, samples, tickets) |
| **Audit duration** | Shorter (weeks) | Longer (months of observation) |
| **Customer confidence** | "Controls exist" | "Controls exist and work reliably" |
| **Common first step** | Yes - often done before Type II | Follows Type I or done directly by mature organisations |

**Trust Service Criteria:**

| Criterion | Focus | Example Controls |
|-----------|-------|-----------------|
| **Security** (Common Criteria - CC) | Protection against unauthorised access | Firewalls, MFA, access reviews, vulnerability management |
| **Availability** | System uptime and performance | Disaster recovery, capacity planning, monitoring |
| **Processing Integrity** | Accurate and complete processing | Input validation, reconciliation, QA procedures |
| **Confidentiality** | Protection of confidential information | Encryption, data classification, access restrictions |
| **Privacy** | Personal information lifecycle management | Consent management, data retention, subject access requests |

!!! info "External Resources"
    - [AICPA SOC 2 Overview](https://www.aicpa-cima.com/topic/audit-assurance/audit-and-assurance-greater-than-soc-2) (AICPA)
    - [SOC 2 Trust Services Criteria](https://us.aicpa.org/interestareas/frc/assuranceadvisoryservices/trustservicescriteria) (AICPA)

---

## ISO 27001 / 27002

ISO 27001 specifies requirements for establishing, implementing, maintaining, and continually improving an Information Security Management System (ISMS). ISO 27002 provides a reference set of controls and implementation guidance.

**Certification process:**

1. **Gap analysis** - compare current state against ISO 27001 requirements
2. **Risk assessment** - identify information assets, threats, vulnerabilities; calculate risk
3. **Statement of Applicability (SoA)** - declare which Annex A controls apply and justify exclusions
4. **Implement controls** - deploy technical and organisational measures
5. **Internal audit** - verify ISMS effectiveness before external audit
6. **Stage 1 audit** - auditor reviews documentation, SoA, risk assessment
7. **Stage 2 audit** - auditor verifies controls are implemented and effective
8. **Certification** - valid for 3 years with annual surveillance audits
9. **Recertification** - full audit every 3 years

**ISO 27001 Annex A control families (2022 revision):**

| Category | Controls | Examples |
|----------|----------|---------|
| **Organisational** (37 controls) | Policies, roles, threat intelligence, asset management | Information security policy, asset inventory, supplier security |
| **People** (8 controls) | HR security, awareness, remote work | Background checks, security training, disciplinary process |
| **Physical** (14 controls) | Physical perimeter, equipment, utilities | Secure areas, equipment maintenance, clear desk policy |
| **Technological** (34 controls) | Technical security measures | Access control, cryptography, logging, network security, secure development |

!!! info "External Resources"
    - [ISO 27001 Standard](https://www.iso.org/isoiec-27001-information-security.html) (ISO)
    - [ISO 27002:2022 Controls](https://www.iso.org/standard/75652.html) (ISO)

---

## PCI DSS

The Payment Card Industry Data Security Standard applies to all entities involved in payment card processing. PCI DSS v4.0 defines 12 core requirements grouped into six goals.

**Requirements:**

| Goal | Requirement | Description |
|------|-------------|-------------|
| **Build and maintain a secure network** | 1 | Install and maintain network security controls |
| | 2 | Apply secure configurations to all system components |
| **Protect account data** | 3 | Protect stored account data (encryption, masking, hashing) |
| | 4 | Protect cardholder data with strong cryptography during transmission |
| **Maintain a vulnerability management program** | 5 | Protect all systems against malware |
| | 6 | Develop and maintain secure systems and software |
| **Implement strong access control** | 7 | Restrict access by business need-to-know |
| | 8 | Identify users and authenticate access |
| | 9 | Restrict physical access to cardholder data |
| **Monitor and test networks** | 10 | Log and monitor all access to system components and cardholder data |
| | 11 | Test security of systems and networks regularly |
| **Maintain an information security policy** | 12 | Support information security with organisational policies and programs |

**Validation levels (merchants):**

| Level | Transaction volume | Validation requirement |
|-------|-------------------|----------------------|
| 1 | >6 million/year | Annual on-site audit by QSA + quarterly network scan by ASV |
| 2 | 1-6 million/year | Annual SAQ + quarterly ASV scan |
| 3 | 20K-1 million/year (e-commerce) | Annual SAQ + quarterly ASV scan |
| 4 | <20K (e-commerce) or <1 million (other) | Annual SAQ recommended; quarterly ASV scan if applicable |

**Key PCI DSS v4.0 changes:**

- Customised approach as alternative to defined approach for meeting requirements
- Stronger multi-factor authentication requirements (Requirement 8)
- Enhanced security awareness training
- Targeted risk analysis for each requirement
- New e-commerce and phishing protections

!!! info "External Resources"
    - [PCI DSS v4.0 Standard](https://www.pcisecuritystandards.org/document_library/) (PCI SSC)
    - [PCI DSS Quick Reference Guide](https://www.pcisecuritystandards.org/pdfs/pci_ssc_quick_guide.pdf) (PCI SSC)

---

## GDPR

The General Data Protection Regulation governs the processing of personal data of individuals in the European Union.

**Core principles (Article 5):**

| Principle | Meaning |
|-----------|---------|
| **Lawfulness, fairness, transparency** | Process data with a legal basis and inform data subjects |
| **Purpose limitation** | Collect data for specified, explicit, and legitimate purposes only |
| **Data minimisation** | Collect only what is necessary for the stated purpose |
| **Accuracy** | Keep personal data accurate and up to date |
| **Storage limitation** | Retain data only as long as necessary |
| **Integrity and confidentiality** | Protect data with appropriate technical and organisational measures |
| **Accountability** | The controller must demonstrate compliance |

**Legal bases for processing (Article 6):**

1. Consent
2. Contract performance
3. Legal obligation
4. Vital interests
5. Public interest / official authority
6. Legitimate interests (requires balancing test)

**Data subject rights:**

- Right of access (Article 15)
- Right to rectification (Article 16)
- Right to erasure / "right to be forgotten" (Article 17)
- Right to restriction of processing (Article 18)
- Right to data portability (Article 20)
- Right to object (Article 21)
- Rights related to automated decision-making (Article 22)

**Breach notification:**

- **Supervisory authority**: within 72 hours of becoming aware (Article 33)
- **Data subjects**: without undue delay when breach is likely to result in high risk to rights and freedoms (Article 34)

**Penalties**: up to 20 million EUR or 4% of annual global turnover, whichever is higher.

**Key roles:**

| Role | Responsibility |
|------|---------------|
| **Data Controller** | Determines purposes and means of processing |
| **Data Processor** | Processes data on behalf of the controller |
| **Data Protection Officer (DPO)** | Mandatory for public authorities and large-scale processing; advises on compliance |

!!! info "External Resources"
    - [GDPR Full Text](https://gdpr-info.eu/) (gdpr-info.eu)
    - [European Data Protection Board](https://edpb.europa.eu/) (EDPB)

---

## HIPAA

The Health Insurance Portability and Accountability Act protects the privacy and security of Protected Health Information (PHI) in the United States.

**Key rules:**

| Rule | Focus |
|------|-------|
| **Privacy Rule** | Sets standards for who can access and disclose PHI |
| **Security Rule** | Requires administrative, physical, and technical safeguards for electronic PHI (ePHI) |
| **Breach Notification Rule** | Requires notification to individuals, HHS, and media (for large breaches) |
| **Enforcement Rule** | Defines investigation and penalty procedures |

**Security Rule safeguards:**

| Category | Examples |
|----------|---------|
| **Administrative** | Risk analysis, workforce training, security management process, contingency plan, business associate agreements |
| **Physical** | Facility access controls, workstation security, device and media controls |
| **Technical** | Access control, audit controls, integrity controls, transmission security (encryption) |

**Covered entities vs business associates:**

- **Covered entities**: health plans, healthcare clearinghouses, healthcare providers conducting electronic transactions
- **Business associates**: third parties that create, receive, maintain, or transmit PHI on behalf of a covered entity (requires BAA - Business Associate Agreement)

---

## NIST Cybersecurity Framework (CSF)

NIST CSF provides a voluntary framework for managing cybersecurity risk. Version 2.0 (2024) added a sixth core function: Govern.

**Core functions:**

| Function | Purpose | Example Categories |
|----------|---------|-------------------|
| **Govern (GV)** | Establish and monitor cybersecurity risk management strategy, expectations, and policy | Organisational context, risk management strategy, roles and responsibilities, policy, oversight |
| **Identify (ID)** | Understand assets, risks, and the business context | Asset management, risk assessment, supply chain risk management |
| **Protect (PR)** | Implement safeguards to ensure service delivery | Identity management, data security, platform security, technology resilience |
| **Detect (DE)** | Discover cybersecurity events | Continuous monitoring, adverse event analysis |
| **Respond (RS)** | Take action on detected events | Incident management, analysis, mitigation, reporting |
| **Recover (RC)** | Restore capabilities after an incident | Recovery planning, communications |

**Implementation tiers:**

| Tier | Name | Characteristics |
|------|------|----------------|
| 1 | Partial | Ad-hoc, reactive, limited awareness of cybersecurity risk |
| 2 | Risk Informed | Risk management approved by management but not org-wide policy |
| 3 | Repeatable | Formally approved policies, regularly updated based on risk |
| 4 | Adaptive | Continuous improvement, lessons learned feed back into program |

!!! info "External Resources"
    - [NIST CSF 2.0](https://www.nist.gov/cyberframework) (NIST)
    - [NIST SP 800-53 Security Controls](https://csrc.nist.gov/publications/detail/sp/800-53/rev-5/final) (NIST)

---

## FedRAMP

The Federal Risk and Authorization Management Program provides a standardised approach to security assessment, authorisation, and continuous monitoring for cloud products and services used by US federal agencies.

**Authorisation paths:**

| Path | Process |
|------|---------|
| **Agency Authorisation** | Cloud provider works with a specific agency; agency grants ATO (Authority to Operate) |
| **Joint Authorisation Board (JAB)** | Provisional ATO (P-ATO) reviewed by JAB (DoD, DHS, GSA); reusable across agencies |

**Impact levels (based on FIPS 199):**

| Level | Suitable for | Control baseline |
|-------|-------------|-----------------|
| **Low** | Publicly available data; low confidentiality/integrity/availability impact | ~156 controls |
| **Moderate** | Most government data; serious adverse effects if compromised | ~325 controls |
| **High** | Law enforcement, emergency services, financial, health; severe/catastrophic effects | ~421 controls |

**Continuous monitoring requirements:**

- Monthly vulnerability scanning
- Annual penetration testing
- Annual security assessment
- Significant change requests reviewed and approved
- Continuous monitoring of Plan of Action and Milestones (POA&M)

!!! info "External Resources"
    - [FedRAMP Official Site](https://www.fedramp.gov/) (GSA)
    - [FedRAMP Marketplace](https://marketplace.fedramp.gov/) (GSA)
