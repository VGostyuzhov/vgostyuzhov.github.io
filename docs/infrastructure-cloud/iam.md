# Cloud Identity & Access Management

## IAM Fundamentals & Principles

IAM (Identity and Access Management) controls who can do what to which resources in cloud environments. In cloud, identity is the perimeter.

**Core IAM components:**

| Component | Description |
|-----------|------------|
| **Principal** | Entity requesting access (user, service account, role, federated identity) |
| **Action/Permission** | Operation being performed (read, write, delete, admin) |
| **Resource** | Target of the action (VM, bucket, database, API) |
| **Policy** | Rules binding principals to permissions on resources |
| **Condition** | Contextual constraints (source IP, time, MFA status, tags) |

**IAM across providers:**

| Concept | AWS | GCP | Azure |
|---------|-----|-----|-------|
| **Identity store** | IAM Users, Roles | Google Workspace, Cloud Identity | Azure AD (Entra ID) |
| **Permission unit** | IAM Policy (JSON) | IAM Role binding | Role assignment (RBAC) |
| **Service identity** | IAM Role (assumed by services) | Service Account | Managed Identity |
| **Org hierarchy** | Organization -> OU -> Account | Organization -> Folder -> Project | Tenant -> Management Group -> Subscription -> Resource Group |
| **Policy inheritance** | SCPs flow down OUs | Org policies inherited by folders/projects | Azure Policy inherited by management groups |

**Principle of least privilege:**

- Grant minimum permissions required for the task
- Prefer specific actions over wildcards (`s3:GetObject` not `s3:*`)
- Use time-bounded access where possible (temporary credentials, just-in-time access)
- Regularly review and revoke unused permissions (access advisor, policy analyser)
- Permissions decay silently - automation and periodic recertification are essential

