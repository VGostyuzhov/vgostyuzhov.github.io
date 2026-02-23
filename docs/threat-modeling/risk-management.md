# Risk Management

## Risk Assessment Fundamentals

Risk is the potential for loss or harm when a threat exploits a vulnerability. Risk assessment identifies, analyses, and evaluates risks to inform decision-making.

**Core formula:** `Risk = Likelihood x Impact`

**Key terms:**

| Term | Definition |
|------|-----------|
| **Threat** | A potential cause of harm (attacker, natural disaster, insider) |
| **Vulnerability** | A weakness that can be exploited (unpatched software, misconfiguration) |
| **Asset** | Something of value to protect (data, systems, reputation) |
| **Risk** | The potential for a threat to exploit a vulnerability and cause harm to an asset |
| **Control** | A measure that reduces risk (firewall, encryption, policy) |
| **Residual risk** | Risk remaining after controls are applied |
| **Risk appetite** | The level of risk an organisation is willing to accept |

**Assessment approaches:**

| Approach | Method | Pros | Cons |
|----------|--------|------|------|
| **Qualitative** | Risk matrix (High/Medium/Low) | Simple, fast, accessible | Subjective, imprecise |
| **Quantitative** | Financial loss calculation (ALE, SLE, ARO) | Precise, business-aligned | Data-intensive, assumptions required |
| **Semi-quantitative** | Numeric scales with defined criteria | Balance of precision and simplicity | Still somewhat subjective |

