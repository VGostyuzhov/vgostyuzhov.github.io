# Threat Modeling Frameworks

## STRIDE

STRIDE is a threat classification model developed by Microsoft. Each letter represents a category of threat mapped to a security property.

| Category | Threat | Security Property Violated | Example |
|----------|--------|---------------------------|---------|
| **S**poofing | Pretending to be someone/something else | Authentication | Forged JWT, stolen session cookie, IP spoofing |
| **T**ampering | Modifying data or code | Integrity | SQL injection altering records, man-in-the-middle modifying requests |
| **R**epudiation | Denying an action was performed | Non-repudiation | User claims they did not authorise a transaction; no audit log exists |
| **I**nformation Disclosure | Exposing data to unauthorized parties | Confidentiality | SSRF reading cloud metadata, verbose error messages, directory traversal |
| **D**enial of Service | Making a system unavailable | Availability | DDoS, resource exhaustion, deadlocks, algorithmic complexity attacks |
| **E**levation of Privilege | Gaining unauthorized access level | Authorization | IDOR, broken access control, container escape, kernel exploit |

**How to apply STRIDE:**

1. Draw a data flow diagram (DFD) of the system
2. Identify trust boundaries (where data crosses privilege levels)
3. For each element (process, data store, data flow, external entity), ask each STRIDE question
4. Document threats with description, affected component, and severity
5. Propose mitigations for each threat
6. Track mitigations as backlog items

**When to use:** Early in design phase. Best for application and system-level threat modeling. Good fit for engineering teams new to threat modeling because the categories are intuitive.