!!! info "External Resources"
    - [AWS IAM Best Practices](https://docs.aws.amazon.com/IAM/latest/UserGuide/best-practices.html) (AWS)
    - [GCP IAM Overview](https://cloud.google.com/iam/docs/overview) (Google Cloud)
    - [Azure RBAC Documentation](https://learn.microsoft.com/en-us/azure/role-based-access-control/overview) (Microsoft)

## Common IAM Misconfigurations

IAM misconfigurations are the leading cause of cloud security incidents. Identity is the cloud perimeter, and a misconfigured policy can expose an entire account.

**High-risk misconfigurations:**

| Misconfiguration | Risk | Detection |
|-----------------|------|-----------|
| **Wildcard permissions** (`*:*`) | Full account takeover | IAM Access Analyzer, ScoutSuite, Prowler |
| **Overly permissive roles** | Lateral movement, data exfiltration | Unused permission analysis, access advisors |
| **Long-lived access keys** | Credential theft, no rotation | Key age monitoring, enforce rotation policy |
| **No MFA on privileged accounts** | Account takeover via password compromise | IAM credential report, SCP enforcement |
| **Public resource policies** | Data exposure | Config rules, CSPM tools |
| **Cross-account trust without conditions** | Unauthorized external access | External principal analysis |
| **Default service account usage** | Over-permissioned compute instances | Audit default SA bindings |
| **Unused accounts/roles** | Dormant attack surface | Last-used analysis, automated cleanup |

**Permission creep:**

- Permissions accumulate over time as projects change
- Developers request broad access for debugging, then never revoke
- Automated tools (Terraform, CI/CD) often use overprivileged service accounts
- Mitigation: regular access reviews, automated permission right-sizing, just-in-time access

!!! info "External Resources"
    - [AWS IAM Access Analyzer](https://docs.aws.amazon.com/IAM/latest/UserGuide/what-is-access-analyzer.html) (AWS)
    - [GCP IAM Recommender](https://cloud.google.com/iam/docs/recommender-overview) (Google Cloud)
    - [Prowler - Cloud Security Assessment](https://github.com/prowler-cloud/prowler) (Prowler)

## Service Accounts & Workload Identity

Service accounts provide identity for non-human workloads (applications, VMs, CI/CD pipelines, containers).

**Service account risks:**

- Often over-permissioned because "the service needs it"
- Keys exported and stored in code repos, config files, or CI/CD variables
- Shared across multiple services (blast radius amplification)
- Used for lateral movement: compromise one service, impersonate its SA to reach others
- GCPloit and similar tools automate privilege escalation via service accounts

**Workload identity (keyless authentication):**

| Provider | Mechanism | How it works |
|----------|-----------|-------------|
| **AWS** | IAM Roles for EC2/ECS/EKS (IRSA) | Instance metadata service provides temporary credentials |
| **GCP** | Workload Identity, Workload Identity Federation | K8s SA mapped to GCP SA; external IdP tokens exchanged for GCP tokens |
| **Azure** | Managed Identity (system/user-assigned) | Azure platform injects credentials; no key management |

**Best practices:**

- Prefer workload identity over exported keys whenever possible
- One service account per service/workload (not shared)
- Scope permissions to specific resources, not project/account-wide
- Disable service account key creation via org policy where feasible
- Monitor service account usage: alert on SA actions from unexpected IPs or regions
- Rotate keys automatically if key-based authentication is unavoidable
- Use short-lived tokens (1 hour or less) via token exchange or impersonation

!!! info "External Resources"
    - [AWS IAM Roles for Service Accounts (IRSA)](https://docs.aws.amazon.com/eks/latest/userguide/iam-roles-for-service-accounts.html) (AWS)
    - [GCP Workload Identity](https://cloud.google.com/kubernetes-engine/docs/concepts/workload-identity) (Google Cloud)
    - [Azure Managed Identities](https://learn.microsoft.com/en-us/azure/active-directory/managed-identities-azure-resources/overview) (Microsoft)

## Role Design & Least Privilege

**Role design principles:**

- **Functional roles** - align with job functions (developer, SRE, security analyst, auditor)
- **Environment separation** - different roles for dev/staging/prod access
- **Resource scoping** - bind roles to specific resources, not entire accounts/projects
- **Temporal scoping** - just-in-time (JIT) access for privileged operations

**Role design patterns:**

| Pattern | Description | Use Case |
|---------|------------|----------|
| **Predefined roles** | Provider-managed roles with fixed permissions | Standard workloads (viewer, editor, admin) |
| **Custom roles** | User-defined permission sets | Least-privilege for specific workflows |
| **Permission boundaries** | Maximum permission ceiling (AWS) | Delegate role creation without risk |
| **Deny policies** | Explicit deny overrides allow | Guardrails (no public resources, no key export) |
| **Service Control Policies (SCPs)** | Org-level permission boundaries (AWS) | Enforce baseline across all accounts |
| **Organization Policies** | Org-level constraints (GCP) | Disable SA key creation, enforce location |

**Just-in-time (JIT) access:**

- Privileged access granted on-demand for a limited duration
- Requires approval workflow and audit trail
- Tools: AWS SSO with temporary role assumption, GCP PAM, CyberArk, HashiCorp Boundary
- Reduces standing privilege - if credentials are compromised, they may already be expired

!!! info "External Resources"
    - [AWS Permissions Boundaries](https://docs.aws.amazon.com/IAM/latest/UserGuide/access_policies_boundaries.html) (AWS)
    - [GCP Organization Policy Service](https://cloud.google.com/resource-manager/docs/organization-policy/overview) (Google Cloud)
    - [Principle of Least Privilege - NIST](https://csrc.nist.gov/glossary/term/least_privilege) (NIST)

## Cross-Account & Federation

**Cross-account access:**

- AWS: `sts:AssumeRole` with trust policies specifying allowed source accounts
- GCP: IAM bindings can reference identities from other projects/organizations
- Azure: cross-subscription role assignments, Azure Lighthouse for managed service providers

**Security controls for cross-account:**

- Always specify conditions: `aws:PrincipalOrgID`, external ID for third parties
- Use dedicated cross-account roles with minimal permissions
- Log all cross-account access in CloudTrail/Audit Logs
- Regularly audit trust relationships; remove stale external access

**Identity federation:**

| Federation Type | Protocol | Use Case |
|----------------|----------|----------|
| **Workforce federation** | SAML 2.0, OIDC | Employees accessing cloud console/CLI via corporate IdP |
| **Workload federation** | OIDC, SPIFFE | CI/CD pipelines, on-prem services accessing cloud without long-lived keys |
| **Social federation** | OIDC | Customer-facing apps (Login with Google/GitHub) |

**Federation security considerations:**

- Trust is placed in the external IdP; if it is compromised, all federated access is compromised
- Validate token claims (`aud`, `iss`, `sub`) strictly
- Map external identities to cloud roles with least privilege
- Use attribute-based conditions (group membership, repository name for CI/CD)
- Monitor for anomalous federation token usage (unusual source, unexpected claims)

!!! info "External Resources"
    - [AWS Cross-Account Access](https://docs.aws.amazon.com/IAM/latest/UserGuide/tutorial_cross-account-with-roles.html) (AWS)
    - [GCP Workload Identity Federation](https://cloud.google.com/iam/docs/workload-identity-federation) (Google Cloud)
    - [Azure External Identities](https://learn.microsoft.com/en-us/azure/active-directory/external-identities/) (Microsoft)

## IAM for Microservices

In microservice architectures, every service is a principal that must be authenticated and authorised.

**Service-to-service authentication:**

| Method | Description | Pros/Cons |
|--------|------------|-----------|
| **mTLS** | Mutual TLS with service certificates | Strong identity; certificate management overhead |
| **JWT** | Signed tokens issued by identity service | Stateless; must validate claims and expiry |
| **SPIFFE/SPIRE** | Standard for service identity (SVIDs) | Vendor-neutral; integrates with service mesh |
| **Cloud workload identity** | Platform-injected credentials | No key management; cloud-specific |

**Authorization patterns:**

- **API gateway** - centralised auth at the edge; services trust the gateway's identity header
- **Service mesh** - sidecar proxies enforce mTLS and authorization policies (Istio AuthorizationPolicy)
- **Embedded** - each service validates tokens and checks permissions internally
- **Policy engine** - centralised policy (OPA) queried by services at decision time

**Key considerations:**

- Humans should not be in the runtime auth path for service-to-service calls
- Short-lived credentials reduce blast radius
- Each service should have its own identity (not shared service accounts)
- Propagate identity context (request principal) through the call chain for audit
- Defence in depth: combine network policies, service mesh auth, and application-level checks

!!! info "External Resources"
    - [SPIFFE Documentation](https://spiffe.io/docs/) (SPIFFE)
    - [Istio Security - Authorization Policy](https://istio.io/latest/docs/reference/config/security/authorization-policy/) (Istio)
    - [NIST SP 800-204 - Security for Microservices](https://csrc.nist.gov/publications/detail/sp/800-204/final) (NIST)

## Privilege Escalation in Cloud

Cloud privilege escalation occurs when an attacker gains higher permissions than initially compromised, often through IAM misconfigurations.

**Common escalation paths:**

| Vector | Provider | Technique |
|--------|----------|-----------|
| **IAM policy modification** | AWS/GCP/Azure | Attach admin policy to own user/role |
| **Role chaining** | AWS | Assume role A, which can assume role B (transitive trust) |
| **Service account impersonation** | GCP | `iam.serviceAccounts.getAccessToken` permission enables impersonation |
| **Instance metadata** | AWS | SSRF to `169.254.169.254` retrieves IAM role credentials |
| **Lambda/Function creation** | AWS | Create function with privileged role, invoke it |
| **Custom role creation** | GCP | Create role with `iam.roles.update`, grant self higher permissions |
| **Pass-role abuse** | AWS/GCP | Pass a high-privilege role to a new resource you control |
| **Storage bucket policy** | AWS/GCP | Modify bucket policy to grant self access to sensitive data |

**Metadata service exploitation:**

- AWS IMDSv1: simple GET to `http://169.254.169.254/latest/meta-data/iam/security-credentials/`
- AWS IMDSv2: requires PUT with hop-limit header (mitigates many SSRF vectors)
- GCP: `http://metadata.google.internal/computeMetadata/v1/` with `Metadata-Flavor: Google` header
- Always enforce IMDSv2 on AWS; restrict metadata access in GCP

**Detection and prevention:**

- Monitor for IAM policy changes (CloudTrail `PutRolePolicy`, `AttachUserPolicy`)
- Alert on new role assumptions from unexpected sources
- Use permission boundaries and SCPs to cap maximum permissions
- Audit `iam:PassRole`, `iam.serviceAccounts.actAs` grants
- Tools: Pacu (AWS exploitation), GCPBucketBrute, ScoutSuite, Prowler

!!! info "External Resources"
    - [Rhino Security Labs - AWS Privilege Escalation](https://rhinosecuritylabs.com/aws/aws-privilege-escalation-methods-mitigation/) (Rhino Security Labs)
    - [GCP Privilege Escalation - GitLab](https://about.gitlab.com/blog/2020/02/12/plundering-gcp-escalating-privileges-in-google-cloud-platform/) (GitLab)
    - [MITRE ATT&CK - Cloud Privilege Escalation](https://attack.mitre.org/tactics/TA0004/) (MITRE)
