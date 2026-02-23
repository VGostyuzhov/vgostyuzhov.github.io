# Container Security

## Container Architecture & Isolation

Containers share the host kernel and use Linux kernel features for isolation. This is fundamentally different from VMs, which have their own kernel.

**Isolation mechanisms:**

| Mechanism | What it isolates | Notes |
|-----------|-----------------|-------|
| **Namespaces** | PID, network, mount, UTS, IPC, user, cgroup | Each container sees its own process tree, network stack, filesystem |
| **cgroups** | CPU, memory, I/O, PIDs | Limits resource consumption; prevents noisy-neighbour DoS |
| **seccomp** | System calls | Filters which syscalls the container can invoke; default Docker profile blocks ~44 syscalls |
| **AppArmor/SELinux** | File access, capabilities | Mandatory access control confines container processes |
| **Capabilities** | Privilege subsets | Containers drop most capabilities by default; avoid `--privileged` |

**Containers vs VMs:**

| Property | Containers | VMs |
|----------|-----------|-----|
| Isolation boundary | Kernel features (namespaces, cgroups) | Hypervisor (hardware-enforced) |
| Kernel | Shared with host | Separate per VM |
| Startup time | Milliseconds to seconds | Seconds to minutes |
| Resource overhead | Minimal | Full OS per VM |
| Escape risk | Kernel vulnerabilities affect all containers | Hypervisor escape required |