!!! info "External Resources"
    - [NIST SP 800-30 - Risk Assessment Guide](https://csrc.nist.gov/publications/detail/sp/800-30/rev-1/final) (NIST)
    - [ISO 27005 - Information Security Risk Management](https://www.iso.org/standard/75281.html) (ISO)
    - [FAIR Institute](https://www.fairinstitute.org/) (FAIR Institute)

## Risk Prioritization (Likelihood x Impact)

**Risk matrix (qualitative):**

| | Low Impact | Medium Impact | High Impact | Critical Impact |
|---|-----------|--------------|-------------|----------------|
| **High Likelihood** | Medium | High | Critical | Critical |
| **Medium Likelihood** | Low | Medium | High | Critical |
| **Low Likelihood** | Low | Low | Medium | High |

**Likelihood factors:**

- Threat actor capability and motivation
- Vulnerability exposure (internet-facing, internal, air-gapped)
- Existing controls and their effectiveness
- Historical incident frequency
- Industry targeting patterns

**Impact factors:**

- Data sensitivity (PII, financial, health, classified)
- Number of affected users/customers
- Financial loss (direct costs, fines, remediation)
- Reputational damage
- Regulatory and legal consequences
- Operational disruption duration

**Quantitative risk metrics:**

| Metric | Formula | Meaning |
|--------|---------|---------|
| **SLE** (Single Loss Expectancy) | Asset Value x Exposure Factor | Expected loss from one incident |
| **ARO** (Annualised Rate of Occurrence) | Number of incidents per year | How often the event is expected |
| **ALE** (Annualised Loss Expectancy) | SLE x ARO | Expected yearly cost of the risk |

**Prioritisation in practice:**

- Rank risks by ALE (quantitative) or risk matrix position (qualitative)
- Focus resources on risks with highest combination of likelihood and impact
- Consider risk velocity - how fast the impact materialises (ransomware = minutes; data leak = gradual)
- Re-evaluate periodically; threat landscape and business context change

!!! info "External Resources"
    - [NIST SP 800-30 - Risk Matrix Examples](https://csrc.nist.gov/publications/detail/sp/800-30/rev-1/final) (NIST)
    - [CVSS - Common Vulnerability Scoring System](https://www.first.org/cvss/) (FIRST)
    - [OWASP Risk Rating Methodology](https://owasp.org/www-community/OWASP_Risk_Rating_Methodology) (OWASP)

## Risk Treatment Options (Accept, Mitigate, Transfer, Avoid)

After assessing risks, each must be treated with one of four strategies.

| Treatment | Action | When to use | Example |
|-----------|--------|-------------|---------|
| **Mitigate** | Implement controls to reduce likelihood or impact | Risk is above appetite but can be reduced cost-effectively | Deploy WAF, enable MFA, encrypt data |
| **Accept** | Acknowledge risk and take no additional action | Residual risk is within appetite; cost of mitigation exceeds potential loss | Low-severity vulnerability in internal tool |
| **Transfer** | Shift financial impact to another party | Risk impact is primarily financial; insurance or SLA covers it | Cyber insurance, outsourcing to managed security provider |
| **Avoid** | Eliminate the risk by removing the source | Risk is unacceptable and cannot be adequately mitigated | Discontinue a service, do not collect certain data |

**Decision factors:**

- Cost of mitigation vs expected loss (cost-benefit analysis)
- Regulatory requirements (some risks cannot be accepted)
- Organisational risk appetite and tolerance
- Stakeholder expectations
- Available resources and timeline

**Documenting risk treatment:**

- Each risk should have a documented treatment decision
- Include justification, responsible owner, and review date
- Accepted risks require explicit sign-off from appropriate authority
- Track treatment implementation status in a risk register

!!! info "External Resources"
    - [ISO 27005 - Risk Treatment](https://www.iso.org/standard/75281.html) (ISO)
    - [NIST SP 800-39 - Managing Information Security Risk](https://csrc.nist.gov/publications/detail/sp/800-39/final) (NIST)
    - [CISO's Guide to Risk Management](https://www.sans.org/white-papers/risk-management/) (SANS)

## Blast Radius & Containment

Blast radius is the scope of impact when a security control fails or a system is compromised. Containment strategies limit blast radius.

**Blast radius dimensions:**

| Dimension | Question | Example |
|-----------|----------|---------|
| **Data** | How much data is exposed? | Single record vs entire database |
| **Systems** | How many systems are affected? | One container vs entire cluster |
| **Users** | How many users are impacted? | One account vs all customers |
| **Time** | How long until detection and containment? | Minutes (automated) vs months (APT) |
| **Trust** | How much trust is broken? | Internal service vs customer-facing auth |

**Containment strategies:**

- **Network segmentation** - VLANs, VPCs, microsegmentation limit lateral movement
- **Account isolation** - separate cloud accounts/projects per environment (dev/staging/prod)
- **Least privilege** - minimize permissions so compromised identity has limited reach
- **Service isolation** - one service account per workload; one role per function
- **Data classification** - encrypt sensitive data separately; access requires additional authorization
- **Short-lived credentials** - compromised tokens expire before significant damage
- **Circuit breakers** - automated shutdown of services exhibiting anomalous behaviour

**Interview framing:** When discussing any security control, frame it in terms of blast radius reduction. "This control limits the blast radius because..."

!!! info "External Resources"
    - [AWS Multi-Account Strategy](https://docs.aws.amazon.com/whitepapers/latest/organizing-your-aws-environment/organizing-your-aws-environment.html) (AWS)
    - [Google Cloud Resource Hierarchy](https://cloud.google.com/resource-manager/docs/cloud-platform-resource-hierarchy) (Google Cloud)
    - [NIST SP 800-53 - Security Controls](https://csrc.nist.gov/publications/detail/sp/800-53/rev-5/final) (NIST)

## Communicating Risk to Stakeholders

Effective risk communication translates technical findings into business-relevant language for different audiences.

**Audience-specific communication:**

| Audience | What they care about | Language |
|----------|---------------------|---------|
| **Executive/Board** | Business impact, financial exposure, competitive risk | Revenue, reputation, regulatory fines, strategic risk |
| **Engineering leadership** | Technical debt, velocity impact, resource allocation | Sprint impact, architectural risk, remediation effort |
| **Development team** | What to fix and how | Specific vulnerability, code location, remediation steps |
| **Compliance/Legal** | Regulatory exposure, audit findings | Framework reference (SOC 2, GDPR), control gaps |

**Effective risk reporting:**

- Lead with business impact, not technical details
- Use consistent risk scoring (risk matrix, CVSS) across reports
- Provide trend data - is risk improving or worsening over time
- Include comparison to industry benchmarks where available
- Offer clear recommendations with effort estimates
- Avoid jargon; explain technical concepts in business terms

**Common pitfalls:**

- Presenting raw vulnerability counts without context (100 vulns means nothing without severity and exploitability)
- Using fear-based messaging instead of data-driven analysis
- Failing to tie security investment to risk reduction
- Reporting risks without actionable recommendations

!!! info "External Resources"
    - [FAIR Analysis - Factor Analysis of Information Risk](https://www.fairinstitute.org/what-is-fair) (FAIR Institute)
    - [NIST Cybersecurity Framework - Communicate](https://www.nist.gov/cyberframework) (NIST)
    - [CISO Risk Communication - Gartner](https://www.gartner.com/en/cybersecurity) (Gartner)

## Risk Registers & Tracking

A risk register is the central document that records identified risks, their assessment, treatment decisions, and tracking status.

**Risk register fields:**

| Field | Description |
|-------|------------|
| **Risk ID** | Unique identifier |
| **Description** | What could happen and why |
| **Category** | Technical, operational, compliance, strategic |
| **Likelihood** | Probability rating (1-5 or qualitative) |
| **Impact** | Severity rating (1-5 or qualitative) |
| **Risk score** | Likelihood x Impact |
| **Treatment** | Mitigate, accept, transfer, or avoid |
| **Controls** | Existing and planned mitigations |
| **Owner** | Person accountable for managing this risk |
| **Status** | Open, in progress, accepted, closed |
| **Review date** | Next scheduled reassessment |
| **Residual risk** | Risk remaining after controls |

**Risk tracking practices:**

- Review the risk register at least quarterly
- Update after significant incidents, architecture changes, or new threat intelligence
- Track risk trends over time (are risks being closed, or accumulating?)
- Escalate risks that exceed thresholds to appropriate governance bodies
- Integrate risk register with vulnerability management and incident tracking systems

**Tools:** GRC platforms (ServiceNow GRC, Archer, LogicGate), spreadsheet-based registers (sufficient for smaller organisations), JIRA/project management tools with custom fields.

!!! info "External Resources"
    - [NIST SP 800-39 - Risk Management Framework](https://csrc.nist.gov/publications/detail/sp/800-39/final) (NIST)
    - [ISO 31000 - Risk Management](https://www.iso.org/iso-31000-risk-management.html) (ISO)
    - [OWASP Risk Assessment Framework](https://owasp.org/www-project-risk-assessment-framework/) (OWASP)
