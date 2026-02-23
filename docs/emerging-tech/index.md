# Emerging Technologies & Practices

This section covers modern attack techniques, zero trust architecture, and DevSecOps practices. These topics represent the current direction of security engineering - how threats are evolving, how architecture is adapting, and how security integrates into development workflows.

## Core Topics

### Threat Landscape

- **[Modern Attacks](modern-attacks.md)** - Supply chain attacks (SolarWinds, Log4j, Codecov), cloud-native attacks, API abuse and business logic attacks, AI/ML security risks, side-channel attacks, ransomware evolution

### Architecture

- **[Zero Trust](zero-trust.md)** - Zero trust principles, BeyondCorp model, identity-centric security, microsegmentation, continuous verification, implementation roadmap

### Secure Development

- **[DevSecOps](devsecops.md)** - Secure SDLC and threat modeling, dependency management, code review and linters, infrastructure as code security, container and image scanning, policy as code (OPA, Sentinel)

## Key Learning Objectives

After studying this section, you should understand:

- How modern supply chain attacks compromise build and distribution pipelines
- Cloud-native attack patterns and their defences
- AI/ML-specific security risks and adversarial techniques
- Zero trust principles and how they differ from perimeter-based security
- BeyondCorp architecture and identity-centric access control
- Microsegmentation design and continuous verification
- Secure SDLC phases and how to embed security at each stage
- Dependency management, SBOM generation, and supply chain hardening
- Infrastructure as code scanning and policy enforcement
- Container security scanning across build, registry, and runtime stages

!!! info "Prerequisites"
    Familiarity with cloud infrastructure, CI/CD pipelines, and basic security concepts is assumed. For background, refer to [Google BeyondCorp Papers](https://cloud.google.com/beyondcorp) and [OWASP DevSecOps Guideline](https://owasp.org/www-project-devsecops-guideline/).
