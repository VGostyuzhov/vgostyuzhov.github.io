# Infrastructure & Cloud Security

This section covers virtualization, cloud security architecture, container security, identity and access management, and secrets management - the core topics for infrastructure and cloud security engineering roles.

## Virtualization & Cloud Foundations

- **[Virtualization & Cloud](virtualization.md)** - Hypervisors (Type 1 vs Type 2), VM security, cloud service models (IaaS, PaaS, SaaS), shared responsibility model, cloud vs on-prem differences

## Cloud Security Architecture

- **[Cloud Security](cloud-security.md)** - Cloud networking and segmentation, object storage security, cloud logging and monitoring, serverless and PaaS security, multi-cloud considerations
- **[Cloud IAM](iam.md)** - IAM fundamentals, common misconfigurations, service accounts and workload identity, role design, cross-account federation, privilege escalation in cloud
- **[Secrets Management](secrets-management.md)** - Secrets management tools (Vault, AWS Secrets Manager, GCP Secret Manager), rotation lifecycle, secrets in CI/CD, emergency revocation

## Container Security

- **[Container Security](containers.md)** - Container architecture and isolation, image security, runtime security, Kubernetes fundamentals, network policies, container escape techniques, hardening

## Key Learning Objectives

After studying this section, you should understand:

- Hypervisor types and virtual machine isolation boundaries
- Cloud service models and the shared responsibility model
- Cloud network segmentation and VPC design principles
- Object storage security risks and mitigations
- Container isolation mechanisms and their limitations
- Kubernetes security fundamentals and admission control
- Cloud IAM design patterns and common misconfigurations
- Service account security and workload identity federation
- Secrets lifecycle management and rotation strategies
- Privilege escalation paths in cloud environments

!!! info "Prerequisites"
    Familiarity with basic networking and Linux administration is assumed. For cloud fundamentals, refer to [AWS Well-Architected Framework](https://docs.aws.amazon.com/wellarchitected/latest/framework/welcome.html), [Google Cloud Architecture Framework](https://cloud.google.com/architecture/framework), and [Azure Well-Architected Framework](https://learn.microsoft.com/en-us/azure/well-architected/).
