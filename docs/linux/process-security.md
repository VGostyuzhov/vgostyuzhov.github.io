## Process Security

### Basics and Core Concepts

**Process Isolation and Sandboxing**
Process isolation ensures that processes operate in separate **virtual address spaces**, preventing them from accessing each other's memory or interfering with execution. This is enforced by the kernel via **Memory Management Units (MMU)** and protection rings (Ring 0 for kernel, Ring 3 for user). Modern sandboxing extends this by virtualizing system resources to limit a compromised process's blast radius. **Linux Namespaces** (e.g., PID, NET, MNT) provide an isolated view of the system, while **Seccomp-BPF** (Secure Computing Mode) restricts the **system calls** a process can make, effectively reducing the kernel attack surface.

**Resource Limits and Quotas**
Unchecked resource consumption can lead to **Denial of Service (DoS)** attacks or system instability (e.g., "fork bombs"). Administrators enforce limits using **`ulimit`** (user-level shell limits) and **PAM** (`/etc/security/limits.conf`) to restrict open file descriptors, stack size, and process counts. For more granular control, **Control Groups (cgroups)** manage resource allocation (CPU, memory, I/O) for collections of processes. Cgroups are the fundamental mechanism behind container resource limiting, ensuring that a single application cannot starve the host or other containers.

**Process Monitoring and Auditing**
Effective detection requires immutable logging of process behavior and system calls. The **Linux Audit Framework (`auditd`)** is the standard subsystem for this, allowing administrators to write rules that intercept and log specific syscalls (e.g., `execve`, `connect`, `open`) and file access events. This data is critical for **forensic analysis** and identifying **Indicators of Compromise (IoC)**. Advanced monitoring often employs **eBPF** (Extended Berkeley Packet Filter) tools, which allow for high-performance, programmable tracing of kernel events with minimal system overhead.

**Signal Handling and IPC Security**
**Inter-Process Communication (IPC)** mechanisms—such as shared memory, pipes, message queues, and sockets—bypass process isolation to allow data exchange. Security here relies on strict **Access Control Lists (ACLs)** and proper permission bits (mode) on IPC objects to prevent unauthorized read/write access. **Signals** (asynchronous notifications like `SIGTERM`) require careful handling; poorly designed signal handlers can introduce **race conditions** or reentrancy vulnerabilities. Secure design dictates authenticating the other party (e.g., checking `SO_PEERCRED` in Unix sockets) before processing data.

---

### Cheatsheet: Process Security Mechanisms

| Feature | Primary Tool / Mechanism | Security Goal | Key Risk Mitigated |
| :--- | :--- | :--- | :--- |
| **Memory Isolation** | Virtual Memory / ASLR / DEP | Confidentiality & Integrity | Buffer overflows affecting other processes |
| **System View Isolation** | Namespaces (PID, NET, MNT) | Isolation | Container breakouts / Information Disclosure |
| **Attack Surface Reduction**| Seccomp-BPF | Least Privilege | Kernel exploitation via syscalls |
| **Resource Control** | Cgroups / `ulimit` | Availability | Denial of Service (DoS) / Fork bombs |
| **Auditing** | `auditd` / `auditctl` / eBPF | Non-repudiation | Unnoticed privilege escalation / Malware execution |
| **IPC Control** | DAC Permissions / SELinux | Access Control | Unauthorized data injection or snooping |

---

### External Resources for Deep Dive

!!! info "Essential Reading"
    * **[Docker Docs: Isolation with Namespaces and Cgroups](https://docs.docker.com/engine/security/userns-remap/)**
        * *Context*: Although focused on Docker, this is the canonical explanation for how modern Linux process isolation functions (Namespaces) and resource limiting (Cgroups) work in practice.
    * **[Red Hat: System Auditing with Auditd](https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/7/html/security_guide/chap-system_auditing)**
        * *Context*: The official documentation for the Linux Audit framework, covering rule creation (`auditctl`), log analysis, and configuration for compliance.
    * **[Seccomp Security Profiles for Docker](https://docs.docker.com/engine/security/seccomp/)**
        * *Context*: A practical look at how Seccomp filters are applied to restrict system calls, including standard profiles used in production environments.
    * **[Beej's Guide to Unix IPC](https://beej.us/guide/bgipc/)**
        * *Context*: A classic, technical deep-dive into the implementation details of IPC (signals, pipes, sockets), critical for understanding the developer's perspective on securing these channels.

### Practical Labs & Commands

**1. Resource Limiting (Mitigating Fork Bombs)**
Demonstrate how to prevent Denial of Service via process exhaustion using `ulimit` (shell-level) and `systemd` (cgroup-level).

* **Check current limits:**
    ```bash
    ulimit -a
    ```
* **Set a restrictive process limit (Session only):**
    This limits the user to 10 processes, effectively neutralizing a fork bomb in this session.
    ```bash
    ulimit -u 10
    ```
* **Ephemeral Cgroup Limit (Systemd):**
    Run a process with a hard memory cap using systemd scopes.
    ```bash
    # Runs 'stress' with a 50MB memory limit. OOM killer triggers if exceeded.
    systemd-run --scope -p MemoryMax=50M stress --vm 1 --vm-bytes 100M
    ```

**2. Process Isolation (Namespaces)**
Use `unshare` to manually create Linux namespaces, mimicking how containers (like Docker) work under the hood.

* **Create a shell in a new Network and PID namespace:**
    ```bash
    # --net: New network stack (will see only loopback)
    # --pid: New PID namespace (process usually becomes PID 1 inside)
    # --fork --mount-proc: Remount /proc to reflect the new namespace
    sudo unshare --net --pid --fork --mount-proc /bin/bash
    ```
* **Verify Isolation:**
    Inside the new shell, run `ps aux` (you should only see a few processes) and `ip a` (you should not see the host's network interfaces).

**3. System Call Tracing (Attack Surface Analysis)**
Use `strace` to audit exactly which system calls a binary uses. This is the first step in building a **Seccomp** whitelist profile.

* **Trace specific calls (File Open & Network Connect):**
    ```bash
    # -e trace=... : Filter specific syscalls
    # -f : Follow child processes (forks)
    strace -f -e trace=openat,connect,execve curl [http://example.com](http://example.com)
    ```
    *Interview Context*: You would analyze this output to determine that `curl` requires `connect`, but perhaps not `listen` or `accept`, allowing you to block those calls in production.

**4. System Auditing (Auditd)**
Configure the kernel audit subsystem to log unauthorized access to a sensitive file.

* **Create a temporary watch rule:**
    ```bash
    # -w: Watch file
    # -p wa: Trigger on Write (w) or Attribute change (a)
    # -k key_name: Tag for searching later
    sudo auditctl -w /etc/passwd -p wa -k identity_changes
    ```
* **Trigger and Search:**
    Touch the file (simulating access) and search the logs.
    ```bash
    sudo touch /etc/passwd
    sudo ausearch -k identity_changes
    ```

**5. IPC Analysis**
Inspect current Inter-Process Communication facilities to identify open shared memory segments or message queues.

* **List active IPC facilities:**
    ```bash
    ipcs -a
    ```
* **View detailed info on a specific Process ID:**
    Check `/proc` to see environment variables and status of a running process (e.g., PID 1234).
    ```bash
    cat /proc/1234/environ  # Check for leaked secrets in env vars
    cat /proc/1234/status   # Check Seccomp status and Capabilities
    ```