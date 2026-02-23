# Secrets Management

## Why Secrets Management Matters

Secrets (API keys, database credentials, TLS private keys, tokens, encryption keys) are the keys to infrastructure. Leaked secrets are the most common initial access vector in cloud breaches.

**Where secrets leak:**

- Source code repositories (committed `.env` files, hardcoded credentials)
- CI/CD pipeline logs and environment variables
- Container image layers (secrets baked into images)
- Configuration files on disk (plaintext in `/etc`, application configs)
- Chat messages and documentation (Slack, Confluence, wikis)
- Cloud metadata and instance user data

**Impact of secret exposure:**

- Direct access to databases, APIs, cloud accounts
- Lateral movement to connected services
- Data exfiltration without triggering normal authentication alerts
- Persistent access until secret is rotated (which may be never)

!!! info "External Resources"
    - [OWASP Secrets Management Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Secrets_Management_Cheat_Sheet.html) (OWASP)
    - [GitGuardian State of Secrets Sprawl](https://www.gitguardian.com/state-of-secrets-sprawl-report) (GitGuardian)
    - [NIST SP 800-57 - Key Management](https://csrc.nist.gov/publications/detail/sp/800-57-part-1/rev-5/final) (NIST)

## Secrets Management Tools (Vault, AWS SM, GCP SM)

| Tool | Type | Key Features |
|------|------|-------------|
| **HashiCorp Vault** | Self-hosted / HCP managed | Dynamic secrets, PKI, transit encryption, multi-cloud, rich ACL, audit logging |
| **AWS Secrets Manager** | Managed service | Automatic rotation for RDS/Redshift/DocumentDB, cross-account sharing, CloudFormation integration |
| **AWS Systems Manager Parameter Store** | Managed service | Free tier (standard), KMS encryption, hierarchical paths, less rotation automation |
| **GCP Secret Manager** | Managed service | Versioning, IAM-based access, automatic replication, audit logging |
| **Azure Key Vault** | Managed service | Secrets, keys, certificates in one service, HSM-backed, RBAC and access policies |
| **CyberArk Conjur** | Enterprise | Designed for DevOps, policy-based access, integrates with K8s and CI/CD |
| **Doppler** | SaaS | Developer-focused, syncs secrets to multiple environments, audit trail |

**HashiCorp Vault architecture:**

- **Secrets engines** - generate or store secrets (KV, database, PKI, AWS, transit)
- **Auth methods** - authenticate clients (AppRole, Kubernetes, OIDC, AWS IAM, TLS certs)
- **Policies** - HCL-based ACL rules controlling who can access what paths
- **Audit devices** - log every request and response for compliance
- **Seal/Unseal** - Vault starts sealed; requires key shares or auto-unseal via cloud KMS

!!! info "External Resources"
    - [HashiCorp Vault Documentation](https://developer.hashicorp.com/vault/docs) (HashiCorp)
    - [AWS Secrets Manager Documentation](https://docs.aws.amazon.com/secretsmanager/) (AWS)
    - [GCP Secret Manager Documentation](https://cloud.google.com/secret-manager/docs) (Google Cloud)

## Secret Rotation & Lifecycle

**Secret lifecycle phases:**

1. **Generation** - create with sufficient entropy; use cryptographic RNG
2. **Storage** - store in secrets manager, never in code or config files
3. **Distribution** - inject at runtime via environment, mounted volume, or API call
4. **Usage** - application retrieves and uses secret; minimize in-memory lifetime
5. **Rotation** - replace secret on schedule or on-demand; update all consumers
6. **Revocation** - invalidate immediately when compromised or no longer needed
7. **Audit** - log all access, rotation, and revocation events

**Rotation strategies:**

| Strategy | How it works | Pros | Cons |
|----------|-------------|------|------|
| **Automated rotation** | Secrets manager rotates on schedule (e.g., every 30 days) | No human intervention, consistent | Requires app to re-fetch; can break if misconfigured |
| **Dynamic secrets** | Vault generates unique, short-lived credentials per request | No shared secrets, automatic expiry | Requires Vault integration, more complex |
| **Dual-secret rotation** | Two versions active simultaneously during transition period | Zero-downtime rotation | Application must support fallback to previous version |

**Rotation best practices:**

- Automate rotation for all secrets; manual rotation does not scale
- Use short TTLs (hours, not months) where infrastructure supports it
- Test rotation in staging before enabling in production
- Monitor for rotation failures and alert immediately
- Keep previous version valid during transition window to prevent outages

!!! info "External Resources"
    - [AWS Secrets Manager Rotation](https://docs.aws.amazon.com/secretsmanager/latest/userguide/rotating-secrets.html) (AWS)
    - [Vault Dynamic Secrets](https://developer.hashicorp.com/vault/docs/secrets) (HashiCorp)
    - [NIST SP 800-57 - Key Lifecycle](https://csrc.nist.gov/publications/detail/sp/800-57-part-1/rev-5/final) (NIST)

## Secrets in CI/CD Pipelines

CI/CD pipelines are high-value targets because they typically have broad access to deploy infrastructure and applications.

**Common CI/CD secret risks:**

- Secrets stored as plaintext environment variables in pipeline configuration
- Secrets visible in build logs (echoed by debug statements or error messages)
- Secrets accessible to all branches (including feature branches from forks)
- Shared service accounts across pipelines with overprivileged access
- Secrets persisted in pipeline artifacts or caches

**Best practices:**

| Practice | Detail |
|----------|--------|
| **Use native secret stores** | GitHub Actions secrets, GitLab CI variables (masked), Jenkins credentials |
| **Limit secret scope** | Environment-specific secrets (prod secrets only in prod deployment jobs) |
| **Mask in logs** | Enable secret masking; never echo secrets in scripts |
| **Short-lived credentials** | Use OIDC federation (GitHub Actions OIDC to AWS/GCP) instead of long-lived keys |
| **No secrets in code** | Use `.gitignore` for `.env` files; run pre-commit secret scanners |
| **Audit access** | Log which pipeline jobs access which secrets |
| **Rotate on exposure** | If a pipeline log leaks a secret, rotate immediately |

**OIDC federation for CI/CD:**

- GitHub Actions, GitLab CI, and CircleCI can issue OIDC tokens
- Cloud providers exchange these tokens for short-lived credentials
- No long-lived secrets stored in CI/CD configuration
- Conditions can restrict access by repository, branch, or environment

```yaml
# Example: GitHub Actions OIDC to AWS
permissions:
  id-token: write
steps:
  - uses: aws-actions/configure-aws-credentials@v4
    with:
      role-to-assume: arn:aws:iam::123456789:role/github-deploy
      aws-region: us-east-1
```

!!! info "External Resources"
    - [GitHub Actions OIDC](https://docs.github.com/en/actions/deployment/security-hardening-your-deployments/about-security-hardening-with-openid-connect) (GitHub)
    - [GitLab CI/CD Secrets](https://docs.gitlab.com/ee/ci/secrets/) (GitLab)
    - [Securing CI/CD Pipelines - CISA](https://www.cisa.gov/sites/default/files/2023-06/defending_ci_cd_environments.pdf) (CISA)

## Application-Level Secret Injection

Secrets must reach applications securely at runtime without being persisted in code, images, or configuration files.

**Injection methods:**

| Method | How it works | Security | Use Case |
|--------|-------------|----------|----------|
| **Environment variables** | Set by orchestrator at container/process start | Visible in `/proc/*/environ`, leaked in crash dumps | Simple apps, 12-factor methodology |
| **Mounted files** | Secrets manager mounts secrets as files in a tmpfs volume | Not persisted to disk; file permissions enforced | Kubernetes (CSI driver, Vault Agent) |
| **API fetch** | Application calls secrets manager API at startup | Secret never on disk; requires SDK integration | Vault API, AWS SDK, GCP client libraries |
| **Sidecar injection** | Sidecar container fetches secrets and makes available to main container | Transparent to application; handles rotation | Vault Agent Injector, external-secrets operator |
| **Init container** | Runs before main container; populates shared volume | Simple setup; secret available at startup | Kubernetes init containers |

**Kubernetes secret injection options:**

- **K8s Secrets** - base64-encoded (not encrypted by default); enable encryption at rest in etcd
- **External Secrets Operator** - syncs secrets from Vault/AWS SM/GCP SM into K8s Secrets
- **Vault CSI Provider** - mounts Vault secrets as files directly into pods
- **Vault Agent Injector** - sidecar that renders Vault secrets to shared tmpfs volume

**Best practices:**

- Never bake secrets into container images (they persist in layers)
- Prefer file-based injection over environment variables (env vars leak more easily)
- Use short-lived dynamic secrets where possible
- Implement secret caching with TTL to reduce API calls and handle outages
- Log secret access for audit; alert on anomalous access patterns

!!! info "External Resources"
    - [Vault Agent Injector for Kubernetes](https://developer.hashicorp.com/vault/docs/platform/k8s/injector) (HashiCorp)
    - [External Secrets Operator](https://external-secrets.io/) (External Secrets)
    - [Kubernetes Secrets Best Practices](https://kubernetes.io/docs/concepts/security/secrets-good-practices/) (Kubernetes)

## Emergency Revocation & Incident Response

When a secret is compromised, speed of revocation determines blast radius.

**Incident response playbook for secret exposure:**

1. **Identify scope** - which secret, what does it access, where is it used
2. **Revoke immediately** - disable/delete the key, rotate the credential, revoke the token
3. **Assess damage** - check access logs for unauthorized usage during exposure window
4. **Rotate dependent secrets** - if the exposed secret could access other secrets, rotate those too
5. **Update consumers** - deploy new credentials to all legitimate consumers
6. **Root cause** - how was the secret exposed (committed to repo, logged, social engineering)
7. **Improve controls** - add pre-commit hooks, restrict access, shorten TTLs

**Preparation for fast revocation:**

- Maintain an inventory of all secrets and their consumers
- Automate revocation with runbooks (one-click rotation scripts)
- Test revocation procedures regularly (game days, tabletop exercises)
- Ensure applications handle credential refresh gracefully (retry with new credentials)
- Pre-configure alerts for secret usage from unexpected locations or at unusual times

**Secret scanning tools:**

| Tool | Scope |
|------|-------|
| **Gitleaks** | Git repository scanning (pre-commit and CI) |
| **TruffleHog** | Git history scanning, entropy-based detection |
| **GitGuardian** | SaaS platform for continuous repository monitoring |
| **AWS Macie** | S3 bucket scanning for sensitive data |
| **detect-secrets (Yelp)** | Pre-commit hook and CI scanner |

!!! info "External Resources"
    - [GitHub Secret Scanning](https://docs.github.com/en/code-security/secret-scanning/about-secret-scanning) (GitHub)
    - [Gitleaks Documentation](https://gitleaks.io/) (Gitleaks)
    - [SANS - Incident Response for Cloud Credential Compromise](https://www.sans.org/blog/cloud-incident-response/) (SANS)
