# Security Governance

## Security Policies, Standards, and Procedures

Governance documents form a hierarchy that translates business intent into actionable security controls.

| Document type | Purpose | Audience | Update frequency | Example |
|---------------|---------|----------|-----------------|---------|
| **Policy** | High-level statement of intent and direction | All employees, executive leadership | Annual or on significant change | "All systems must enforce MFA for privileged access" |
| **Standard** | Specific mandatory requirements to implement a policy | Technical teams, security | Annual or on technology change | "Passwords must be minimum 14 characters; TOTP or FIDO2 required for admin accounts" |
| **Procedure** | Step-by-step instructions for carrying out a standard | Operators, administrators | As needed when tooling changes | "How to enroll a FIDO2 key in Okta for admin accounts" |
| **Guideline** | Recommended practices (not mandatory) | All staff | As needed | "Recommended browser security settings for remote workers" |
| **Baseline** | Minimum configuration for a technology | System administrators | Per release cycle | "CIS Benchmark Level 1 for Ubuntu 22.04" |

**Common security policies:**

- Acceptable Use Policy (AUP)
- Information Classification and Handling Policy
- Access Control Policy
- Incident Response Policy
- Data Retention and Disposal Policy
- Remote Work / BYOD Policy
- Encryption Policy
- Vendor and Third-Party Security Policy
- Change Management Policy
- Business Continuity and Disaster Recovery Policy

---

## Organisational Structure

**Key security roles:**

| Role | Responsibility | Reports to |
|------|---------------|------------|
| **CISO** | Overall security strategy, risk management, compliance | CEO, CTO, or Board |
| **Security Architect** | Design security controls and reference architectures | CISO or VP Engineering |
| **Security Engineer** | Implement and operate security controls, tooling | Security Architect or CISO |
| **Security Analyst (SOC)** | Monitor alerts, triage incidents, investigate threats | SOC Manager |
| **GRC Analyst** | Manage compliance programs, audit coordination, policy maintenance | CISO |
| **Application Security Engineer** | Secure SDLC, code review, SAST/DAST, developer training | CISO or VP Engineering |
| **Incident Responder** | Lead incident investigation and containment | SOC Manager or CISO |
| **DPO (Data Protection Officer)** | GDPR/privacy compliance, data subject requests | Legal or CISO (must be independent) |

**Governance committees:**

| Committee | Purpose | Typical members | Cadence |
|-----------|---------|----------------|---------|
| **Security Steering Committee** | Strategic direction, budget, risk acceptance | CISO, CTO, CFO, Legal, business leaders | Quarterly |
| **Change Advisory Board (CAB)** | Approve changes with security impact | CISO rep, IT ops, dev leads | Weekly or per change |
| **Risk Committee** | Review risk register, approve risk treatment | CISO, CRO, business unit leads | Monthly/Quarterly |
| **Incident Review Board** | Post-incident review, lessons learned | CISO, IR team, affected team leads | After major incidents |

---

## Audit and Assessment Programs

**Audit types:**

| Type | Conducted by | Purpose | Examples |
|------|-------------|---------|---------|
| **Internal audit** | Internal audit team or security team | Verify control effectiveness, identify gaps | Annual ISMS review, access review, configuration compliance |
| **External audit** | Third-party auditor | Independent assurance for stakeholders | SOC 2 Type II, ISO 27001 certification, PCI DSS QSA |
| **Regulatory audit** | Government/regulatory body | Verify legal compliance | GDPR supervisory authority, HIPAA OCR audit |
| **Penetration test** | Internal red team or external firm | Test security controls under simulated attack | Annual pentest, red team engagement, bug bounty |

**Evidence collection best practices:**

- Maintain a centralised evidence repository (GRC tool or structured file share)
- Automate evidence collection where possible (API pulls from SIEM, IAM, CMDB)
- Timestamp and version all evidence artifacts
- Map evidence to specific control requirements
- Retain evidence for the required period (typically audit period + 1 year minimum)
- Use screenshots with metadata (date, system, user) for manual processes

**Audit lifecycle:**

1. **Planning** - define scope, objectives, timeline, resource needs
2. **Fieldwork** - collect evidence, interview process owners, test controls
3. **Reporting** - document findings, classify severity, recommend remediation
4. **Remediation tracking** - owners address findings with deadlines
5. **Follow-up** - verify remediation effectiveness
6. **Continuous monitoring** - ongoing control monitoring between audits

---

## Vendor and Third-Party Risk Management

Third-party vendors can introduce significant risk. A vendor risk management (VRM) program assesses and monitors these risks throughout the vendor lifecycle.

