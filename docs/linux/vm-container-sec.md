# VM and Container Security

A Linux container is a set of processes configured through kernel/runtime mechanisms, not a separate guest kernel. A VM supplies a guest kernel behind a virtual-machine boundary. Both need protection of management interfaces, credentials, shared resources, and updates.

## Namespaces and User IDs

Namespaces change views of selected resources. A mount namespace changes visible mounts; a PID namespace changes process numbering/visibility; a network namespace changes the network stack. A user namespace maps IDs and scopes relevant capability checks. None of these automatically removes sensitive host mounts, inherited descriptors, service sockets, or all kernel vulnerabilities.

```bash
# Read-only Linux inspection of an authorized process.
target_pid=1234
readlink /proc/$target_pid/ns/user
readlink /proc/$target_pid/ns/mnt
readlink /proc/$target_pid/ns/pid
cat /proc/$target_pid/uid_map
cat /proc/$target_pid/gid_map
# UID 0 inside a user namespace can map to an unprivileged host UID.
# Inspect the mapping; UID 0 text alone does not identify host authority.
```

An ancestor PID namespace can see descendants; a child view is not a symmetric host-security guarantee. Shared/inherited namespaces, hostPID/hostNetwork, mount propagation, and privileged APIs change the boundary. A chroot changes pathname roots but is not a standalone sandbox for privileged code.

!!! info "External Resources"
    - [Namespace Types and Membership](https://man7.org/linux/man-pages/man7/namespaces.7.html) (Linux man-pages)
    - [User Namespace Mappings and Capability Scope](https://man7.org/linux/man-pages/man7/user_namespaces.7.html) (Linux man-pages)

## Controls That Must Be Combined

| Mechanism | Restriction | What it does not establish |
|---|---|---|
| Namespaces | Selected resource views | Permission to use every exposed object |
| cgroups | Enabled resource-controller limits/accounting | General network policy or complete DoS prevention |
| Capabilities | Specific privileged kernel operations | Safety merely because the process is non-root |
| Seccomp | Reachable syscalls/argument predicates | Path-aware authorization or a separate kernel |
| SELinux/AppArmor | Applicable mandatory confinement | Coverage of unconfined processes or every misconfigured path |
| Read-only root | Writes through container root filesystem | Read-only mounted volumes, no writable memory, or no network access |

cgroup v2 does not supply a universal built-in network-bandwidth controller; traffic shaping and network policy require the appropriate additional mechanisms. A kernel bug must be reachable and exploitable under the actual configuration to enable escape; “any kernel vulnerability escapes every container” is false. A runtime daemon socket with broad administrative authority can expose the host regardless of how non-root the application otherwise appears. Mounting the socket path read-only does not turn its protocol into a read-only API: a process that can connect may still ask the daemon to perform a mutating operation. The daemon's authorization, not the mount label alone, governs that delegated operation.

!!! info "External Resources"
    - [cgroup v2 Controller Scope](https://docs.kernel.org/admin-guide/cgroup-v2.html) (Linux kernel)
    - [Seccomp Filtering Is Not a Complete Sandbox](https://docs.kernel.org/userspace-api/seccomp_filter.html) (Linux kernel)
    - [Docker Daemon Attack Surface](https://docs.docker.com/engine/security/) (Docker)

## Kubernetes Workload Settings and Host Exposure

The following **Pod-spec fragment** assumes a supported Linux Kubernetes/runtime setup; it is not a complete deployable manifest. The application must tolerate a read-only root and have separately defined writable data/temporary volumes where necessary.

```yaml
securityContext:
  runAsNonRoot: true
  runAsUser: 10001
  runAsGroup: 10001
  seccompProfile:
    type: RuntimeDefault
containers:
  - name: app
    image: example.invalid/study-app:fixture
    securityContext:
      allowPrivilegeEscalation: false
      readOnlyRootFilesystem: true
      capabilities:
        drop: [ALL]
```

`runAsNonRoot` does not revoke filesystem access already granted to that UID/groups or cloud credentials exposed to it. `allowPrivilegeEscalation: false` controls no_new_privs under supported conditions; it is not a replacement for dropping existing privilege. `RuntimeDefault` requests the runtime's profile, not one fixed Kubernetes syscall deny-list. Actual enforcement depends on the runtime, kernel, workload configuration, and admission policy. Privileged workloads have important exceptions; do not assume the fragment overrides a privileged host-level deployment.

Kubernetes **Pod Security Standards Restricted** is an admission-policy baseline, not a complete workload-hardening configuration. For example, a read-only root filesystem is useful hardening but is not itself required by Restricted. It also does not establish network segmentation or per-resource application authorization. Define those separately.

A writable hostPath or node-management socket can expose host data/control. `fsGroup` can grant volume access through group/ownership behavior where supported; it does not mean non-root code is unable to read every mounted secret. Inspect actual mount permissions and the volume driver's behavior before concluding that storage is isolated.

!!! info "External Resources"
    - [Kubernetes Security Context Semantics](https://kubernetes.io/docs/tasks/configure-pod-container/security-context/) (Kubernetes)
    - [Pod Security Standards](https://kubernetes.io/docs/concepts/security/pod-security-standards/) (Kubernetes)
    - [Linux Kernel Controls for Pods](https://kubernetes.io/docs/concepts/security/linux-kernel-security-constraints/) (Kubernetes)

## Virtual Machines and Shared Hardware

KVM relies on the Linux kernel and CPU virtualization support; userspace device models such as QEMU are part of the attack surface. A guest can attack interfaces it is exposed to, including device emulation, shared filesystems, or agents. Minimize unnecessary devices/channels, patch host kernel/hypervisor/device models, and constrain management interfaces and identities.

VM isolation does not remove CPU side channels or host exhaustion. Spectre/Meltdown/L1TF mitigations depend on the specific vulnerability, CPU, kernel/hypervisor, microcode, and deployment requirements; KPTI or disabling SMT is not a universal fix for all speculative-execution issues. Test the vendor-supported mitigation set and capacity impact. Secure Boot validates an applicable boot trust chain; it is not proof that a running guest or host remains uncompromised.

!!! info "External Resources"
    - [KVM Security and Vulnerability Reporting](https://www.linux-kvm.org/page/Security) (KVM)
    - [Hardware Vulnerability Mitigations](https://docs.kernel.org/admin-guide/hw-vuln/index.html) (Linux kernel)
