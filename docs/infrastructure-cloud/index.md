# Infrastructure & Cloud Security

Study the mechanisms before combining them into an operating design. The AWS worked examples make policy and account boundaries concrete; provider behavior should be checked separately when applying the same requirements elsewhere.

## Learning Path

1. [Virtualization & Cloud Fundamentals](virtualization.md): hypervisors, isolation, service models, and shared responsibility.
2. [Cloud Security](cloud-security.md): network segmentation, storage protection, and cloud security controls.
3. [Container Security](containers.md): workload isolation, Kubernetes access, and runtime constraints.
4. [Cloud IAM](iam.md): policy evaluation, workload credentials, cross-account trust, federation, and revocation.
5. [Secrets Management](secrets-management.md): storage, distribution, rotation, and emergency revocation of application secrets.
6. [Multi-Account Security Architecture](multi-account-security.md): integrate IAM, central evidence, account onboarding, staged policy changes, and emergency access in a 200-account exercise.
7. [Staff / Principal Cloud Scenarios](../interview-prep/staff-principal.md): practice the IAM and architecture reasoning taught in this section.

For a focused IAM path, start with [grants and ceilings](iam.md#aws-policy-evaluation-grants-and-permission-ceilings), trace [cross-account assumption and resource access](iam.md#trace-caller-trust-and-target-action), then test the [architecture's operating claims](multi-account-security.md#validation-and-operating-measures).
