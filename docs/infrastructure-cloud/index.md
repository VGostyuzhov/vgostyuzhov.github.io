# Virtualization & Cloud Fundamentals

## Hypervisors (Type 1 vs Type 2)

A hypervisor (Virtual Machine Monitor) creates and manages virtual machines, providing hardware abstraction and isolation between guest operating systems.

| Property | Type 1 (Bare-metal) | Type 2 (Hosted) |
|----------|---------------------|-----------------|
| **Runs on** | Directly on hardware | On top of a host OS |
| **Performance** | Near-native | Overhead from host OS layer |
| **Use case** | Production servers, cloud infrastructure | Development, testing, desktop |
| **Examples** | VMware ESXi, KVM, Xen, Microsoft Hyper-V (Server) | VirtualBox, VMware Workstation, Parallels |
| **Attack surface** | Smaller - no host OS to compromise | Larger - host OS vulnerabilities expose VMs |

**KVM (Kernel-based Virtual Machine):**

- Linux kernel module that turns the kernel itself into a Type 1 hypervisor
- Uses hardware virtualization extensions (Intel VT-x, AMD-V)
- QEMU provides device emulation; libvirt provides management API
- Basis for most cloud providers (AWS Nitro uses KVM)

**Hyperjacking:**

- Attacker installs a rogue hypervisor beneath the legitimate OS
- The original OS runs as an unknowing guest, giving the attacker full control
- Mitigated by Secure Boot, TPM-based attestation, and hardware-rooted trust chains

