# Zero Trust

## Core Principles

Zero trust is a security model based on the principle "never trust, always verify." It assumes that threats can exist both inside and outside the network, eliminating implicit trust based on network location.

**Zero trust vs perimeter-based security:**

| Property | Perimeter-based | Zero Trust |
|----------|----------------|------------|
| **Trust model** | Trust everything inside the network perimeter | Trust nothing; verify every request |
| **Network assumption** | Internal network is safe | No safe network; assume breach |
| **Access control** | VPN grants broad network access | Per-request authorisation based on identity, device, context |
| **Lateral movement** | Easy once inside the perimeter | Restricted by microsegmentation and continuous verification |
| **Visibility** | Focus on north-south traffic | Inspect both north-south and east-west traffic |
| **Authentication** | Often one-time at VPN login | Continuous, per-resource authentication |

**NIST SP 800-207 Zero Trust tenets:**

1. All data sources and computing services are considered resources
2. All communication is secured regardless of network location
3. Access to individual resources is granted on a per-session basis
4. Access is determined by dynamic policy (identity, device state, behaviour, environment)
5. The enterprise monitors and measures the integrity and security posture of all assets
6. All resource authentication and authorisation are dynamic and strictly enforced
7. The enterprise collects information about assets, network infrastructure, and communications to improve security posture

---

## BeyondCorp Model

Google's BeyondCorp is a practical implementation of zero trust that shifts access controls from the network perimeter to individual devices and users.

**BeyondCorp architecture components:**

| Component | Function |
|-----------|----------|
| **Device inventory service** | Tracks all known devices, their state, and ownership |
| **Device identity** | Certificates issued to managed devices; device is identified in every request |
| **User identity** | Strong authentication (SSO + MFA); user identity bound to every request |
| **Access proxy** | All application access flows through the proxy; enforces access policy per request |
| **Access control engine** | Evaluates policy based on user identity, device trust level, and resource sensitivity |
| **Trust inference** | Dynamically computes device trust level from inventory data, patch state, and security posture |
| **Pipeline** | Continuous analysis of device state, user behaviour, and policy compliance |

**Key BeyondCorp principles:**

- Access to services must not be determined by the network you connect from
- Access is granted based on what we know about the user and their device
- All access must be authenticated, authorised, and encrypted
- Access policy can (and should) differ per resource based on sensitivity

**Access decision flow:**

1. User authenticates via SSO with MFA
2. Device presents certificate proving managed status
3. Access proxy receives the request
4. Access control engine evaluates: user identity + group membership + device trust level + resource policy
5. If policy is satisfied, request is forwarded to the application
6. If not, access is denied with reason; user may need to remediate device state

