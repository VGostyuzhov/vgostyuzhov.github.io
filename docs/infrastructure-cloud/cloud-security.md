# Cloud Security

## Cloud Networking & Segmentation

**Virtual Private Cloud (VPC) design:**

- VPCs provide isolated network segments within cloud environments
- Subnets (public and private) control routing and internet exposure
- Public subnets have routes to an Internet Gateway; private subnets use NAT Gateway for outbound-only access

**Network controls:**

| Control | Scope | Statefulness | Use Case |
|---------|-------|-------------|----------|
| **Security Groups** | Instance/ENI level | Stateful | Allow-list inbound/outbound per resource |
| **NACLs** | Subnet level | Stateless | Broad subnet-level deny rules |
| **VPC Flow Logs** | VPC/Subnet/ENI | N/A (logging) | Network forensics, anomaly detection |
| **PrivateLink / Private Service Connect** | Service level | N/A | Access cloud services without public internet |
| **Transit Gateway / VPC Peering** | Inter-VPC | N/A | Connect VPCs without internet routing |

**Segmentation strategy:**

- Separate environments (dev, staging, prod) into distinct VPCs or accounts
- Use private subnets for databases and internal services
- Minimise public-facing resources; front with load balancers
- Enable VPC Flow Logs for all subnets; forward to SIEM
- Implement microsegmentation with security groups per workload

**Zero Trust networking in cloud:**

- Do not rely on VPC boundaries as security perimeters
- Authenticate and authorise every request (service mesh with mTLS)
- Use identity-aware proxies (BeyondCorp Enterprise, AWS Verified Access)
- See [Zero Trust Architecture](../emerging-tech/zero-trust.md) for detailed coverage