!!! info "External Resources"
    - [Microsoft STRIDE Documentation](https://learn.microsoft.com/en-us/azure/security/develop/threat-modeling-tool-threats) (Microsoft)
    - [Threat Modeling: Designing for Security](https://shostack.org/books/threat-modeling-book) (Adam Shostack)
    - [OWASP Threat Modeling](https://owasp.org/www-community/Threat_Modeling) (OWASP)

## PASTA (Process for Attack Simulation and Threat Analysis)

PASTA is a seven-stage, risk-centric threat modeling methodology that aligns business objectives with technical risk.

**Seven stages:**

| Stage | Name | Activities |
|-------|------|-----------|
| 1 | Define objectives | Business impact analysis, compliance requirements, risk appetite |
| 2 | Define technical scope | Architecture diagrams, technology stack, dependencies |
| 3 | Application decomposition | Data flow diagrams, trust boundaries, entry/exit points |
| 4 | Threat analysis | Threat intelligence, relevant threat actors, attack patterns |
| 5 | Vulnerability analysis | Map vulnerabilities to threats, scan results, code review findings |
| 6 | Attack modeling | Build attack trees, simulate attack paths, enumerate exploits |
| 7 | Risk and impact analysis | Calculate risk scores, prioritise mitigations, create remediation plan |

**PASTA vs STRIDE:**

| Property | STRIDE | PASTA |
|----------|--------|-------|
| Focus | Threat categories | Risk-based analysis |
| Input | Technical architecture | Business context + technical architecture |
| Output | Threat list per component | Prioritised risk register with business impact |
| Complexity | Lower (can be done on whiteboard) | Higher (requires multiple stakeholders) |
| Best for | Development teams | Enterprise risk management |

!!! info "External Resources"
    - [PASTA Threat Modeling - Tony UcedaVelez](https://www.wiley.com/en-us/Risk+Centric+Threat+Modeling-p-9780470500965) (Wiley)
    - [OWASP PASTA Methodology](https://owasp.org/www-community/Threat_Modeling_Process#pasta) (OWASP)
    - [Threat Modeling Manifesto](https://www.threatmodelingmanifesto.org/) (Community)

## MITRE ATT&CK Framework

MITRE ATT&CK (Adversarial Tactics, Techniques, and Common Knowledge) is a knowledge base of adversary behaviour based on real-world observations.

**Structure:**

- **Tactics** - the adversary's goal (14 tactics for Enterprise: Reconnaissance through Impact)
- **Techniques** - how the tactic is achieved (~200 techniques with sub-techniques)
- **Procedures** - specific implementations observed in the wild
- **Software** - tools and malware mapped to techniques
- **Groups** - threat actor profiles mapped to techniques

**Matrices:**

| Matrix | Scope |
|--------|-------|
| Enterprise | Windows, Linux, macOS, Cloud (AWS, GCP, Azure), Network, Containers |
| Mobile | Android, iOS |
| ICS | Industrial control systems |

**Uses in security operations:**

- **Detection engineering** - map detection rules to ATT&CK techniques; identify coverage gaps
- **Threat intelligence** - understand adversary TTPs; prioritise defences against relevant groups
- **Red teaming** - structure attack simulations using ATT&CK techniques
- **Incident response** - classify observed activity by technique for consistent reporting
- **Security assessment** - evaluate security posture against known adversary behaviours

**ATT&CK Navigator:** Visual tool for annotating the ATT&CK matrix with detection coverage, red team results, or threat intelligence overlays.

!!! info "External Resources"
    - [MITRE ATT&CK](https://attack.mitre.org/) (MITRE)
    - [ATT&CK Navigator](https://mitre-attack.github.io/attack-navigator/) (MITRE)
    - [MITRE ATT&CK Design and Philosophy](https://attack.mitre.org/docs/ATTACK_Design_and_Philosophy_March_2020.pdf) (MITRE)

## Attack Trees

Attack trees model threats as a hierarchical tree structure where the root node is the attacker's goal, and child nodes represent the steps or conditions needed to achieve it.

**Structure:**

- **Root node** - attacker's objective (e.g., "Steal customer database")
- **Child nodes** - sub-goals or methods to achieve parent (AND/OR decomposition)
- **AND nodes** - all child conditions must be met
- **OR nodes** - any one child condition is sufficient
- **Leaf nodes** - atomic actions the attacker takes

**Example attack tree (simplified):**

```
Steal customer database (OR)
├── SQL injection (OR)
│   ├── Find unparameterised query
│   └── Exploit blind SQLi
├── Compromise admin account (AND)
│   ├── Phish admin credentials
│   └── Bypass MFA
└── Insider threat
    └── Bribe database administrator
```

**Annotating attack trees:**

- Cost to attacker (time, money, skill level)
- Probability of success
- Detectability (likelihood of triggering alerts)
- Boolean feasibility (possible/impossible given current controls)

**Benefits:** Forces systematic thinking about all attack paths. Identifies cheapest/easiest paths for the attacker. Helps prioritise which mitigations have the highest impact.

!!! info "External Resources"
    - [Attack Trees - Bruce Schneier](https://www.schneier.com/academic/archives/1999/12/attack_trees.html) (Schneier)
    - [OWASP Attack Tree Guide](https://owasp.org/www-community/Threat_Modeling_Process) (OWASP)
    - [SeaSponge - Attack Tree Tool](https://github.com/nickthecook/seasponge) (GitHub)

## DREAD (Deprecated but Referenced)

DREAD is a risk scoring model formerly used by Microsoft. Deprecated due to subjectivity but still referenced in legacy documentation and interviews.

| Factor | Question | Scale |
|--------|----------|-------|
| **D**amage | How severe is the impact? | 1-10 |
| **R**eproducibility | How easy is it to reproduce the attack? | 1-10 |
| **E**xploitability | How much skill/effort is required? | 1-10 |
| **A**ffected users | How many users are impacted? | 1-10 |
| **D**iscoverability | How easy is it to find the vulnerability? | 1-10 |

**Risk score = (D + R + E + A + D) / 5**

**Why deprecated:**

- Scores are highly subjective - different analysts produce different results
- Discoverability is problematic (assumes security through obscurity)
- No formal methodology for calibrating scores
- CVSS provides a more standardised alternative for vulnerability scoring

**Modern alternatives:** CVSS (vulnerability scoring), STRIDE + risk matrix (threat modeling), FAIR (quantitative risk analysis)

!!! info "External Resources"
    - [DREAD Risk Assessment - Microsoft](https://learn.microsoft.com/en-us/archive/blogs/david_leblanc/dreadful) (Microsoft)
    - [CVSS Calculator - FIRST](https://www.first.org/cvss/calculator/3.1) (FIRST)
    - [FAIR Risk Model](https://www.fairinstitute.org/) (FAIR Institute)

## Choosing a Framework

| Factor | STRIDE | PASTA | ATT&CK | Attack Trees | DREAD |
|--------|--------|-------|--------|-------------|-------|
| **Primary use** | Application threat modeling | Enterprise risk assessment | Detection and response | Specific attack path analysis | Quick risk scoring |
| **Complexity** | Low-medium | High | Medium (for consumption) | Medium | Low |
| **Output** | Threat list per component | Prioritised risk register | Detection coverage map | Attack path visualisation | Numeric risk score |
| **Best audience** | Development teams | Security + business leadership | SOC, detection engineering, red team | Security architects | Legacy/quick triage |
| **Current status** | Active, widely used | Active, growing | Active, industry standard | Active, niche | Deprecated |

**Practical advice:**

- **Start with STRIDE** if your team is new to threat modeling - it is structured enough to be useful but simple enough to learn quickly
- **Use ATT&CK** to ensure detection coverage maps to real-world adversary behaviour
- **Use PASTA** when business stakeholders need risk quantification tied to business objectives
- **Use attack trees** when analysing specific high-value attack scenarios in depth
- Frameworks are complementary, not mutually exclusive - many teams use STRIDE for design and ATT&CK for detection

!!! info "External Resources"
    - [Threat Modeling Manifesto](https://www.threatmodelingmanifesto.org/) (Community)
    - [OWASP Threat Modeling](https://owasp.org/www-community/Threat_Modeling) (OWASP)
    - [Threat Modeling: A Practical Guide for Development Teams](https://www.oreilly.com/library/view/threat-modeling/9781492056546/) (O'Reilly)