!!! info "External Resources"
    - [BeyondCorp: A New Approach to Enterprise Security](https://research.google/pubs/pub43231/) (Google Research)
    - [BeyondCorp Papers](https://cloud.google.com/beyondcorp) (Google Cloud)

---

## Identity-Centric Security

In zero trust, identity is the new perimeter. Every access decision is anchored to a verified identity (user or service) rather than a network location.

**Identity pillars:**

| Pillar | Description | Implementation |
|--------|-------------|---------------|
| **Strong authentication** | Phishing-resistant MFA for all users | FIDO2/WebAuthn, certificate-based auth, push notifications with number matching |
| **Least-privilege access** | Grant minimum permissions needed | RBAC/ABAC, just-in-time (JIT) access, time-bounded elevated permissions |
| **Continuous verification** | Re-evaluate trust throughout the session | Session risk scoring, step-up authentication for sensitive actions |
| **Service identity** | Non-human identities authenticated and authorised | mTLS between services, workload identity (SPIFFE/SPIRE), service accounts with short-lived tokens |
| **Identity governance** | Lifecycle management and access reviews | Automated provisioning/deprovisioning, periodic access certifications |

**SPIFFE / SPIRE:**

| Component | Purpose |
|-----------|---------|
| **SPIFFE (Secure Production Identity Framework for Everyone)** | Standard for service identity; defines SPIFFE ID (URI format) and SVID (verifiable identity document) |
| **SPIRE (SPIFFE Runtime Environment)** | Reference implementation; attests workload identity and issues SVIDs (x509 or JWT) |
| **Use case** | Service-to-service authentication without shared secrets; enables mTLS between microservices |

---

## Microsegmentation

Microsegmentation divides the network into small, isolated segments with granular access controls, limiting lateral movement even after a breach.

**Microsegmentation approaches:**

| Approach | How it works | Granularity | Examples |
|----------|-------------|-------------|---------|
| **Network-based** | VLANs, subnets, firewall rules between segments | Subnet or VLAN level | Traditional firewalls, cloud VPC security groups |
| **Host-based** | Agent on each host enforces policy | Per-host or per-process | iptables, Windows Firewall, Calico |
| **Identity-based** | Policy based on workload identity rather than IP | Per-workload / per-service | Istio service mesh, Cilium with identity-aware policies |
| **Application-layer** | Enforce at L7 based on application context | Per-API or per-request | Service mesh mTLS + authorisation policies |

**Implementation strategy:**

1. **Discover** - map all communication flows between workloads (flow logs, network monitoring)
2. **Classify** - group workloads by function, sensitivity, and compliance requirements
3. **Define policy** - create allow-list rules for required communication paths
4. **Test in audit mode** - run policies in monitor/log-only mode to validate
5. **Enforce** - switch to enforcement mode, blocking unauthorized traffic
6. **Monitor and adapt** - continuously review logs, update policies as applications evolve

**Microsegmentation in Kubernetes:**

| Tool | Mechanism |
|------|-----------|
| **NetworkPolicy** | Native K8s resource; L3/L4 rules based on pod labels, namespaces, CIDRs |
| **Calico** | NetworkPolicy + GlobalNetworkPolicy; supports L7 policies, DNS-based rules |
| **Cilium** | eBPF-based; identity-aware L3/L4/L7 policies; transparent encryption |
| **Istio** | Service mesh; mTLS between pods + AuthorizationPolicy for L7 rules |

---

## Continuous Verification

Zero trust requires ongoing assessment rather than a single authentication check at the start of a session.

**Verification signals:**

| Signal | Source | Risk indicators |
|--------|--------|----------------|
| **Device posture** | MDM/EDR agent | Outdated OS, missing patches, disabled firewall, unsigned binaries |
| **User behaviour** | UBA/UEBA platform | Unusual login times, impossible travel, abnormal data access patterns |
| **Authentication context** | Identity provider | New device, unfamiliar location, failed MFA attempts |
| **Network context** | NAC, proxy, firewall | Connection from untrusted network, VPN with known-bad IP reputation |
| **Application context** | Application logs | Unusual API patterns, privilege escalation attempts, bulk data access |

**Adaptive access control responses:**

| Risk level | Response |
|------------|----------|
| **Low** | Allow access normally |
| **Medium** | Require step-up authentication (additional MFA factor) |
| **High** | Allow read-only access, block write operations |
| **Critical** | Block access entirely, alert security team |

---

## Implementation Roadmap

Adopting zero trust is a journey, not a single project. A phased approach reduces risk and builds organisational capability.

**Phase 1: Foundation (months 1-6)**

- Deploy strong identity provider with MFA for all users
- Inventory all assets, applications, and data flows
- Classify data by sensitivity level
- Implement centralised logging and monitoring
- Establish device management and posture assessment

**Phase 2: Quick wins (months 3-9)**

- Eliminate VPN for SaaS applications (use identity-aware proxy or ZTNA)
- Implement conditional access policies based on user + device + location
- Deploy microsegmentation for most sensitive workloads
- Enable service-to-service authentication (mTLS) for critical paths
- Automate access reviews and deprovisioning

**Phase 3: Maturity (months 6-18)**

- Extend microsegmentation to all workloads
- Implement continuous trust scoring with adaptive access control
- Deploy UEBA for anomaly detection
- Automate incident response for common zero trust policy violations
- Achieve identity-based segmentation replacing network-based controls

**Phase 4: Optimisation (ongoing)**

- Integrate threat intelligence into access decisions
- Implement just-in-time and just-enough access for all privileged operations
- Measure and report on zero trust maturity metrics
- Continuously refine policies based on behavioural analytics

!!! info "External Resources"
    - [NIST SP 800-207 - Zero Trust Architecture](https://csrc.nist.gov/publications/detail/sp/800-207/final) (NIST)
    - [CISA Zero Trust Maturity Model](https://www.cisa.gov/zero-trust-maturity-model) (CISA)
    - [Google BeyondCorp](https://cloud.google.com/beyondcorp) (Google)
    - [SPIFFE/SPIRE](https://spiffe.io/) (CNCF)
