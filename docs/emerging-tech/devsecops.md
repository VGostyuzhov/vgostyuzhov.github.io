# DevSecOps

## Secure SDLC & Threat Modeling

**1. SDLC phases and security activities**

| SDLC phase | Primary security goal | Key activities / artifacts | Shifts when done early |
|------------|----------------------|----------------------------|------------------------|
| **Requirements** | Capture security & compliance needs | Misuse-case workshop, data-classification matrix, privacy impact assessment | Limits costly redesign later |
| **Design** | Build in safeguards before code | Threat Model (STRIDE / PASTA), security architecture review, crypto choices, trust-boundary diagrams | Architects pick correct patterns first time |
| **Implementation** | Prevent vulnerabilities in code | Secure coding standards, SAST in CI, secret-scan, peer review checklist, dependency audit (SBOM) | Bugs found minutes after commit |
| **Verification / Test** | Prove controls work | DAST, API fuzzing, unit + integration security tests, container image scan | Gate broken builds, fail fast |
| **Deployment** | Ship hardened artifacts | IaC lint (Terraform, K8s), least-priv service accounts, signed images, supply-chain attestations | Blocks misconfig drift on every release |
| **Operations & Maintenance** | Detect & respond quickly | Runtime SBOM diff, patch cadence, vuln management, alert tuning, chaos drills | Shortens mean-time-to-detect + repair |
| **Retirement** | Remove or quarantine safely | Data retention policy, key destruction, sun-setting playbook | No zombie attack surface |

---

**2. Core Threat Modeling workflow (STRIDE example)**

1. **Diagram the system** - identify data flows, processes, external entities, data stores.
2. **Identify threats** - for each element ask STRIDE: *Spoofing, Tampering, Repudiation, Information disclosure, Denial of service, Elevation of privilege*.
3. **Rank risk** - Likelihood x Impact - high-risk items bubble up.
4. **Mitigate early** - add or strengthen controls (e.g., auth, input validation, rate limit, logging).
5. **Validate** - ensure mitigations lower risk; peer review and update artifacts.
6. **Iterate** - revisit at every significant design/code change.

> **Outputs**: updated DFD, threat list, risk matrix, mitigation backlog items linked to sprint board.

---

**3. Embedding security in modern CI/CD**

* **Pre-commit hooks** - secret scanners, lint rules.
* **PR pipeline** - SAST, dependency-vuln DB check, license policy, code-review bot enforcing security checklist.
* **Build pipeline** - Reproducible build, SBOM emission, container scanning, signature (Sigstore/Cosign).
* **Deploy pipeline** - IaC security gates, policy as code (OPA Gatekeeper), runtime admission control.
* **Post-deploy** - Canary alert rules, CSP / HSTS headers verification, synthetic probes.

---

**4. Checklists by role**

| Role | Must-do security tasks |
|------|------------------------|
| Product Owner | Capture regulations (GDPR, PCI); define risk appetite. |
| Architect | Produce threat model; approve crypto & auth patterns. |
| Developer | Follow language-specific secure coding guide; fix SAST findings promptly. |
| DevOps | Enforce least-priv in cloud/IaC; rotate secrets; backup & test restores. |
| QA / Security Engineer | Design test cases for abuse, run DAST/penetration tests. |
| SRE | Monitor logs, deploy WAF, tune alerts, lead incident drills. |

---

**5. Metrics to prove SSDLC maturity**

* % stories with **security acceptance criteria**
* Time-to-remediate critical SAST issues <= **7 days**
* Coverage of **threat models per high-risk feature**
* Mean time between **dependency updates**
* Ratio of **incidents traced to design flaws vs implementation bugs**

Continuous tracking turns ad-hoc "pen-test at the end" into reproducible, data-driven secure development.

---

Adopt a Secure SDLC mindset: **shift left**, automate, threat-model early, and treat security findings like any other quality metric. The payoff is fewer production fires and faster, safer releases.

