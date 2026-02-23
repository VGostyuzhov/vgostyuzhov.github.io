# Identity Management

## Identity Lifecycle (Provisioning, Management, Deprovisioning)

The identity lifecycle covers every stage of a digital identity from creation to removal. Failures at any stage - especially deprovisioning - create security risk.

**Lifecycle phases:**

| Phase | Activities | Security Concern |
|-------|-----------|-----------------|
| **Provisioning** | Account creation, initial credential issuance, group membership, role assignment | Over-provisioned access at creation; cloned permissions from existing users |
| **Management** | Password resets, role changes, access requests, MFA enrollment, attribute updates | Permission creep; stale group memberships; lost MFA devices |
| **Deprovisioning** | Account disable/delete, credential revocation, session termination, access removal | Orphaned accounts; delayed revocation; forgotten service accounts |

**Provisioning best practices:**

- Use automated provisioning from HR system (joiner-mover-leaver workflow)
- Apply role templates based on job function rather than copying an existing user
- Require MFA enrollment before granting access to production systems
- Document all provisioned access with justification and approver

**Deprovisioning risks:**

- Average organisation takes 7+ days to fully deprovision a departed employee
- Orphaned accounts are used in 34% of insider threat incidents
- Service accounts outlive the humans who created them
- SaaS applications outside SSO are frequently missed during offboarding

**Deprovisioning checklist:**

1. Disable account in IdP (immediately terminates SSO sessions)
2. Revoke all OAuth tokens and API keys
3. Remove from all groups and role bindings
4. Transfer ownership of shared resources (repos, documents, cloud projects)
5. Archive mailbox and preserve data for legal hold if required
6. Audit for personal devices with cached credentials
7. Review service accounts created by the departing user

