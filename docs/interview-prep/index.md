# Interview Preparation

Use this roadmap to move from explaining a mechanism to defending a design under changed constraints. The collections have different scopes; question count is not a measure of readiness.

## Choose a Practice Path

| Goal | Study and practice |
|---|---|
| Review broad security topics and prepare experience-based answers | [Security Questions](questions.md), the existing general collection |
| Reason about cloud IAM and organization-scale operation | [Cloud IAM](../infrastructure-cloud/iam.md), [Multi-Account Security Architecture](../infrastructure-cloud/multi-account-security.md), then [Staff / Principal Cloud Scenarios](staff-principal.md) |
| Diagnose network and protocol edge cases | [Networking interview questions](../networking/interview-questions.md) and their linked lessons |
| Explain Linux isolation and access-control behavior | [Linux interview questions](../linux/interview-questions.md) and their linked lessons |
| Choose and debug authentication designs | [Authentication design patterns](../authentication/design-patterns.md), [case studies](../authentication/case-studies.md), then [authentication questions](../authentication/interview-questions.md) |

## Practice with Evidence

1. State the scenario's assets, actors, constraints, and trust boundaries.
2. Trace one successful operation and one failure through the actual checks.
3. Explain what changes when credentials are stolen, dependencies fail, or policy changes propagate.
4. Propose a scoped improvement and a test that would expose the original mistake.
5. For architecture discussions, include ownership, rollout, recovery, and measurable operating results.

Use specific examples from your work where available and distinguish what you implemented from what you would propose. The staff/principal page currently covers the first cloud IAM and multi-account batch; it does not imply complete coverage of those career levels or verified questions from particular employers.