!!! info "External Resources"
    - [OWASP Secure SDLC Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Secure_Product_Design_Cheat_Sheet.html) (OWASP)
    - [NIST Secure Software Development Framework](https://csrc.nist.gov/Projects/ssdf) (NIST)
    - [Microsoft Security Development Lifecycle](https://www.microsoft.com/en-us/securityengineering/sdl) (Microsoft)

## Dependency Management

**1. Core goals**

* Guarantee that every build uses **known, reproducible versions**.
* Detect and patch **vulnerabilities** in direct *and* transitive packages.
* Track **licenses** and legal obligations.
* Protect build pipeline from **malicious or hijacked packages**.

---

**2. Lock, pin, and verify**

| Practice | What it does | Typical tool | Extra tips |
|----------|--------------|--------------|------------|
| **Version pinning** | Explicitly states exact version. | `requirements.txt`, `package.json` with `"lodash": "4.17.21"` | Pin *all* deps, not `^1.2`. |
| **Lockfile** | Captures full dep graph with checksums. | `poetry.lock`, `package-lock.json`, `Cargo.lock`, `go.sum` | Commit lockfile; fail build if diff not reviewed. |
| **Checksum / signature verification** | Ensures binary/module wasn't tampered with. | `npm ci` (shasum), Maven Central PGP, `pip --require-hashes` | Turn on **npm's `--strict-peer-deps`**. |
| **Reproducible builds** | Bit-for-bit identical artefacts from same source. | Go, Rust, Bazel | Store artefact hash in SBOM/attestation. |

---

**3. Automated scanning & monitoring**

| Stage | What to scan | Tools (example) |
|-------|--------------|-----------------|
| **Pull Request** | SCA (static), secret leaks, license policy | Snyk, Renovate, Trivy, Gitleaks |
| **Build** | SBOM generation, checksum compare | CycloneDX, Syft, Grype |
| **Registry / Image** | CVE feed diff, outdated base images | Clair, Anchore, AWS ECR scan |
| **Runtime** | In-memory lib version, process hash | Falco, eBPF scanners |

*Alert rules* - "Critical CVE present in prod > 30 days" triggers escalation.

---

**4. Update strategy**

* **Renovate / Dependabot** create automatic PRs; configure *batch & schedule* (e.g., every Monday).
* **SemVer tiers** - auto-merge patch/minor; manual review for major.
* Maintain **changelogs** and roll-forward plans; use canary deploy to catch regressions.
* If vendor won't patch quickly, apply **selective patching** (fork with fix) or **security shim** (e.g., `npm audit fix --force` plus lockfile diff).

---

**5. Supply-chain hardening**

1. **Use private proxies / caches** (e.g., Artifactory, Verdaccio) to avoid live pulls from public registries during build.
2. Enable **Two-factor auth & signing** for package publishers in your org.
3. Require **verified commit signatures** (`git config commit.gpgsign true`) on main branches.
4. Adopt **Sigstore/cosign** to sign OCI images and verify in cluster admission control.
5. Implement **Provenance attestations** (SLSA >= level 2) so downstream consumers trust your output.
6. Watch for **typosquatting / dependency confusion** - use scoped package names and set `pypi-proxy=false` for internal libs.

---

**6. License compliance quick-view**

| License family | Typical constraint | Action |
|----------------|--------------------|--------|
| *Permissive* (MIT, BSD, Apache-2.0) | Notice & attribution | Keep NOTICE file in artefact. |
| *Weak Copyleft* (MPL-2.0, LGPL-2.1) | Modifications to library must be released | Prefer dynamic linking. |
| *Strong Copyleft* (GPL-3.0) | Derivative work must be OSS | Avoid in SaaS closed source unless business OK. |
| *Patents* (Apache-2.0) | Grants patent license | Safe for commercial use. |

Automate license scan - block build on forbidden licenses, generate attribution docs for distribution.

---

**7. Incident response when a dependency pops a zero-day**

* **Assess blast radius** - Which services import the package? Query SBOM DB.
* **Patch or pin** - If fix exists: update and redeploy. No fix - pin previous safe version or hot-patch.
* **Runtime mitigations** - Feature flags, WAF rules, sandboxing.
* **Audit logs** - Look for IOC patterns (malicious post-install script calls, unexpected outbound traffic).
* **Post-mortem** - Improve monitoring rule (CVE watch), tighten publish pipeline, upstream fix contribution.

!!! info "External Resources"
    - [OWASP Dependency-Check](https://owasp.org/www-project-dependency-check/) (OWASP)
    - [SLSA Supply Chain Levels](https://slsa.dev/) (OpenSSF)
    - [Sigstore - Software Supply Chain Security](https://www.sigstore.dev/) (Linux Foundation)

## Code Review & Linters

* Fail build on **new** high-severity findings, not historical debt.
* Cache tool installs to keep pipeline sub-5 min.

---

**Metrics to watch**

* PR review turnaround < 24 h (< 4 h for hotfixes)
* Zero open critical linter/SAST findings in main branch
* Secret-scan MTTR < 30 min
* Code-coverage of security lint rules >= 90 %

---

**Common anti-patterns**

* Rubber-stamp "LGTM" approvals.
* Disabling linter rules project-wide.
* Skipping review for "trivial" changes.
* Treating SAST findings as someone else's problem.

Embed these practices early - let linters catch the predictable, freeing reviewers for deep thinking.

!!! info "External Resources"
    - [OWASP Code Review Guide](https://owasp.org/www-project-code-review-guide/) (OWASP)
    - [Semgrep - Static Analysis](https://semgrep.dev/) (Semgrep)
    - [GitHub Code Scanning with CodeQL](https://docs.github.com/en/code-security/code-scanning/introduction-to-code-scanning/about-code-scanning-with-codeql) (GitHub)

## Infrastructure as Code Security

IaC templates (Terraform, CloudFormation, Pulumi, Helm charts) codify infrastructure, making security auditable and repeatable - but also making misconfigurations reproducible at scale.

**Common IaC security risks**

- Overly permissive IAM policies or security group rules in templates
- Hardcoded secrets or credentials in configuration files
- Public-facing resources created by default (S3 buckets, databases)
- Missing encryption-at-rest or in-transit settings
- Drift between declared state and actual deployed state

**Scanning tools**

| Tool | Focus area |
|------|-----------|
| **tfsec / Trivy** | Terraform misconfigurations |
| **Checkov** | Multi-framework (Terraform, CloudFormation, K8s, Helm) |
| **KICS** | Multi-framework, query-based |
| **OPA/Conftest** | Policy-as-code for any structured data |

**Best practices**

- Scan IaC in CI/CD pipeline; block merges on critical findings.
- Use modules/templates with security defaults baked in.
- Tag all resources for ownership and cost tracking.
- Enforce least-privilege in all IAM definitions.
- Detect and remediate drift with periodic `terraform plan` or equivalent.

!!! info "External Resources"
    - [Checkov - Policy-as-Code for IaC](https://www.checkov.io/) (Bridgecrew)
    - [tfsec - Terraform Static Analysis](https://aquasecurity.github.io/tfsec/) (Aqua Security)
    - [OWASP IaC Security Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Infrastructure_as_Code_Security_Cheat_Sheet.html) (OWASP)

## Container & Image Scanning in CI/CD

Container images introduce a large dependency surface. Scanning must happen at build time, in registries, and at runtime.

**Scanning stages**

| Stage | What to check | Tools |
|-------|--------------|-------|
| **Build** | Base image CVEs, added packages, secrets in layers | Trivy, Grype, Snyk Container |
| **Registry** | New CVEs against stored images, image age/staleness | Harbor, AWS ECR scanning, GCP Artifact Analysis |
| **Admission** | Policy compliance before pod scheduling | OPA Gatekeeper, Kyverno |
| **Runtime** | Unexpected processes, file changes, network connections | Falco, Sysdig, Tetragon |

**Best practices**

- Use minimal base images (`distroless`, `alpine`) to reduce attack surface.
- Pin base image digests, not just tags.
- Rebuild images on a schedule to pick up base-image patches.
- Sign images with Cosign/Notary; verify signatures in admission controllers.
- Never run containers as root; use `USER` directive and `runAsNonRoot` in K8s.

!!! info "External Resources"
    - [Trivy - Comprehensive Security Scanner](https://trivy.dev/) (Aqua Security)
    - [Cosign - Container Signing](https://docs.sigstore.dev/cosign/overview/) (Sigstore)
    - [NSA/CISA Kubernetes Hardening Guide](https://media.defense.gov/2022/Aug/29/2003066362/-1/-1/0/CTR_KUBERNETES_HARDENING_GUIDANCE_1.2_20220829.PDF) (NSA/CISA)

## Policy as Code (OPA, Sentinel)

Policy as code encodes security and compliance rules in machine-readable formats, enabling automated enforcement across infrastructure and applications.

**Key frameworks**

| Framework | Language | Primary use |
|-----------|----------|-------------|
| **Open Policy Agent (OPA)** | Rego | Kubernetes admission, API authorization, Terraform, CI/CD gates |
| **HashiCorp Sentinel** | Sentinel | Terraform Enterprise/Cloud policy enforcement |
| **Kyverno** | YAML | Kubernetes-native policy engine |
| **Cedar** | Cedar | AWS Verified Permissions, fine-grained authorization |

**Common policy categories**

- **Resource constraints**: enforce naming conventions, required tags, size limits
- **Security baselines**: require encryption, deny public access, enforce MFA
- **Compliance**: enforce data residency, retention policies, audit logging
- **Cost controls**: limit instance types, prevent over-provisioning

**Best practices**

- Store policies in version control alongside infrastructure code.
- Test policies with unit tests before enforcing.
- Start in audit/warn mode; switch to enforcement after validation.
- Layer policies: broad guardrails at org level, specific rules per team.
- Generate compliance reports from policy decisions for auditors.

!!! info "External Resources"
    - [Open Policy Agent Documentation](https://www.openpolicyagent.org/docs/latest/) (OPA)
    - [Kyverno Documentation](https://kyverno.io/docs/) (Kyverno)
    - [HashiCorp Sentinel Documentation](https://developer.hashicorp.com/sentinel/docs) (HashiCorp)