!!! info "External Resources"
    - [NIST SP 800-63 - Digital Identity Guidelines](https://pages.nist.gov/800-63-3/) (NIST)
    - [Identity Lifecycle Management - Gartner](https://www.gartner.com/en/information-technology/glossary/identity-lifecycle-management) (Gartner)
    - [OWASP Identity Management Guide](https://owasp.org/www-project-web-security-testing-guide/latest/4-Web_Application_Security_Testing/03-Identity_Management_Testing/) (OWASP)

## Access Control Models (RBAC, ABAC, ReBAC)

Access control models determine how permission decisions are made. The choice of model affects scalability, auditability, and security granularity.

**RBAC (Role-Based Access Control):**

- Permissions assigned to roles; users assigned to roles
- Simple hierarchy: User -> Role -> Permission -> Resource
- Well-suited for organisations with clear job functions
- Limitations: role explosion in complex environments; difficulty expressing conditional access

**ABAC (Attribute-Based Access Control):**

- Decisions based on attributes of the subject, resource, action, and environment
- Policy example: "Allow if user.department == 'engineering' AND resource.classification == 'internal' AND time.hour BETWEEN 9 AND 17"
- Flexible and fine-grained; no role explosion
- More complex to implement and audit; requires attribute infrastructure

**ReBAC (Relationship-Based Access Control):**

- Decisions based on relationships between entities in a graph
- "User A can edit Document X because A is a member of Team B, and Team B owns Project C, which contains Document X"
- Used by Google Zanzibar, AuthZed SpiceDB, Ory Keto
- Scales well for document-sharing and social platform patterns

**Comparison:**

| Property | RBAC | ABAC | ReBAC |
|----------|------|------|-------|
| Granularity | Coarse (role-level) | Fine (attribute-level) | Fine (relationship-level) |
| Scalability | Role explosion risk | Scales with attributes | Scales with graph |
| Audit | Easy (who has which role) | Complex (attribute evaluation) | Graph traversal |
| Best for | Enterprise, cloud IAM | Healthcare, government, dynamic policies | Document sharing, social platforms, SaaS |
| Examples | AWS IAM, K8s RBAC, Azure RBAC | XACML, Cedar, OPA | Google Zanzibar, SpiceDB |

**Hybrid approaches:** Most production systems combine models. Cloud IAM is primarily RBAC with ABAC conditions (IP range, time, MFA status).

!!! info "External Resources"
    - [NIST RBAC Model](https://csrc.nist.gov/projects/role-based-access-control) (NIST)
    - [NIST SP 800-162 - ABAC Guide](https://csrc.nist.gov/publications/detail/sp/800-162/final) (NIST)
    - [Google Zanzibar Paper](https://research.google/pubs/pub48190/) (Google Research)

## Service Accounts vs User Accounts

| Property | User Account | Service Account |
|----------|-------------|----------------|
| **Represents** | Human identity | Application, workload, automation |
| **Authentication** | Password + MFA, SSO, biometrics | API key, certificate, workload identity token |
| **Lifecycle** | Tied to employment (HR-driven) | Tied to application lifecycle (often forgotten) |
| **Session** | Interactive (console, browser, CLI) | Non-interactive (API calls, cron jobs) |
| **MFA** | Required for humans | Not applicable (use certificate/token-based auth) |
| **Risk** | Account takeover, credential phishing | Key leakage, over-permissioning, lateral movement |

**Service account security challenges:**

- Created for one-time automation scripts and never deleted
- Shared across multiple services (blast radius amplification)
- Keys exported to developer laptops, CI/CD configs, or code repos
- Permissions granted broadly ("just make it work") and never right-sized
- No MFA equivalent - authentication strength depends entirely on key protection

**Best practices:**

- Inventory all service accounts with ownership and purpose documentation
- Use cloud workload identity (keyless) wherever possible
- Scope permissions to the minimum required resources
- Set expiration dates on service account keys
- Monitor usage patterns; alert on anomalous activity
- Automate rotation; treat manual rotation as a vulnerability
- For detailed coverage, see [Cloud IAM - Service Accounts](../infrastructure-cloud/iam.md#service-accounts-workload-identity)

!!! info "External Resources"
    - [AWS Service Account Best Practices](https://docs.aws.amazon.com/IAM/latest/UserGuide/best-practices.html#bp-workloads-use-roles) (AWS)
    - [GCP Service Account Security](https://cloud.google.com/iam/docs/best-practices-service-accounts) (Google Cloud)
    - [NIST SP 800-63B - Authentication and Lifecycle](https://pages.nist.gov/800-63-3/sp800-63b.html) (NIST)

## Privileged Access Management (PAM)

PAM controls and monitors access to privileged accounts - root, admin, DBA, cloud owner - which are the highest-value targets for attackers.

**PAM capabilities:**

| Capability | Description |
|-----------|------------|
| **Credential vaulting** | Store privileged passwords in an encrypted vault; check out on demand |
| **Session recording** | Record and audit all privileged sessions (SSH, RDP, database) |
| **Just-in-time (JIT) access** | Grant temporary elevated access with approval workflow |
| **Password rotation** | Automatically rotate privileged passwords after each use or on schedule |
| **Least privilege enforcement** | Grant only the specific admin operations needed, not full root/admin |
| **Break-glass access** | Emergency access procedure with enhanced logging and post-incident review |

**PAM tools:**

| Tool | Type |
|------|------|
| CyberArk | Enterprise PAM (vault, session management, analytics) |
| HashiCorp Boundary | Open-source secure remote access (identity-based) |
| BeyondTrust | Enterprise PAM and endpoint privilege management |
| AWS SSO / IAM Identity Center | Cloud-native temporary role assumption |
| GCP PAM | Just-in-time privileged access for GCP |
| Teleport | Open-source access plane for SSH, K8s, databases |

**Operational practices:**

- Eliminate standing privileged access where possible (JIT over permanent admin)
- Require MFA and approval for all privileged sessions
- Record all privileged sessions for audit and incident investigation
- Alert on off-hours privileged access, access from unusual locations, or failed attempts
- Conduct regular privileged access reviews (quarterly minimum)
- Separate administrative and daily-use accounts (no admin access from regular workstations)

!!! info "External Resources"
    - [NIST SP 800-53 - AC-6 Least Privilege](https://csrc.nist.gov/publications/detail/sp/800-53/rev-5/final) (NIST)
    - [CyberArk PAM Documentation](https://docs.cyberark.com/) (CyberArk)
    - [HashiCorp Boundary Documentation](https://developer.hashicorp.com/boundary/docs) (HashiCorp)

## Federated Identity

Federation allows users to authenticate with one organisation's identity provider (IdP) and access resources in another organisation's service provider (SP), without creating separate accounts.

**Federation protocols:**

| Protocol | Format | Use Case |
|----------|--------|----------|
| **SAML 2.0** | XML assertions | Enterprise SSO (legacy and established) |
| **OIDC** | JWT (ID token) | Modern web/mobile SSO, cloud federation |
| **WS-Federation** | XML/SOAP | Microsoft ecosystem (ADFS) |
| **SCIM** | JSON over REST | Cross-domain user provisioning (not auth, but identity sync) |

**Trust model:**

- The relying party (SP) trusts the IdP to authenticate users correctly
- Trust is established through metadata exchange (certificates, endpoints)
- If the IdP is compromised, all federated services are compromised
- The SolarWinds/SAML golden ticket attack demonstrated this risk

**Federation security considerations:**

- Validate all token claims (`iss`, `aud`, `exp`, `nbf`) strictly
- Map external groups/roles to local permissions with least privilege
- Implement session limits and re-authentication for sensitive operations
- Monitor for anomalous federation events (unusual IdP, unexpected claims)
- Have a plan to rapidly disable federation trust if IdP is compromised
- For protocol details, see [Authentication Section](../authentication/index.md)

!!! info "External Resources"
    - [SAML 2.0 Overview - OASIS](https://www.oasis-open.org/standard/saml/) (OASIS)
    - [OpenID Connect Specification](https://openid.net/specs/openid-connect-core-1_0.html) (OpenID Foundation)
    - [SCIM Protocol - RFC 7644](https://datatracker.ietf.org/doc/html/rfc7644) (IETF)

## Identity Governance & Compliance

Identity governance ensures that access rights are appropriate, documented, and compliant with regulatory requirements.

**Core governance activities:**

| Activity | Description | Frequency |
|----------|------------|-----------|
| **Access certification** | Managers review and confirm/revoke user access rights | Quarterly or semi-annually |
| **Segregation of duties (SoD)** | Prevent conflicting permissions (e.g., create and approve payments) | Continuous enforcement |
| **Policy enforcement** | Automated rules (password complexity, session timeout, MFA requirements) | Continuous |
| **Access request workflow** | Standardised process for requesting, approving, and provisioning access | On-demand |
| **Compliance reporting** | Generate evidence for auditors (SOC 2, ISO 27001, GDPR) | As required |
| **Entitlement review** | Analyse permission usage; identify and remove unused entitlements | Quarterly |

**Compliance requirements by framework:**

| Framework | Identity-related Requirements |
|-----------|------------------------------|
| **SOC 2** | Access provisioning/deprovisioning, MFA, access reviews, logging |
| **ISO 27001** | A.9 Access Control (user access management, system access, responsibilities) |
| **PCI DSS** | Requirement 7 (need-to-know), Requirement 8 (unique ID, MFA), Requirement 10 (logging) |
| **GDPR** | Right to erasure (deprovisioning), data access controls, consent management |
| **HIPAA** | Unique user identification, emergency access, automatic logoff, audit controls |

**Identity governance tools:**

- SailPoint, Saviynt - enterprise identity governance and administration (IGA)
- Okta, Azure AD (Entra ID) - cloud identity with built-in governance features
- Veza - cloud-native authorization and access visibility

**Metrics:**

- Time from termination to full access revocation
- Percentage of users with access certified in last quarter
- Number of SoD violations detected and resolved
- Orphaned accounts count and remediation rate

!!! info "External Resources"
    - [NIST SP 800-53 - Access Control Family](https://csrc.nist.gov/publications/detail/sp/800-53/rev-5/final) (NIST)
    - [ISO 27001 - A.9 Access Control](https://www.iso.org/standard/27001) (ISO)
    - [Identity Governance - Gartner](https://www.gartner.com/reviews/market/identity-governance-and-administration) (Gartner)