!!! info "External Resources"
    - [KVM Documentation](https://www.linux-kvm.org/page/Main_Page) (Linux KVM)
    - [VMware vSphere Security Guide](https://docs.vmware.com/en/VMware-vSphere/7.0/com.vmware.vsphere.security.doc/GUID-52188148-C579-4F6A-8335-CFBCE0DD2167.html) (VMware)
    - [CIS Benchmark for VMware ESXi](https://www.cisecurity.org/benchmark/vmware) (CIS)

## Virtual Machine Security

**Isolation boundaries:**

- VMs share physical hardware but are logically isolated by the hypervisor
- Each VM has its own virtual hardware: CPU, memory, disk, network interface
- The hypervisor enforces memory isolation via hardware page tables (EPT/NPT)
- A compromised guest should not be able to read another guest's memory or escape to the host

**VM hardening best practices:**

- Remove unused virtual hardware (serial ports, USB controllers, floppy drives)
- Disable clipboard sharing and drag-and-drop in production
- Use encrypted VM disk images (vSphere VM Encryption, LUKS)
- Apply host and guest OS patches; keep hypervisor firmware updated
- Restrict VM management interfaces to dedicated management VLANs
- Enable Secure Boot for VMs where supported

**Live migration security:**

- vMotion/live migration transfers VM memory over the network
- Encrypt migration traffic (vMotion encryption in vSphere)
- Restrict migration to trusted hosts within the same cluster

!!! info "External Resources"
    - [NIST SP 800-125 - Guide to Security for Full Virtualization Technologies](https://csrc.nist.gov/publications/detail/sp/800-125/final) (NIST)
    - [NIST SP 800-125A - Security for Hypervisors](https://csrc.nist.gov/publications/detail/sp/800-125a/rev-1/final) (NIST)
    - [CIS Benchmark for KVM](https://www.cisecurity.org/benchmark/kernel-based_virtual_machine_kvm) (CIS)

## VM Escape & Side-Channel Attacks

**VM escape:**

- Attacker exploits a vulnerability in the hypervisor or virtual hardware to break out of the guest and execute code on the host
- Once on the host, all other VMs on that host are compromised
- Historical examples: CVE-2015-3456 (VENOM - virtual floppy drive buffer overflow in QEMU), CVE-2017-5715 (Spectre variant affecting VMs)

**Side-channel attacks:**

| Attack | What it exploits | Impact |
|--------|-----------------|--------|
| **Spectre** | Speculative execution in CPU | Read arbitrary memory across process/VM boundaries |
| **Meltdown** | Out-of-order execution, privilege check bypass | Read kernel memory from userspace |
| **L1TF (Foreshadow)** | L1 data cache, speculative execution | Read data from other VMs on same physical core |
| **MDS (Microarchitectural Data Sampling)** | CPU internal buffers | Leak data across security boundaries |
| **Rowhammer** | DRAM bit-flipping via repeated row access | Privilege escalation, VM escape |

**Mitigations:**

- CPU microcode updates and kernel patches (KPTI, retpolines)
- Disable hyperthreading for high-security workloads (prevents cross-thread leaks)
- Use hardware-assisted isolation (Intel TDX, AMD SEV for confidential VMs)
- Core scheduling - ensure mutually untrusted VMs do not share physical cores
- Keep hypervisor and firmware patched; monitor for new CVEs

!!! info "External Resources"
    - [Spectre and Meltdown - Graz University](https://meltdownattack.com/) (TU Graz)
    - [Intel TDX Documentation](https://www.intel.com/content/www/us/en/developer/tools/trust-domain-extensions/overview.html) (Intel)
    - [AMD SEV Documentation](https://www.amd.com/en/developer/sev.html) (AMD)

## Cloud Service Models (IaaS, PaaS, SaaS)

| Model | You manage | Provider manages | Examples |
|-------|-----------|-----------------|---------|
| **IaaS** | OS, runtime, apps, data | Hardware, hypervisor, networking | AWS EC2, GCP Compute Engine, Azure VMs |
| **PaaS** | Apps, data | OS, runtime, middleware, hardware | AWS Elastic Beanstalk, GCP App Engine, Heroku |
| **SaaS** | Data, user configuration | Everything else | Google Workspace, Salesforce, Slack |
| **FaaS/Serverless** | Function code | Everything including scaling | AWS Lambda, GCP Cloud Functions, Azure Functions |

**Security implications by model:**

- **IaaS** - most control, most responsibility. You patch the OS, configure firewalls, manage identities.
- **PaaS** - platform handles OS patching. Focus on application security, dependency management, access control.
- **SaaS** - least control. Security depends on vendor configuration, API security, data handling, integration security.
- **FaaS** - ephemeral execution reduces persistence risk but introduces cold-start trust, function chaining risks, and IAM complexity.

!!! info "External Resources"
    - [NIST SP 800-145 - Cloud Computing Definition](https://csrc.nist.gov/publications/detail/sp/800-145/final) (NIST)
    - [CSA Cloud Controls Matrix](https://cloudsecurityalliance.org/research/cloud-controls-matrix/) (Cloud Security Alliance)
    - [ENISA Cloud Computing Risk Assessment](https://www.enisa.europa.eu/topics/cloud-and-big-data/cloud-security) (ENISA)

## Shared Responsibility Model

The shared responsibility model defines the boundary between what the cloud provider secures and what the customer must secure.

**AWS shared responsibility model (representative):**

| Layer | AWS Responsibility | Customer Responsibility |
|-------|-------------------|----------------------|
| Physical infrastructure | Data centers, hardware, networking | N/A |
| Hypervisor / Host OS | Patching, hardening, isolation | N/A |
| Network infrastructure | VPC backbone, DDoS protection | Security groups, NACLs, VPC design |
| Guest OS | N/A (IaaS) | Patching, hardening, access control |
| Application | N/A | Code security, dependency management |
| Data | Encryption options provided | Classification, encryption at rest/transit, access control |
| Identity | IAM service provided | IAM policy design, MFA enforcement, least privilege |

**Common mistakes:**

- Assuming the provider handles everything ("it's in the cloud, so it's secure")
- Not encrypting data at rest because "the provider encrypts storage"
- Ignoring control plane security (API keys, console access, CloudTrail logging)
- Failing to monitor provider-level incidents and advisories

!!! info "External Resources"
    - [AWS Shared Responsibility Model](https://aws.amazon.com/compliance/shared-responsibility-model/) (AWS)
    - [Google Cloud Shared Responsibilities](https://cloud.google.com/architecture/framework/security/shared-responsibility-shared-fate) (Google Cloud)
    - [Azure Shared Responsibility](https://learn.microsoft.com/en-us/azure/security/fundamentals/shared-responsibility) (Microsoft)

## Cloud vs On-Prem Security Differences

| Dimension | On-Premises | Cloud |
|-----------|-------------|-------|
| **Perimeter** | Physical network boundary, firewalls | Identity is the perimeter; network is shared |
| **Provisioning speed** | Weeks to months | Minutes - increases risk of misconfiguration |
| **Visibility** | Full packet capture, physical access | API-based logging, provider-dependent telemetry |
| **Patch management** | Full control over timing | Provider patches infrastructure; customer patches apps/OS |
| **Blast radius** | Limited by physical segmentation | Single IAM misconfiguration can expose entire account |
| **Scaling** | Capacity planning required | Auto-scaling - also auto-scales attack surface |
| **Data sovereignty** | Known physical location | Multi-region by default; compliance implications |
| **Supply chain** | Hardware you purchased | Shared infrastructure, shared tooling risk |
| **Cost model** | CapEx (upfront) | OpEx (pay-as-you-go); runaway costs from attacks |
| **Forensics** | Full disk/memory access | Limited by provider APIs; volatile evidence |

**Key mental model shifts for cloud:**

- Speed increases risk - anyone with API credentials can provision infrastructure
- Shared tooling risk - a vulnerability in a cloud service affects all tenants
- Automation is mandatory - manual security reviews cannot keep pace with deployment velocity
- Assume breach, design for containment - focus on detection and blast radius reduction

!!! info "External Resources"
    - [NIST SP 800-144 - Guidelines on Security and Privacy in Cloud Computing](https://csrc.nist.gov/publications/detail/sp/800-144/final) (NIST)
    - [CSA Top Threats to Cloud Computing](https://cloudsecurityalliance.org/research/topics/top-threats/) (Cloud Security Alliance)
    - [Google Infrastructure Security Design Overview](https://cloud.google.com/docs/security/infrastructure/design) (Google Cloud)
