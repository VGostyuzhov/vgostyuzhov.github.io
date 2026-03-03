# A) Security Fundamentals & Core Concepts (1–10)

## 1) What is the CIA Triad and how does it apply in real systems?
* **Focus:** Foundational security thinking
* **Core Idea:** Security is about tradeoffs, not absolutes
* **Strong Answers Cover:**
  * Risk-based prioritisation between availability vs confidentiality
  * Business impact of breaches vs downtime
  * Why availability failures often matter more than data loss
  * Long-term trust implications

## 2) Authentication vs Authorisation. Explain with examples.
* **Focus:** Identity clarity
* **Core Idea:** Most security bugs are access bugs
* **Strong Answers Cover:**
  * Clear boundary between identity and permissions
  * Real production failures caused by confusion
  * How mistakes scale quietly
  * Developer ergonomics vs safety

## 3) Explain XSS, CSRF, and SSRF.
* **Focus:** Web security reasoning
* **Core Idea:** Attacks exploit misplaced trust
* **Strong Answers Cover:**
  * Threat models, not definitions
  * Where trust breaks in modern apps
  * Prevention as design, not filters
  * Developer education impact

## 4) What is SQL Injection and how is it prevented properly?
* **Focus:** Secure coding fundamentals
* **Core Idea:** Input handling is architecture, not validation
* **Strong Answers Cover:**
  * Parameterisation vs sanitisation
  * ORM false sense of safety
  * Risk reduction at framework level
  * Long-term maintenance impact

## 5) What is a replay attack and how do you prevent it?
* **Focus:** Protocol reasoning
* **Core Idea:** Freshness matters as much as secrecy
* **Strong Answers Cover:**
  * Nonces, timestamps, token expiry
  * Tradeoffs with distributed systems
  * Failure modes under clock skew
  * Operational risks

## 6) Hashing vs Encryption vs Encoding
* **Focus:** Crypto fundamentals
* **Core Idea:** Wrong primitive equals broken security
* **Strong Answers Cover:**
  * Use cases, not math
  * Irreversibility vs confidentiality
  * Compliance expectations
  * Developer misuse patterns

## 7) Symmetric vs Asymmetric encryption
* **Focus:** Practical crypto usage
* **Core Idea:** Trust and performance tradeoffs
* **Strong Answers Cover:**
  * Key distribution challenges
  * Hybrid models (TLS)
  * Scalability concerns
  * Real-world misuse

## 8) How does TLS work end to end?
* **Focus:** Secure communication depth
* **Core Idea:** Validation failures break everything
* **Strong Answers Cover:**
  * Cert chains and trust anchors
  * MITM risks
  * Misconfiguration risks
  * Long-term platform trust

## 9) What is Least Privilege and how do you enforce it?
* **Focus:** Access design
* **Core Idea:** Permissions decay silently
* **Strong Answers Cover:**
  * Guardrails over reviews
  * Automation vs manual enforcement
  * Developer trust balance
  * Long-term blast radius reduction

## 10) What is Defense in Depth?
* **Focus:** Security architecture
* **Core Idea:** Assume failure, not perfection
* **Strong Answers Cover:**
  * Layered controls
  * Cost vs protection tradeoffs
  * Failure containment
  * Strategic resilience

---

# B) Cloud Security & Infrastructure (11–19)

## 11) Common IAM misconfigurations you've seen
* **Focus:** Real-world experience
* **Core Idea:** Identity is the cloud perimeter
* **Strong Answers Cover:**
  * Over-permissioned roles
  * Lateral movement risk
  * Monitoring gaps
  * Long-term permission creep

## 12) How do you design IAM for microservices?
* **Focus:** Service identity
* **Core Idea:** Humans shouldn't be in the auth path
* **Strong Answers Cover:**
  * Service-to-service auth
  * Short-lived credentials
  * Operational complexity
  * Scalability impact

## 13) How do you secure cloud networking?
* **Focus:** Network isolation
* **Core Idea:** Flat networks amplify damage
* **Strong Answers Cover:**
  * Segmentation strategies
  * Zero trust assumptions
  * Cost vs complexity
  * Failure containment

## 14) Securing object storage like S3 or GCS
* **Focus:** Data exposure prevention
* **Core Idea:** Defaults are dangerous
* **Strong Answers Cover:**
  * Access policies
  * Logging and alerting
  * Public exposure risks
  * Long-term data trust

## 15) Secrets management in cloud environments
* **Focus:** Sensitive data handling
* **Core Idea:** Secrets leak through convenience
* **Strong Answers Cover:**
  * Rotation strategies
  * Developer UX
  * Automation vs risk
  * Incident response readiness

## 16) What is shared responsibility in cloud security?
* **Focus:** Accountability clarity
* **Core Idea:** Assumptions cause breaches
* **Strong Answers Cover:**
  * Provider vs customer boundaries
  * Misplaced trust
  * Audit readiness
  * Long-term ownership clarity

## 17) Cloud logging strategy for security
* **Focus:** Visibility
* **Core Idea:** You cannot defend what you cannot see
* **Strong Answers Cover:**
  * Control plane vs data plane
  * Cost tradeoffs
  * Signal quality
  * Detection maturity

## 18) How do you think about cloud threat modeling?
* **Focus:** Attacker mindset
* **Core Idea:** Assume breach
* **Strong Answers Cover:**
  * Identity abuse
  * Lateral movement
  * Persistence
  * Recovery planning

## 19) Cloud security vs on-prem security differences
* **Focus:** Mental model shift
* **Core Idea:** Speed increases risk
* **Strong Answers Cover:**
  * Automation impact
  * Shared tooling risk
  * Control loss
  * Long-term governance