**Vendor risk assessment process:**

| Phase | Activities |
|-------|-----------|
| **Due diligence** | Security questionnaire (SIG, CAIQ), review SOC 2/ISO 27001 reports, assess data handling practices |
| **Risk classification** | Tier vendors by data sensitivity and access level (critical, high, medium, low) |
| **Contractual controls** | BAA, DPA, SLA with security requirements, right-to-audit clauses, breach notification terms |
| **Onboarding** | Least-privilege access provisioning, network segmentation, monitoring setup |
| **Ongoing monitoring** | Periodic reassessment, continuous monitoring (security ratings), incident notification tracking |
| **Offboarding** | Access revocation, data return/destruction verification, contract termination checklist |

**Vendor tiering criteria:**

| Tier | Criteria | Assessment rigour |
|------|----------|-------------------|
| **Critical** | Processes sensitive data, has network access, or single point of failure | Full security assessment, on-site audit, quarterly reviews |
| **High** | Handles confidential data or has limited system access | Detailed questionnaire, SOC 2/ISO review, annual review |
| **Medium** | Limited data exposure, no direct system access | Standard questionnaire, certification review |
| **Low** | No data access, no system integration | Basic questionnaire or self-attestation |

**Key assessment frameworks:**

- **SIG (Standardised Information Gathering)** - comprehensive vendor assessment questionnaire by Shared Assessments
- **CAIQ (Consensus Assessments Initiative Questionnaire)** - cloud-specific, maintained by CSA
- **NIST SP 800-161** - supply chain risk management guidance

!!! info "External Resources"
    - [NIST SP 800-161 Rev 1 - C-SCRM](https://csrc.nist.gov/publications/detail/sp/800-161/rev-1/final) (NIST)
    - [Shared Assessments SIG](https://sharedassessments.org/sig/) (Shared Assessments)
    - [CSA CAIQ](https://cloudsecurityalliance.org/star/registry/) (CSA)

---

## Security Awareness Training

Effective security awareness reduces human-factor risk. Training should be role-based, measurable, and regularly refreshed.

**Training program components:**

| Component | Audience | Frequency | Content |
|-----------|----------|-----------|---------|
| **General awareness** | All employees | Annual + onboarding | Phishing, password hygiene, data handling, incident reporting |
| **Role-based training** | Technical staff | Annual | Secure coding, cloud security, incident response procedures |
| **Phishing simulations** | All employees | Monthly/Quarterly | Simulated phishing emails with tracking and follow-up training |
| **Executive briefings** | Leadership | Quarterly | Threat landscape, risk posture, compliance status |
| **Incident tabletop exercises** | IR team + business | Semi-annual | Simulated incident scenarios with decision-making practice |

**Measuring effectiveness:**

- Phishing simulation click rates (trending down over time)
- Time to report suspicious emails
- Training completion rates
- Reduction in security incidents caused by human error
- Post-training quiz scores

---

## Governance Metrics and Reporting

Security metrics translate operational data into business-relevant insights for leadership.

**Operational metrics:**

| Metric | What it measures | Target direction |
|--------|-----------------|-----------------|
| **Mean Time to Detect (MTTD)** | Time from incident occurrence to detection | Lower is better |
| **Mean Time to Respond (MTTR)** | Time from detection to containment/resolution | Lower is better |
| **Vulnerability remediation SLA compliance** | % of vulnerabilities patched within SLA | Higher is better |
| **Patching cadence** | Time to deploy critical patches | Lower is better |
| **Access review completion** | % of access reviews completed on time | 100% target |
| **Security training completion** | % of employees completing required training | 100% target |

**Risk metrics:**

| Metric | What it measures |
|--------|-----------------|
| **Open risk items** | Count and age of items in the risk register |
| **Risk acceptance vs treatment ratio** | Balance between accepted and mitigated risks |
| **Third-party risk score distribution** | Vendor risk posture across the portfolio |
| **Audit finding closure rate** | % of audit findings remediated within deadline |

**Executive dashboard elements:**

- Overall risk posture (heat map or score)
- Compliance status across frameworks (on track / at risk / non-compliant)
- Top 5 risks with treatment status
- Incident trends (volume, severity, type)
- Vulnerability management SLA compliance
- Vendor risk summary
- Security program maturity score (against a model like CMMI or NIST CSF tiers)

!!! info "External Resources"
    - [CIS Controls v8 - Security Metrics](https://www.cisecurity.org/controls) (CIS)
    - [NIST SP 800-55 - Performance Measurement Guide](https://csrc.nist.gov/publications/detail/sp/800-55/rev-2/draft) (NIST)