!!! info "External Resources"
    - [Docker Security Documentation](https://docs.docker.com/engine/security/) (Docker)
    - [Linux Namespaces - man7.org](https://man7.org/linux/man-pages/man7/namespaces.7.html) (Linux man-pages)
    - [Understanding Container Isolation - NCC Group](https://research.nccgroup.com/2020/07/10/understanding-and-hardening-linux-containers/) (NCC Group)

## Image Security & Supply Chain

Container images are the deployment unit. A compromised image compromises every instance that runs it.

**Image security checklist:**

- Use minimal base images (`distroless`, `alpine`, `scratch`) to reduce attack surface
- Pin base images by digest, not tag (tags are mutable)
- Scan images for CVEs at build time and in registries
- Never include secrets, credentials, or SSH keys in image layers
- Use multi-stage builds to exclude build tools from final image
- Rebuild images on a schedule to pick up base-image security patches

**Supply chain hardening:**

| Practice | Tool/Standard |
|----------|--------------|
| **Image signing** | Cosign (Sigstore), Docker Content Trust (Notary) |
| **Provenance attestation** | SLSA framework, in-toto |
| **SBOM generation** | Syft, Trivy, CycloneDX |
| **Admission control** | Verify signatures before pod scheduling (Kyverno, OPA Gatekeeper) |
| **Private registry** | Harbor, AWS ECR, GCP Artifact Registry, Azure ACR |

**Scanning tools:**

| Tool | Scan Type |
|------|-----------|
| **Trivy** | CVE, misconfiguration, secrets, SBOM |
| **Grype** | CVE scanning against SBOM |
| **Snyk Container** | CVE + base image recommendations |
| **Clair** | CVE scanning for registries |

For detailed scanning pipeline coverage, see [DevSecOps - Container & Image Scanning](../emerging-tech/devsecops.md#container-image-scanning-in-cicd).

!!! info "External Resources"
    - [Sigstore / Cosign Documentation](https://docs.sigstore.dev/cosign/overview/) (Sigstore)
    - [SLSA Framework](https://slsa.dev/) (OpenSSF)
    - [Trivy Documentation](https://trivy.dev/) (Aqua Security)

## Container Runtime Security

Runtime security monitors container behaviour after deployment, detecting anomalies and preventing exploitation.

**Runtime threats:**

- Unexpected process execution (cryptominers, reverse shells)
- File system modifications in read-only containers
- Network connections to unexpected destinations
- Privilege escalation via kernel exploits or capability abuse
- Container escape to host

**Runtime security tools:**

| Tool | Approach | Notes |
|------|----------|-------|
| **Falco** | eBPF/kernel module syscall monitoring | Rule-based detection of runtime anomalies |
| **Tetragon** | eBPF-based observability and enforcement | Cilium project; can block syscalls in real time |
| **Sysdig** | Commercial platform built on Falco | Enterprise runtime security and forensics |
| **gVisor** | User-space kernel (application kernel) | Intercepts syscalls; stronger isolation, performance cost |
| **Kata Containers** | Lightweight VMs for containers | Hardware-enforced isolation per container |

**Runtime hardening:**

- Run containers as non-root (`USER` directive in Dockerfile, `runAsNonRoot` in K8s)
- Use read-only root filesystem (`readOnlyRootFilesystem: true`)
- Drop all capabilities, add only what is needed (`drop: ["ALL"]`, `add: ["NET_BIND_SERVICE"]`)
- Never run with `--privileged` flag
- Mount only necessary volumes; avoid mounting Docker socket
- Set resource limits to prevent DoS (CPU, memory, PID limits)

!!! info "External Resources"
    - [Falco Documentation](https://falco.org/docs/) (Falco)
    - [gVisor Documentation](https://gvisor.dev/docs/) (Google)
    - [NIST SP 800-190 - Container Security Guide](https://csrc.nist.gov/publications/detail/sp/800-190/final) (NIST)

## Kubernetes Security Fundamentals

Kubernetes (K8s) orchestrates containers at scale. Its complexity introduces a large security surface.

**K8s attack surface:**

| Component | Risk | Mitigation |
|-----------|------|-----------|
| **API Server** | Unauthenticated access, RBAC bypass | Enable RBAC, disable anonymous auth, audit logging |
| **etcd** | Stores all cluster state including secrets | Encrypt at rest, restrict network access, TLS client certs |
| **Kubelet** | Node-level container management | Disable anonymous auth, use NodeRestriction admission |
| **Pod** | Compromised workload | Pod Security Standards, seccomp, AppArmor |
| **Service Account** | Default SA auto-mounted in every pod | Disable auto-mount (`automountServiceAccountToken: false`) |
| **Container registry** | Malicious images | Image pull policies (`Always`), admission control |

**Pod Security Standards (PSS):**

| Level | Description |
|-------|------------|
| **Privileged** | No restrictions (admin workloads only) |
| **Baseline** | Blocks known privilege escalation (no hostNetwork, no privileged containers) |
| **Restricted** | Hardened (non-root, read-only root FS, drop all capabilities) |

**RBAC essentials:**

- Use `Role`/`RoleBinding` (namespace-scoped) over `ClusterRole`/`ClusterRoleBinding` where possible
- Avoid `cluster-admin` for workloads; create specific roles
- Audit RBAC regularly; use tools like `rbac-tool` or `kubectl auth can-i --list`

!!! info "External Resources"
    - [Kubernetes Security Documentation](https://kubernetes.io/docs/concepts/security/) (Kubernetes)
    - [NSA/CISA Kubernetes Hardening Guide](https://media.defense.gov/2022/Aug/29/2003066362/-1/-1/0/CTR_KUBERNETES_HARDENING_GUIDANCE_1.2_20220829.PDF) (NSA/CISA)
    - [CIS Kubernetes Benchmark](https://www.cisecurity.org/benchmark/kubernetes) (CIS)

## Network Policies & Service Mesh

**Kubernetes Network Policies:**

- By default, all pods can communicate with all other pods (flat network)
- NetworkPolicy resources define allow-list rules for ingress and egress
- Require a CNI plugin that supports NetworkPolicy (Calico, Cilium, Weave)

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-all-ingress
spec:
  podSelector: {}
  policyTypes:
    - Ingress
```

**Default deny strategy:** Apply a deny-all policy to each namespace, then add specific allow rules. This prevents unexpected lateral movement.

**Service mesh (Istio, Linkerd, Cilium):**

| Feature | Security Benefit |
|---------|-----------------|
| **mTLS** | Automatic mutual TLS between all services; identity-based auth |
| **Authorization policies** | Fine-grained L7 access control (method, path, headers) |
| **Traffic observability** | Distributed tracing, access logs per request |
| **Rate limiting** | Per-service and per-endpoint throttling |
| **Circuit breaking** | Prevents cascading failures |

**Service mesh security considerations:**

- mTLS eliminates network-level eavesdropping between pods
- Authorization policies enforce zero trust within the cluster
- Sidecar proxy adds latency and resource overhead
- Control plane (istiod) is a high-value target; secure with RBAC and network isolation

!!! info "External Resources"
    - [Kubernetes Network Policies](https://kubernetes.io/docs/concepts/services-networking/network-policies/) (Kubernetes)
    - [Cilium Documentation](https://docs.cilium.io/) (Cilium)
    - [Istio Security](https://istio.io/latest/docs/concepts/security/) (Istio)

## Container Escape Techniques

Container escape is when an attacker breaks out of the container's isolation and gains access to the host system.

**Common escape vectors:**

| Vector | Mechanism | Prerequisites |
|--------|-----------|--------------|
| **Privileged container** | Full host capabilities, device access | `--privileged` flag or all capabilities granted |
| **Docker socket mount** | Create new privileged containers from within | `/var/run/docker.sock` mounted |
| **Kernel exploit** | Exploit kernel vulnerability from within container | Shared kernel, outdated host |
| **Writable hostPath** | Write to host filesystem (cron, SSH keys, systemd) | `hostPath` volume with write access |
| **CAP_SYS_ADMIN** | Mount host filesystems, use `nsenter` | Single dangerous capability |
| **CVE-2019-5736 (runc)** | Overwrite host runc binary via `/proc/self/exe` | Vulnerable runc version |
| **CVE-2022-0185** | Heap overflow in filesystem context | Unprivileged user namespace |

**Detection indicators:**

- Processes accessing `/proc/1/root` or host-mounted paths
- New mount operations from within containers
- Unexpected capability usage detected by Falco/Tetragon
- Network connections from container to host management ports

**Prevention:**

- Never use `--privileged` in production
- Do not mount the Docker/containerd socket into containers
- Drop all capabilities; add only required ones
- Use read-only root filesystem
- Keep host kernel and container runtime patched
- Use gVisor or Kata Containers for high-risk workloads
- Enable seccomp and AppArmor/SELinux profiles

!!! info "External Resources"
    - [Container Escape - HackTricks](https://book.hacktricks.xyz/linux-hardening/privilege-escalation/docker-security/docker-breakout-privilege-escalation) (HackTricks)
    - [Trail of Bits - Container Security](https://blog.trailofbits.com/2019/07/19/understanding-docker-container-escapes/) (Trail of Bits)
    - [MITRE ATT&CK - Escape to Host](https://attack.mitre.org/techniques/T1611/) (MITRE)

## Container Hardening Best Practices

**Build-time hardening:**

- Use minimal base images; prefer `distroless` for production
- Run as non-root user; create a dedicated user in Dockerfile
- Remove package managers and shells from production images
- Use `.dockerignore` to exclude secrets, docs, and test files
- Scan images in CI; fail builds on critical/high CVEs

**Runtime hardening checklist:**

| Control | Setting |
|---------|---------|
| Non-root | `runAsNonRoot: true`, `runAsUser: 1000` |
| Read-only FS | `readOnlyRootFilesystem: true` |
| No privilege escalation | `allowPrivilegeEscalation: false` |
| Drop capabilities | `drop: ["ALL"]` |
| Seccomp | `RuntimeDefault` or custom profile |
| Resource limits | CPU and memory limits set |
| Service account | `automountServiceAccountToken: false` |

**Orchestration-level hardening:**

- Enforce Pod Security Standards (Restricted level) via admission controllers
- Use NetworkPolicies with default-deny
- Rotate and scope service accounts per workload
- Enable audit logging for the Kubernetes API server
- Scan running workloads with runtime security tools (Falco, Tetragon)
- Regularly audit RBAC permissions and remove unused roles

!!! info "External Resources"
    - [Docker Bench for Security](https://github.com/docker/docker-bench-security) (Docker)
    - [CIS Docker Benchmark](https://www.cisecurity.org/benchmark/docker) (CIS)
    - [Kubernetes Pod Security Standards](https://kubernetes.io/docs/concepts/security/pod-security-standards/) (Kubernetes)