!!! info "External Resources"
    - [AWS VPC Security Best Practices](https://docs.aws.amazon.com/vpc/latest/userguide/vpc-security-best-practices.html) (AWS)
    - [Google Cloud VPC Documentation](https://cloud.google.com/vpc/docs/overview) (Google Cloud)
    - [Azure Network Security Best Practices](https://learn.microsoft.com/en-us/azure/security/fundamentals/network-best-practices) (Microsoft)

## Object Storage Security (S3, GCS, Blob)

Object storage is the most common source of cloud data breaches, typically due to misconfigured access controls.

**Common misconfigurations:**

- Public buckets/blobs (intentional or accidental)
- Overly permissive bucket policies or ACLs
- Missing server-side encryption
- Cross-account access without proper condition constraints
- Logging disabled (no audit trail of access)

**Security controls:**

| Control | AWS S3 | GCP GCS | Azure Blob |
|---------|--------|---------|-----------|
| **Block public access** | S3 Block Public Access (account + bucket level) | Uniform bucket-level access | Storage account public access setting |
| **Encryption at rest** | SSE-S3, SSE-KMS, SSE-C | Google-managed, CMEK, CSEK | Microsoft-managed, CMK |
| **Access logging** | S3 Server Access Logging, CloudTrail data events | Cloud Audit Logs | Storage Analytics, Azure Monitor |
| **Versioning** | S3 Versioning | Object Versioning | Blob Versioning |
| **Lifecycle policies** | Transition/expiration rules | Lifecycle management | Lifecycle management |
| **Object lock** | S3 Object Lock (WORM) | Retention policies | Immutable storage |

**Best practices:**

- Enable "Block Public Access" at the account level by default
- Use bucket policies (not ACLs) for access control; ACLs are legacy
- Encrypt with customer-managed keys (CMEK/SSE-KMS) for data you control
- Enable access logging and forward to SIEM
- Set up alerts for public access changes, large downloads, or unusual API calls
- Use pre-signed URLs for temporary access rather than broad permissions
- Enable versioning and object lock for critical data (ransomware protection)

!!! info "External Resources"
    - [AWS S3 Security Best Practices](https://docs.aws.amazon.com/AmazonS3/latest/userguide/security-best-practices.html) (AWS)
    - [GCP Cloud Storage Security](https://cloud.google.com/storage/docs/best-practices#security) (Google Cloud)
    - [Azure Storage Security Guide](https://learn.microsoft.com/en-us/azure/storage/common/storage-security-guide) (Microsoft)

## Cloud Logging & Monitoring

Logging is the foundation of cloud security operations. You cannot defend what you cannot see.

**Log types:**

| Log Category | Examples | Security Value |
|-------------|---------|---------------|
| **Control plane** | AWS CloudTrail, GCP Admin Activity, Azure Activity Log | Who did what to infrastructure (IAM changes, resource creation, policy modifications) |
| **Data plane** | S3 access logs, GCS data access, VPC Flow Logs | Who accessed what data, network connections |
| **Application** | Custom app logs, Lambda/Cloud Function logs | Business logic anomalies, authentication events |
| **Audit** | Config changes, compliance events | Regulatory evidence, change tracking |

**Cloud-native logging services:**

| Provider | Logging Service | SIEM/Analysis | Alert Service |
|----------|----------------|---------------|---------------|
| AWS | CloudTrail, CloudWatch Logs, VPC Flow Logs | Security Hub, GuardDuty, Athena | CloudWatch Alarms, EventBridge |
| GCP | Cloud Audit Logs, VPC Flow Logs | Security Command Center, Chronicle | Cloud Monitoring, Pub/Sub |
| Azure | Activity Log, Diagnostic Logs, NSG Flow Logs | Sentinel, Defender for Cloud | Azure Monitor Alerts |

**Best practices:**

- Enable control plane logging in all accounts/projects (CloudTrail, Admin Activity logs)
- Centralise logs into a dedicated security account/project with write-once storage
- Set retention policies based on compliance requirements (typically 1-7 years)
- Alert on high-value events: root/admin login, IAM policy changes, security group modifications, public resource creation
- Use log analytics for baseline behaviour; detect anomalies against that baseline
- Protect log integrity: immutable storage, cross-account replication, tamper detection

**Cost management:**

- Data plane logs generate high volume; sample or filter where acceptable
- Use tiered storage (hot/warm/cold) based on query frequency
- Set up log budgets and alerts for unexpected volume spikes

!!! info "External Resources"
    - [AWS Security Logging Guide](https://docs.aws.amazon.com/prescriptive-guidance/latest/logging-monitoring-for-application-owners/aws-services-for-logging-and-monitoring.html) (AWS)
    - [GCP Cloud Logging Best Practices](https://cloud.google.com/logging/docs/best-practices) (Google Cloud)
    - [Azure Monitoring Best Practices](https://learn.microsoft.com/en-us/azure/cloud-adoption-framework/manage/monitor/best-practices) (Microsoft)

## Cloud Threat Modeling

Cloud threat modeling requires an attacker mindset focused on identity abuse, misconfiguration exploitation, and lateral movement.

**Cloud-specific threat categories:**

| Category | Examples |
|----------|---------|
| **Identity abuse** | Stolen API keys, compromised service accounts, OAuth token theft |
| **Misconfiguration** | Public S3 buckets, overly permissive IAM, disabled logging |
| **Lateral movement** | Service account impersonation, cross-project access, metadata service exploitation |
| **Persistence** | Backdoor IAM users, Lambda functions, modified AMIs/images |
| **Data exfiltration** | Snapshot sharing, bucket replication to external accounts, DNS exfil |

**Cloud threat modeling approach:**

1. Map the architecture - accounts, VPCs, services, data flows, trust boundaries
2. Identify assets - data classification, regulatory scope, business criticality
3. Enumerate threats - use STRIDE against cloud components; consider cloud-specific vectors
4. Assess risk - likelihood based on exposure and controls; impact based on data classification
5. Prioritise mitigations - focus on identity, network segmentation, detection

**Assume breach mindset:**

- What happens when an EC2 instance is compromised? Can it reach the metadata service? What IAM role is attached?
- What happens when a developer's credentials are leaked? What can they access? Is MFA enforced?
- What happens when a CI/CD pipeline is compromised? Can it deploy to production? What secrets does it have?

!!! info "External Resources"
    - [MITRE ATT&CK Cloud Matrix](https://attack.mitre.org/matrices/enterprise/cloud/) (MITRE)
    - [Threat Modeling Manifesto](https://www.threatmodelingmanifesto.org/) (Community)
    - [AWS Threat Model for Serverless](https://docs.aws.amazon.com/whitepapers/latest/security-overview-aws-lambda/threat-model-for-lambda.html) (AWS)

## Serverless & PaaS Security

Serverless and PaaS shift infrastructure management to the provider, changing the security model.

**Serverless security considerations:**

| Concern | Detail |
|---------|--------|
| **Function permissions** | Each function gets an IAM role; principle of least privilege per function, not per application |
| **Event injection** | Input from queues, API gateways, storage triggers - all must be validated |
| **Cold start trust** | Execution environment may be reused; do not cache secrets in global state |
| **Dependency risk** | Deployment packages include dependencies; scan for vulnerabilities |
| **Execution duration** | Short timeouts limit certain attack persistence; but event-driven chains can persist |
| **Logging** | Functions must explicitly log; ensure structured logging for SIEM ingestion |

**PaaS security considerations:**

- Provider manages OS and runtime; customer manages application code and data
- Platform vulnerabilities affect all tenants (Log4j in managed Java services)
- Restrict deployment to trusted sources (signed artifacts, approved registries)
- Understand data residency and multi-tenancy implications

**Common attack patterns:**

- Overprivileged function roles enabling lateral movement
- Deserialization of untrusted event payloads
- SSRF from functions with network access to internal services
- Environment variable secrets readable by compromised functions

!!! info "External Resources"
    - [OWASP Serverless Top 10](https://owasp.org/www-project-serverless-top-10/) (OWASP)
    - [AWS Lambda Security Best Practices](https://docs.aws.amazon.com/lambda/latest/dg/lambda-security.html) (AWS)
    - [Serverless Security - CSA](https://cloudsecurityalliance.org/research/topics/serverless/) (Cloud Security Alliance)

## Multi-Cloud & Hybrid Considerations

**Multi-cloud** - using services from multiple cloud providers (AWS + GCP + Azure).

**Hybrid cloud** - combining on-premises infrastructure with one or more cloud providers.

**Security challenges:**

| Challenge | Detail |
|-----------|--------|
| **Identity fragmentation** | Separate IAM systems per provider; need federated identity (OIDC, SAML) |
| **Inconsistent controls** | Security groups vs NSGs vs firewall rules - different semantics per provider |
| **Visibility gaps** | Logs in different formats, different retention, different query languages |
| **Secret sprawl** | Secrets in multiple vaults, different rotation mechanisms |
| **Compliance complexity** | Different certifications per provider; data residency across jurisdictions |
| **Skill requirements** | Team must understand security tooling across all providers |

**Mitigation strategies:**

- Use a centralised identity provider (Okta, Azure AD) federated to all cloud accounts
- Standardise on infrastructure-as-code (Terraform) with shared security modules
- Aggregate logs into a single SIEM (Splunk, Elastic, Chronicle)
- Adopt a cloud security posture management (CSPM) tool that supports all providers
- Define a common security baseline (CIS benchmarks exist for each provider)
- Abstract secrets management where possible (HashiCorp Vault across providers)

!!! info "External Resources"
    - [NIST SP 800-210 - General Access Control Guidance for Cloud](https://csrc.nist.gov/publications/detail/sp/800-210/final) (NIST)
    - [HashiCorp Multi-Cloud Security](https://www.hashicorp.com/solutions/multi-cloud-security) (HashiCorp)
    - [CSA Cloud Controls Matrix](https://cloudsecurityalliance.org/research/cloud-controls-matrix/) (Cloud Security Alliance)