---

# C) Application Security & Secure SDLC (20–28)

## 20) How do you secure APIs in a modern microservices architecture?
* **Focus:** AppSec design thinking
* **Core Idea:** APIs are the most attacked surface today
* **Strong answers cover:**
  * AuthN vs AuthZ at the API layer
  * Rate limiting, abuse detection, and input validation
  * Tradeoffs between security and developer velocity
  * Long-term API versioning and backward compatibility risks

## 21) What are the most common application security mistakes you see?
* **Focus:** Pattern recognition
* **Core Idea:** Most bugs repeat, only contexts change
* **Strong answers cover:**
  * Broken access control
  * Secrets exposure
  * Insecure defaults
  * Why teams repeat the same mistakes at scale

## 22) How do you integrate security into CI/CD pipelines?
* **Focus:** Shift-left maturity
* **Core Idea:** Security that blocks pipelines gets bypassed
* **Strong answers cover:**
  * SAST vs DAST vs dependency scanning tradeoffs
  * False positive management
  * Developer trust and adoption
  * Measuring effectiveness over time

## 23) How do you think about dependency and supply chain security?
* **Focus:** Modern attack vectors
* **Core Idea:** Your code is only as safe as your weakest dependency
* **Strong answers cover:**
  * SBOMs and dependency visibility
  * Risk-based patching
  * Balancing velocity with exposure
  * Long-term ecosystem risk

## 24) How do you handle secrets in application code?
* **Focus:** Secure engineering discipline
* **Core Idea:** Secrets leak through convenience
* **Strong answers cover:**
  * Environment-based secret injection
  * Rotation and revocation
  * Developer ergonomics
  * Incident response readiness

## 25) What is your approach to secure authentication flows?
* **Focus:** Identity-first security
* **Core Idea:** Auth bugs are catastrophic bugs
* **Strong answers cover:**
  * Token lifecycle management
  * Session handling risks
  * Tradeoffs between UX and security
  * Long-term identity trust

## 26) How do you test security controls in applications?
* **Focus:** Validation mindset
* **Core Idea:** Controls that aren't tested don't exist
* **Strong answers cover:**
  * Automated vs manual testing
  * Regression prevention
  * Coverage gaps
  * Metrics for confidence

## 27) How do you educate developers about security without slowing them down?
* **Focus:** Influence and communication
* **Core Idea:** Security scales through people, not policies
* **Strong answers cover:**
  * Just-in-time education
  * Secure defaults
  * Feedback loops
  * Long-term culture building

## 28) How do you balance security requirements with product deadlines?
* **Focus:** Judgment
* **Core Idea:** Security is prioritisation, not absolutism
* **Strong answers cover:**
  * Risk acceptance vs mitigation
  * Business alignment
  * Documented tradeoffs
  * Trust with leadership

---

# D) Threat Modeling & System Design (29–35)

## 29) How do you approach threat modeling for a new system?
* **Focus:** Structured thinking
* **Core Idea:** Ask before answering
* **Strong answers cover:**
  * Assets, actors, trust boundaries
  * Clarifying assumptions
  * Threat prioritisation
  * Long-term design impact

## 30) Threat model a real-world product (e.g., payment system)
* **Focus:** Applied reasoning
* **Core Idea:** Practical beats theoretical
* **Strong answers cover:**
  * Abuse cases
  * Failure modes
  * Defense tradeoffs
  * Business risk alignment

## 31) How do you prioritise security risks?
* **Focus:** Risk management
* **Core Idea:** Not all risks deserve equal attention
* **Strong answers cover:**
  * Impact vs likelihood
  * User trust implications
  * Cost of mitigation
  * Long-term exposure reduction

## 32) How do you think about blast radius?
* **Focus:** Containment strategy
* **Core Idea:** Fail safely, not perfectly
* **Strong answers cover:**
  * Isolation techniques
  * Least privilege
  * Recovery time objectives
  * Organisational resilience

## 33) How do you design systems assuming breach?
* **Focus:** Defensive realism
* **Core Idea:** Breach is inevitable
* **Strong answers cover:**
  * Lateral movement prevention
  * Detection-first mindset
  * Recovery planning
  * Long-term survivability

## 34) How do you evaluate new security tools or frameworks?
* **Focus:** Tool judgment
* **Core Idea:** Tools don't fix bad thinking
* **Strong answers cover:**
  * Risk reduction vs complexity
  * Integration cost
  * Developer trust
  * Long-term maintainability

## 35) How do you communicate risk to non-technical stakeholders?
* **Focus:** Leadership communication
* **Core Idea:** Security fails when it can't be explained
* **Strong answers cover:**
  * Business impact framing
  * Clear tradeoffs
  * Metrics that matter
  * Executive trust

---

# E) Detection Engineering & Incident Response (36–38)

## 36) How do you design a detection strategy?
* **Focus:** Visibility
* **Core Idea:** Detection beats prevention alone
* **Strong answers cover:**
  * Signal-to-noise ratio
  * Coverage gaps
  * Cost tradeoffs
  * Continuous improvement

## 37) What logs are most important for security?
* **Focus:** Observability
* **Core Idea:** Logs are your memory
* **Strong answers cover:**
  * Identity events
  * Privilege changes
  * Control-plane activity
  * Long-term forensic value

## 38) How do you reduce alert fatigue?
* **Focus:** Operational maturity
* **Core Idea:** Burned-out teams miss real attacks
* **Strong answers cover:**
  * Alert quality metrics
  * Context enrichment
