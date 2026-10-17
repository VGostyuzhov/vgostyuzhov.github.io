# Cloud Identity & Access Management

This page teaches how a cloud request becomes an authorization decision, using AWS for worked mechanisms. Provider summaries are orientation; AWS policy evaluation rules do not transfer unchanged to Google Cloud or Azure.

**Prerequisites:** [principals, credentials, and trust boundaries](../authentication/index.md), [shared responsibility](virtualization.md#shared-responsibility-model), and [Kubernetes service accounts and access control](containers.md#kubernetes-security-fundamentals) for the workload examples. Continue with the [multi-account architecture](multi-account-security.md) and [scenario practice](../interview-prep/staff-principal.md).

## IAM Fundamentals & Principles

IAM (Identity and Access Management) controls which principal may perform an action on a resource under particular conditions. Authentication establishes the caller; authorization evaluates the requested operation. Network reachability and valid credentials alone do not grant permission.

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
| **Org hierarchy** | Organization -> Organizational unit (OU) -> Account | Organization -> Folder -> Project | Tenant -> Management Group -> Subscription -> Resource Group |
| **Governance constraints** | Service control policies (SCPs) restrict member-account principals | Organization policies constrain supported resource configurations | Azure Policy evaluates supported resource configurations |

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

## AWS Policy Evaluation: Grants and Permission Ceilings

Start with one concrete request: an assumed role calls `s3:GetObject` for `arn:aws:s3:::example-reports/daily.csv`. An **Amazon Resource Name (ARN)** identifies an AWS resource or principal. The request context includes the authenticated session, action, resource, and available condition keys. AWS evaluates applicable policies; a missing condition key is not proof that its condition was satisfied.

An **identity policy** on a role grants actions to that role. A **resource policy**, such as a bucket policy, grants or denies access at the resource. An **explicit deny** is a matching `Deny` statement. An **implicit deny** means the required permission was not granted. A matching explicit deny wins over an allow; an implicit deny must be interpreted using the rules for that policy type and principal.

A **permissions boundary** caps permissions granted by a user or role's identity policies. An **SCP** caps permissions of principals in covered organization member accounts. Neither grants permissions. A **session policy** can further restrict the identity-policy permissions of a particular temporary session. For an ordinary identity-policy grant, adding an allow to the role cannot override an applicable boundary or SCP restriction.

Synthetic decision trace; assume no separate resource-policy grant, no session policy, and no other restrictions:

```text
# Role identity policy: Allow s3:GetObject and s3:DeleteObject on example-reports/*
# Boundary:            Allow s3:GetObject on example-reports/*
# Effective SCPs:      Permit these S3 actions
GetObject daily.csv    -> ALLOW: identity grant survives the boundary and SCPs
DeleteObject daily.csv -> DENY: boundary does not allow deletion (implicit deny)
PutObject daily.csv    -> DENY: no identity grant; an SCP allow cannot create one

# Add an SCP Deny for s3:GetObject on the same object:
GetObject daily.csv    -> DENY: explicit deny overrides the identity allow
```

Do not turn this trace into a universal intersection formula. Within one account, resource policies can provide a separate permission grant. In particular, a grant directly to an IAM user ARN or an assumed-role **session ARN** has different boundary behavior from a grant to a role ARN. A same-account resource grant directly to a role session is not limited by implicit denies in identity, boundary, or session policies. Applicable explicit denies and organization controls still matter. A boundary alone therefore cannot establish that every possible resource grant is harmless.

Resource control policies (**RCPs**) are another Organizations policy type: they constrain access to supported resources in member accounts, whereas SCPs constrain covered principals. Service-specific rules also matter; for example, an encrypted object can require authorization to its encryption key as well as to the object. The examples below state when these additional checks are outside their assumptions.

| Policy | Question it answers | Does an allow grant access by itself? |
|---|---|---|
| Identity policy | May this identity request this action/resource? | Supplies a grant, subject to applicable limits |
| Resource policy | Which principals may use this resource? | Can supply a grant; cross-account evaluation also checks the caller |
| Role trust policy | Who may obtain a session for this role? | Authorizes assumption, not the later application action |
| Permissions boundary | Which identity-policy permissions may this entity receive? | No |
| SCP | Which actions may covered member-account principals perform? | No |
| Session policy | Which identity-policy permissions survive for this session? | No |

!!! info "External Resources"
    - [AWS policy evaluation logic](https://docs.aws.amazon.com/IAM/latest/UserGuide/reference_policies_evaluation-logic.html) (AWS)
    - [Effective permissions with boundaries and resource policies](https://docs.aws.amazon.com/IAM/latest/UserGuide/access_policies_boundaries.html) (AWS)
    - [Enforcement logic, including SCPs and RCPs](https://docs.aws.amazon.com/IAM/latest/UserGuide/reference_policies_evaluation-logic_policy-eval-denyallow.html) (AWS)

### SCP Inheritance and Scope

AWS Organizations groups accounts into OUs. For a covered principal, the SCPs must permit an action at every level on the path from the organization root through its OUs to its account. Multiple allows at a single level can combine; a deny at any applicable level wins. Keeping `FullAWSAccess` alongside a deny policy is a common way to prohibit selected actions while leaving others available for IAM to grant. Removing that allow without a replacement can block all actions at that level.

SCPs cover member-account IAM users and roles, including the member account's root user, but not identities in the management account or service-linked roles used by AWS services. A delegated-administrator account is still a member account. An SCP on a bucket owner's account does not itself restrict an external principal accessing that bucket directly; resource policies and, where supported, RCPs address that resource-side exposure.

For example, a Production OU deny on `s3:DeleteObject` cannot be reversed by an allow attached to a child account. An approved exception must change the applicable deny's scope or the account's policy path, with the broader consequences reviewed. The [architecture rollout](multi-account-security.md#staged-enforcement-and-expiring-exceptions) shows how to avoid turning that exception into a permanent bypass.

!!! info "External Resources"
    - [SCP evaluation through roots, OUs, and accounts](https://docs.aws.amazon.com/organizations/latest/userguide/orgs_manage_policies_scps_evaluation.html) (AWS)
    - [SCP scope and exceptions](https://docs.aws.amazon.com/organizations/latest/userguide/orgs_manage_policies_scps.html) (AWS)

## Common IAM Misconfigurations

IAM mistakes can turn a single stolen credential into access across resources or accounts. Review effective access paths, including who can change policies, rather than only searching for wildcard strings.

**High-risk misconfigurations:**

| Misconfiguration | Risk | Detection |
|-----------------|------|-----------|
| **Wildcard permissions** (`*:*`) | Full account takeover | IAM Access Analyzer, ScoutSuite, Prowler |
| **Overly permissive roles** | Lateral movement, data exfiltration | Unused permission analysis, access advisors |
| **Long-lived access keys** | Credential theft, no rotation | Key age monitoring, enforce rotation policy |
| **No multi-factor authentication (MFA) for privileged humans** | Password compromise enables privileged sign-in | Verify the actual workforce sign-in and emergency-access paths |
| **Public resource policies** | Data exposure | Config rules, CSPM tools |
| **Broad or stale cross-account trust** | Unintended principals obtain sessions | External principal analysis and negative assumption tests |
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

A workload identity lets software obtain credentials for its own role instead of sharing a human account or an exported access key. Temporary credentials still grant real authority: a compromised authorized process can use them until access is denied or they expire. Give each workload a separate identity and scope its permissions to the resources it needs.

### EC2 Metadata Credentials and IMDSv2

For an Amazon Elastic Compute Cloud (**EC2**) instance, an instance profile associates an IAM role with the instance. The AWS Instance Metadata Service (**IMDS**) exposes temporary role credentials to software on that instance. A supported AWS Software Development Kit (**SDK**) obtains and refreshes credentials; application code need not store a permanent key.

IMDSv2 adds a token exchange before metadata retrieval. This annotated protocol fragment is illustrative; the token and returned credentials are omitted:

```text
# Request an IMDS session token from the local instance.
PUT http://169.254.169.254/latest/api/token
X-aws-ec2-metadata-token-ttl-seconds: 300

# Use the returned token to retrieve credentials for the attached role.
GET http://169.254.169.254/latest/meta-data/iam/security-credentials/ExampleAppRole
X-aws-ec2-metadata-token: <IMDS-session-token>

# The response contains an access key ID, secret key, session token, and expiration.
# The 300-second IMDS token lifetime is NOT the AWS credential lifetime.
```

Requiring IMDSv2 rejects tokenless metadata requests and impedes server-side request forgery (**SSRF**) primitives that can only issue a simple GET. The `HttpPutResponseHopLimit` instance option separately sets the IP hop limit of the token response; it is not a PUT request header. Extra network hops in container environments can affect token delivery, so verify the intended network path before changing it.

A process able to execute code on the instance, or an SSRF primitive able to complete the required exchange, may still retrieve usable role credentials. Enforce IMDSv2, restrict metadata reachability for workloads that do not need it, and minimize the node role. These controls supplement fixing the SSRF or process compromise; they do not remove its consequences.

!!! info "External Resources"
    - [How IMDSv2 works](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/configuring-instance-metadata-service.html) (AWS)
    - [Metadata options and response hop limits](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/configuring-instance-metadata-options.html) (AWS)

### EKS IRSA: Exchange a Service-Account Token

IAM Roles for Service Accounts (**IRSA**) is a federation mechanism for Amazon Elastic Kubernetes Service (**EKS**). It does not retrieve the pod's role credentials from EC2 metadata.

1. The cluster issues a signed, projected service-account token to the pod. Its subject names the Kubernetes namespace and service account; its audience identifies the intended recipient.
2. A supported SDK reads the configured token file and role ARN, then presents the token to AWS Security Token Service (**STS**) using `AssumeRoleWithWebIdentity`.
3. STS validates the token against the configured OpenID Connect (**OIDC**) identity provider and evaluates the target role's trust conditions. OIDC is the federation protocol; the [JWT lesson](../authentication/jwt.md#acceptance-and-authorization) explains signed-token validation.
4. STS returns temporary role credentials. The SDK caches them until refresh is needed; subsequent AWS requests use these credentials and the role's permissions.

This synthetic trust policy uses a placeholder cluster issuer, account, and service account. The matching IAM OIDC provider and role permissions must exist separately. `aud` restricts the token recipient; `sub` restricts which workload may exchange it.

```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Principal": {
      "Federated": "arn:aws:iam::111122223333:oidc-provider/oidc.eks.eu-central-1.amazonaws.com/id/EXAMPLE"
    },
    "Action": "sts:AssumeRoleWithWebIdentity",
    "Condition": {
      "StringEquals": {
        "oidc.eks.eu-central-1.amazonaws.com/id/EXAMPLE:aud": "sts.amazonaws.com",
        "oidc.eks.eu-central-1.amazonaws.com/id/EXAMPLE:sub": "system:serviceaccount:payments:report-reader"
      }
    }
  }]
}
```

An otherwise valid token for `system:serviceaccount:development:report-reader` fails this trust policy. A token for the accepted service account can obtain the role even if the pod's code is malicious. Protect who can create pods using that service account, alter its workloads, or read its token. Check the actual SDK credential source so an older or explicitly configured provider does not use unexpected credentials.

IRSA does not isolate a compromised node. Pods that can reach IMDS may also reach the node role; `hostNetwork` pods have IMDS access. Restrict that path, limit the node role, and maintain the [container and host isolation controls](containers.md#container-runtime-security). EKS Pod Identity is a separate supported mechanism; its agent-based flow should not be described as IRSA.

!!! info "External Resources"
    - [IRSA mechanism and isolation limits](https://docs.aws.amazon.com/eks/latest/userguide/iam-roles-for-service-accounts.html) (AWS)
    - [Assign roles with audience and subject trust conditions](https://docs.aws.amazon.com/eks/latest/userguide/associate-service-account-role.html) (AWS)
    - [AssumeRoleWithWebIdentity](https://docs.aws.amazon.com/STS/latest/APIReference/API_AssumeRoleWithWebIdentity.html) (AWS)

### Workload Identity Across Providers

| Provider | Mechanism | Credential source or exchange |
|---|---|---|
| AWS | EC2 instance profile; ECS task role; EKS IRSA | Distinct instance metadata, task credential, and OIDC-to-STS flows |
| Google Cloud | Service accounts and Workload Identity Federation | Workload or external identity obtains short-lived Google credentials under configured trust |
| Azure | Managed identities and workload identity federation | Platform identity or federated assertion obtains access tokens without an application-managed static key |

Avoid sharing an identity among unrelated workloads. Restrict who can impersonate or attach it, monitor its actual use, and remove unused grants. Where a key is unavoidable, protect its distribution and automate rotation with an application rollout plan; rotation alone does not reduce excessive permissions.

!!! info "External Resources"
    - [Workload Identity Federation for GKE](https://cloud.google.com/kubernetes-engine/docs/concepts/workload-identity) (Google Cloud)
    - [Managed identities for Azure resources](https://learn.microsoft.com/en-us/entra/identity/managed-identities-azure-resources/overview) (Microsoft)
    - [Amazon ECS task IAM roles](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/task-iam-roles.html) (AWS)

## Role Design & Least Privilege

**Role design principles:**

- **Functional roles** - align with job functions (developer, SRE, security analyst, auditor)
- **Environment separation** - different roles for dev/staging/prod access
- **Resource scoping** - bind roles to specific resources, not entire accounts/projects
- **Temporal scoping** - just-in-time (JIT) access for privileged operations

**Role design patterns:**

| Pattern | Description | Use Case |
|---------|------------|----------|
| **Predefined roles** | Provider-maintained permission sets that can evolve | Standard workloads (viewer, editor, admin) |
| **Custom roles** | User-defined permission sets | Least-privilege for specific workflows |
| **Permissions boundaries** | Limit identity-policy grants (AWS) | Delegate role creation with protected boundary and trust controls |
| **Deny policies** | Explicit deny overrides allow | Guardrails (no public resources, no key export) |
| **Service control policies (SCPs)** | Organization constraints on covered AWS principals | Enforce a baseline in member accounts within their documented scope |
| **Organization Policies** | Org-level constraints (GCP) | Disable SA key creation, enforce location |

**Safe delegation:** A team allowed to create roles must be required to attach the approved boundary and prevented from deleting it, changing its policy versions, or modifying more privileged roles. Protect trust policies and `iam:PassRole` too: a role within a boundary can still be abused if an attacker can run code under it. Restrict sensitive resource-policy changes because the [grant path](#aws-policy-evaluation-grants-and-permission-ceilings) matters.

**Just-in-time (JIT) access** grants privilege only for an approved interval. A broker or access workflow authenticates the requester, checks their eligibility and requested account/action, records approval, and enables a constrained session. Temporary STS credentials alone do not prove approval: someone with permanent permission to assume a role can repeatedly obtain new sessions. Stop new issuance when approval ends, bound session duration, and define how existing sessions are denied when urgent revocation is required.

!!! info "External Resources"
    - [AWS Permissions Boundaries](https://docs.aws.amazon.com/IAM/latest/UserGuide/access_policies_boundaries.html) (AWS)
    - [GCP Organization Policy Service](https://cloud.google.com/resource-manager/docs/organization-policy/overview) (Google Cloud)
    - [Principle of Least Privilege - NIST](https://csrc.nist.gov/glossary/term/least_privilege) (NIST)

## Cross-Account & Federation

Cross-account role assumption delegates authority across an account boundary. Federation accepts identity evidence from an external identity provider (**IdP**) to issue a cloud session. Both require separating the right to obtain credentials from the permissions those credentials receive.

### Trace Caller, Trust, and Target Action

Assume account A (`111122223333`) holds `BuildPublisher`, and account B (`444455556666`) owns `ArtifactReader` and an S3 artifact bucket. The caller uses A's credentials to request a B role session. All ARNs and bucket names below are illustrative; these policies are a minimal scenario, not an account baseline.

**1. Caller grant in A.** Attach this identity policy to `BuildPublisher`. Its boundary, session policy, and A's applicable SCPs must also permit this assumption.

```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Action": "sts:AssumeRole",
    "Resource": "arn:aws:iam::444455556666:role/ArtifactReader"
  }]
}
```

**2. Trust in B.** `ArtifactReader` accepts the particular caller role. Trusting an account's root ARN would delegate principal selection to that account; it would not mean only its root user can assume the role.

```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Principal": {"AWS": "arn:aws:iam::111122223333:role/BuildPublisher"},
    "Action": "sts:AssumeRole"
  }]
}
```

**3. Permission after assumption.** Attach this policy to B's `ArtifactReader`. It grants object reads on one prefix, not writes or bucket listing. Assume the bucket belongs to B, has no conflicting resource policy, uses no customer-managed encryption key requiring another grant, and all applicable organization controls permit the read.

```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Action": "s3:GetObject",
    "Resource": "arn:aws:s3:::example-release-artifacts/releases/*"
  }]
}
```

The following trace distinguishes failures that would otherwise look like the same “access denied” incident:

```text
# A role session calls STS; STS evaluates A's caller permission and B's trust.
A:BuildPublisher -> AssumeRole(B:ArtifactReader) -> B role-session credentials

# Subsequent S3 request is made as B:ArtifactReader, not A:BuildPublisher.
B:ArtifactReader -> GetObject releases/v1.zip -> ALLOW under stated assumptions
B:ArtifactReader -> PutObject releases/v1.zip -> DENY: B role has no write grant

# Independent negative tests:
Remove A's AssumeRole allow   -> no B credentials issued
Replace caller with A:Other   -> B trust does not accept it
Deny AssumeRole in A's SCP    -> assumption denied, even with B trust
Deny GetObject in B's SCP     -> B session cannot read, even if assumption succeeded
```

A's permissions do not carry over as an additional permission set or ceiling for B's resulting session. B's role permissions and applicable limits govern that session. A caller-account SCP restricts the original assumption; the role-owning account's SCPs restrict later calls by its role session. Review both steps instead of assuming an SCP on B automatically blocks every outside caller.

Direct cross-account S3 access is a different flow: A remains the caller and both A's identity permissions and B's bucket policy must authorize it. If the object uses an AWS Key Management Service (**KMS**) key, evaluate the needed key permissions separately. A successful STS exchange is neither an S3 permission test nor a KMS permission test.

!!! info "External Resources"
    - [Cross-account evaluation and role trust](https://docs.aws.amazon.com/IAM/latest/UserGuide/reference_policies_evaluation-logic-cross-account.html) (AWS)
    - [AssumeRole permissions and resulting sessions](https://docs.aws.amazon.com/STS/latest/APIReference/API_AssumeRole.html) (AWS)
    - [SCP effects on principals and external access](https://docs.aws.amazon.com/organizations/latest/userguide/orgs_manage_policies_scps.html) (AWS)

### Confused Deputy and External ID Limits

A **confused deputy** is an authorized intermediary tricked into using its authority for the wrong party. Suppose a monitoring vendor serves customers Red and Blue. Red supplies Blue's role ARN to the vendor. If the vendor chooses the requested role without binding it to its authenticated customer, Blue's trust in the vendor can be misused.

The vendor should assign a unique external ID per customer account, retain that association in its own trusted configuration, and send the matching `ExternalId` when assuming a customer's role. Blue's trust policy must require that ID as well as the vendor principal. This fragment belongs in the trust statement accepting the vendor:

```json
{
  "Condition": {
    "StringEquals": {"sts:ExternalId": "vendor-assigned-customer-blue-7f42"}
  }
}
```

Red's request then carries Red's vendor-selected ID and fails Blue's condition. An external ID is not a password: a compromised vendor principal that can select Blue's ID may still obtain access. It does not establish least privilege, protect against vendor compromise, or help if an alternative trust statement accepts the vendor without the condition. Test correct, missing, and wrong IDs.

Other conditions solve different problems. `aws:PrincipalOrgID` can restrict an eligible principal to a particular AWS organization; it does not select a specific team or make every member account trustworthy. For AWS service-to-service confused-deputy risks, use the documented `aws:SourceArn`/`aws:SourceAccount` conditions where supported. A customer's external ID is not a substitute for checking the source resource in that different flow.

!!! info "External Resources"
    - [Third-party access and external IDs](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_roles_common-scenarios_third-party.html) (AWS)
    - [Cross-account and cross-service confused deputy prevention](https://docs.aws.amazon.com/IAM/latest/UserGuide/confused-deputy.html) (AWS)
    - [Global condition keys and their availability](https://docs.aws.amazon.com/IAM/latest/UserGuide/reference_policies_condition-keys.html) (AWS)

### Federation Claims, Issuance, and Outages

A cloud federation configuration retains the trusted issuer/provider, its verification configuration, and the mapping from accepted claims to roles. For OIDC, the exchange validates the signed token from that issuer, its audience and time validity, and the subject conditions selecting the intended workload. Signature success alone is insufficient: many unrelated workloads can receive valid tokens from the same issuer. The [IRSA example](#eks-irsa-exchange-a-service-account-token) shows exact audience/subject checks; the [CI design](../authentication/design-patterns.md#6-replace-a-ci-deployment-secret-with-workload-federation) explains repository and deployment-context restrictions.

Workforce federation similarly maps an authenticated employee to permitted accounts and roles. SAML (Security Assertion Markup Language) uses signed assertions with recipient, audience, and time checks; see [SAML validation](../authentication/saml.md#assertion-validation). Enforce MFA at the real workforce authentication path. Do not assume the IAM `aws:MultiFactorAuthPresent` key describes MFA performed by every external IdP or federation flow.

Three events have separate lifecycles: the IdP authentication, the assertion accepted by STS, and the resulting AWS session. Disabling the employee blocks new IdP authentication, but an already-issued valid assertion may remain exchangeable until it expires. Blocking the cloud trust/provider path can stop those exchanges after the change takes effect. Neither action automatically erases already issued AWS credentials.

Illustrative timeline for a directly federated role, assuming the assertion is not reused and there is no refresh or downstream assumption:

```text
09:00  IdP issues assertion expiring at 09:05.
09:01  STS accepts it and issues role credentials expiring at 10:01.
09:02  IdP account disabled; new sign-ins blocked.
09:05  Assertion expires. Existing AWS session can still call allowed APIs.
10:01  AWS session expires; continuing work requires a new successful exchange.
```

For urgent containment, block new issuance and deny existing role sessions using the appropriate provider mechanism. For ordinary IAM roles, AWS can attach a deny based on `aws:TokenIssueTime` to revoke older sessions; this affects other users of that role too. IAM Identity Center permission-set sessions have their own revocation procedure. Account for policy propagation and confirm denial with a controlled old session. Do not promise a fixed revocation bound without measuring every relevant path.

If the IdP, its verification endpoint, or STS is unavailable, new exchanges can fail while unexpired AWS credentials continue working where the destination service remains available. Clients should refresh before expiry, use bounded retries, and expose refresh failures. They must not keep using expired credentials or fall back to a broad static key. Recovery for operators belongs in a separately tested [emergency access path](multi-account-security.md#emergency-access-and-containment).

!!! info "External Resources"
    - [OIDC federation in IAM](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_roles_providers_oidc.html) (AWS)
    - [Revoking IAM role sessions and Identity Center caveat](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_roles_use_revoke-sessions.html) (AWS)
    - [IAM changes and eventual consistency](https://docs.aws.amazon.com/IAM/latest/UserGuide/troubleshoot.html#troubleshoot_general_eventual-consistency) (AWS)

### Role Chaining Does Not Preserve the Original Expiry

**Role chaining** means using one role's temporary credentials to assume another role. Each hop requires permission to assume the next role and that role's trust; “A trusts B” is not automatically transitive to C. API/CLI sessions obtained through role chaining have a maximum duration of one hour, even when the target role permits a longer session.

The new session has its own issue and expiry time. An attacker using A's session shortly before it expires can obtain a B session that lasts beyond A's expiry if the hop is permitted. Revoking A does not recursively revoke B sessions already issued. Track reachable downstream roles and deny their sessions or permissions during containment. Preserve trustworthy actor attribution across hops; a caller-chosen session name alone is not proof of a human identity.

```text
# Synthetic timeline: both hops and the requested one-hour session are allowed.
09:00  A session issued; expires 10:00.
09:59  A session assumes B; B session expires 10:59.
10:00  A expires. B's already-issued session does not expire with it.
# A one-hour initial session therefore did not cap the whole access path at 10:00.
```

!!! info "External Resources"
    - [AssumeRole duration and chaining constraints](https://docs.aws.amazon.com/STS/latest/APIReference/API_AssumeRole.html) (AWS)
    - [Disabling permissions for temporary credentials](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_credentials_temp_control-access_disable-perms.html) (AWS)

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

- **API gateway** - authenticates at the edge; downstream services accept identity context only over an authenticated, authorized gateway path that rejects spoofed client headers
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
| **Role chaining** | AWS | Use an allowed A-to-B assumption path; each hop needs its own grant and trust |
| **Service account impersonation** | GCP | `iam.serviceAccounts.getAccessToken` permission enables impersonation |
| **Instance metadata** | AWS | SSRF to `169.254.169.254` retrieves IAM role credentials |
| **Lambda/Function creation** | AWS | Create function with privileged role, invoke it |
| **Custom role modification** | GCP | Add permissions to a custom role already bound to the attacker; a new binding additionally requires the relevant policy-binding authority |
| **Pass-role abuse** | AWS/GCP | Pass a high-privilege role to a new resource you control |
| **Storage bucket policy** | AWS/GCP | Modify bucket policy to grant self access to sensitive data |

**Pass-role attack path:** An attacker may be unable to assume a privileged execution role directly but able to pass it to a service and control the code that service runs. For example, the combination of function creation, `iam:PassRole` on a privileged execution role, and invocation can expose that role's authority. Limit pass-role permissions to exact role ARNs and the intended service using `iam:PassedToService`, and restrict creation or modification of privileged workloads. `PassRole` is a permission check, not a standalone API event; inspect the service operation that received the role.

**Metadata credential theft:** Review [IMDSv2's exchange and limits](#ec2-metadata-credentials-and-imdsv2). Requiring a token reduces some SSRF paths; it does not stop an already compromised authorized workload from using its role.

**Detection and prevention:**

- Monitor for IAM policy changes (CloudTrail `PutRolePolicy`, `AttachUserPolicy`)
- Alert on new role assumptions from unexpected sources
- Use permission boundaries and SCPs to cap maximum permissions
- Audit `iam:PassRole`, `iam.serviceAccounts.actAs` grants
- Tools: Pacu (AWS exploitation), GCPBucketBrute, ScoutSuite, Prowler

!!! info "External Resources"
    - [PassRole permissions and service-operation auditing](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_roles_use_passrole.html) (AWS)
    - [Rhino Security Labs - AWS Privilege Escalation](https://rhinosecuritylabs.com/aws/aws-privilege-escalation-methods-mitigation/) (Rhino Security Labs)
    - [GCP Privilege Escalation - GitLab](https://about.gitlab.com/blog/2020/02/12/plundering-gcp-escalating-privileges-in-google-cloud-platform/) (GitLab)
    - [MITRE ATT&CK - Cloud Privilege Escalation](https://attack.mitre.org/tactics/TA0004/) (MITRE)
