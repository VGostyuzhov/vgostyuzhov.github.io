# VM and Container Security: A Study Guide

## Container Security

### Linux Container Isolation
Containers are not "real" objects in the Linux kernel; they are a collection of processes isolated using several distinct kernel features.

#### 1. Namespaces
Namespaces wrap a global system resource in an abstraction that makes it appear to the processes within the namespace that they have their own isolated instance of the resource.

- **PID:** Isolates process IDs. Processes in a namespace cannot see or signal processes outside.
- **NET:** Isolates network interfaces, IP addresses, routing tables, and firewall rules.
- **MNT:** Isolates mount points. A container has its own view of the file system.
- **UTS:** Isolates hostname and NIS domain name.
- **IPC:** Isolates Inter-Process Communication (System V IPC, POSIX message queues).
- **USER:** Isolates user and group IDs. A user can be `root` (UID 0) inside a container but a non-privileged user on the host.
- **CGROUP:** Isolates the view of control groups.

#### 2. Control Groups (cgroups)
While namespaces provide isolation (what you can see), cgroups provide resource management (how much you can use).

- **Resource Limiting:** Limits on CPU, memory, I/O, and network bandwidth.
- **Prioritization:** Giving more CPU/bandwidth to certain containers.
- **Accounting:** Measuring how many resources a container is using.
- **Control:** Freezing and restarting groups of processes.
- **Security Impact:** Prevents "noisy neighbor" attacks and DoS by resource exhaustion.

#### 3. Linux Capabilities
Capabilities break down the power of the `root` user into smaller, distinct privileges. By default, containers should drop as many as possible.

- **CAP_CHOWN:** Make arbitrary changes to file UIDs and GIDs.
- **CAP_NET_ADMIN:** Perform various network-related operations (e.g., interface config, firewall).
- **CAP_SYS_ADMIN:** The "new root" - a catch-all for many privileged operations. **Highly dangerous.**
- **CAP_SYS_PTRACE:** Trace arbitrary processes using `ptrace`.

### Hardening with LSMs and Seccomp
Standard permissions are often insufficient for container isolation. Mandatory Access Control (MAC) and system call filtering add essential layers.

#### 1. Seccomp (Secure Computing Mode)
Seccomp allows a process to transition into a state where it cannot make any system calls except for a very limited set (like `read`, `write`, `exit`, `sigreturn`).

- **Seccomp-BPF:** A more flexible version that uses BPF filters to allow/deny syscalls based on arguments.
- **Default Profiles:** Docker and Kubernetes provide default seccomp profiles that block ~44 dangerous syscalls (e.g., `mount`, `reboot`, `ptrace`).

#### 2. AppArmor and SELinux
These Linux Security Modules (LSMs) provide fine-grained control over what processes can do.

- **AppArmor:** Path-based MAC. It uses profiles to restrict what files a process can read/write and what capabilities it can use.
- **SELinux:** Label-based MAC. Every object (file, process, socket) has a security label. Rules define which labels can interact.

### Container Escaping Techniques
An "escape" occurs when a process inside a container gains access to the host system.

1. **Privileged Containers:** Running with `--privileged` gives the container nearly all host capabilities and access to all devices in `/dev`.
2. **Docker Socket Mounting:** Mounting `/var/run/docker.sock` allows a container to talk to the Docker daemon and start new, privileged containers.
3. **Kernel Exploits:** Since containers share the host kernel, any kernel vulnerability (e.g., "Dirty Cow") can be used to escape.
4. **Misconfigured Mounts:** Mounting sensitive host paths (like `/etc`, `/root`, or `/proc`) can lead to privilege escalation.
5. **CAP_SYS_ADMIN:** Granting this capability is often enough to perform a mount-based escape or use `nsenter` to enter other namespaces.

## Virtual Machine Security

Virtual Machines run a full guest OS on a hypervisor. In the Linux ecosystem, **KVM** (Kernel-based Virtual Machine) is the primary technology.

### Hypervisors and Isolation

- **KVM:** A kernel module that turns the Linux kernel into a Type 1 hypervisor. It uses hardware extensions (Intel VT-x, AMD-V) to provide isolation.
- **Hyperjacking:** A theoretical attack where a rogue hypervisor is installed beneath a running OS.
- **VM Escape:** The most critical threat. An attacker exploits a vulnerability in the hypervisor (or virtualized hardware like QEMU) to break out of the guest and execute code on the host.

### Side-Channel Attacks
Because VMs share physical hardware (especially CPU caches and branch predictors), they are vulnerable to side-channel attacks.

- **Spectre and Meltdown:** Exploit speculative execution to leak data across process or VM boundaries.
- **L1 Terminal Fault (L1TF):** Leaks data from the L1 cache.
- **Mitigation:** Requires kernel patches (e.g., KPTI), microcode updates, and sometimes disabling Hyper-Threading (SMT).

## VM vs. Container Security Cheat Sheet

| Concern | Container Security | VM Security |
| :--- | :--- | :--- |
| **Primary Threat** | Kernel exploit leading to host compromise. | VM escape via hypervisor vulnerability. |
| **Isolation Level** | Weaker (shared kernel, namespaces). | Stronger (separate guest OS and kernel). |
| **Resource Control** | cgroups (soft/hard limits). | Hypervisor (strict allocation). |
| **Key Hardening** | Drop capabilities, Seccomp, AppArmor/SELinux. | Keep hypervisor patched, Secure Boot. |
| **Fatal Misconfiguration** | `--privileged` mode; mounting Docker socket. | Exposing management interface. |

!!! info "External Resources"
    *   **Linux Namespaces Man Page:** [https://man7.org/linux/man-pages/man7/namespaces.7.html](https://man7.org/linux/man-pages/man7/namespaces.7.html)
    *   **KVM Security Overview:** [https://www.linux-kvm.org/page/Security](https://www.linux-kvm.org/page/Security)
    *   **NSA/CISA Kubernetes Hardening Guide:** [https://www.nsa.gov/Press-Room/News-Highlights/Article/Article/2716980/nsa-cisa-release-kubernetes-hardening-guidance/](https://www.nsa.gov/Press-Room/News-Highlights/Article/Article/2716980/nsa-cisa-release-kubernetes-hardening-guidance/)
