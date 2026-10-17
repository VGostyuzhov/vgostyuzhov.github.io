# Process Security

A process's authority includes its user/group credentials, namespaces, capabilities, open descriptors, applied mandatory policy, and inherited runtime state. Non-root execution is useful but does not summarize all of that authority.

## Credentials, Capabilities, and Execution

**Capabilities** divide privileged operations into kernel-defined permissions. Relevant sets include **effective** (used for current checks), **permitted** (available to make effective), **inheritable**, **ambient** (propagated across qualifying execution), and the **bounding set** (constrains capabilities obtainable through file execution). Their interaction at exec matters; a bit's presence does not grant unlimited power across all user namespaces.

**no_new_privs** prevents exec from gaining privilege through mechanisms such as set-ID/file capabilities. It does not remove current capabilities, undo writable mounts, revoke cloud credentials, or stop every service-level privilege change. Set it together with a narrow capability policy and the rest of the service's confinement.

```bash
# Read-only checks on an authorized Linux process and executable.
target_pid=1234
sed -n '/^Uid:/p; /^Gid:/p; /^Groups:/p; /^Cap/p; /^NoNewPrivs:/p; /^Seccomp/p' /proc/$target_pid/status
getcap /usr/local/bin/app  # File capabilities can differ from running process state.
# Resolve hex capability bits with the platform's capability tooling when available.
```

!!! info "External Resources"
    - [Capability Sets, Exec, and Namespaces](https://man7.org/linux/man-pages/man7/capabilities.7.html) (Linux man-pages)
    - [No New Privileges](https://docs.kernel.org/userspace-api/no_new_privs.html) (Linux kernel)

## Sandboxing a systemd Service

For a system service on supported systemd/Linux versions (per-user services have different support/privilege constraints), directives can reduce privileges and writable paths without pretending that the application is bug-free. The following is a **drop-in fragment**, not a complete runnable service; it assumes an installed `app.service` whose identity, executable, and required files are already defined.

```ini
[Service]
NoNewPrivileges=yes
CapabilityBoundingSet=
AmbientCapabilities=
ProtectSystem=strict
ProtectHome=yes
PrivateTmp=yes
StateDirectory=study-app
# ProtectSystem makes the service's filesystem view largely read-only.
# StateDirectory provides its managed writable state path under /var/lib.
MemoryHigh=192M
MemoryMax=256M
TasksMax=64
```

`PrivateTmp` changes the service's temporary-directory view; it does not authorize access to arbitrary host files or block network connections. `ProtectSystem` does not make every host filesystem globally read-only, and other mounts/writable exceptions matter. Inspect effective configuration and test required/rejected operations under the deployed version.

```bash
systemctl cat app.service
systemctl show app.service -p User -p NoNewPrivileges -p CapabilityBoundingSet -p MemoryMax -p TasksMax
systemd-analyze security app.service  # Heuristic review, not a proof of confinement.
```

!!! info "External Resources"
    - [systemd Execution Sandboxing](https://manpages.debian.org/bookworm/systemd/systemd.exec.5.en.html) (systemd)
    - [systemd Resource Controls](https://manpages.debian.org/bookworm/systemd/systemd.resource-control.5.en.html) (systemd)

## Resource Limits and cgroup v2

A shell's `ulimit` configures inherited resource limits, not every daemon/container on the machine. `RLIMIT_NPROC` counts processes/threads against a real UID under its rules and has privileged exemptions; a very low interactive limit can break ordinary sessions. PAM limits apply only where the application opens the relevant PAM session. A systemd service can have different limits from your shell.

**cgroup v2** applies controller policy to groups of processes. `memory.high` induces reclaim/throttling pressure; `memory.max` is a hard boundary that can cause cgroup OOM handling. An OOM kill is a result to investigate, not automatic evidence of an attack. `pids.max` counts tasks, including threads, and an ancestor cgroup can impose a tighter aggregate boundary than the workload's own setting. A cgroup namespace changes the exposed view rather than automatically setting a new resource limit. `pids.events` can supply evidence of rejected creation under the controller. CPU quotas throttle rather than guaranteeing a request's latency. Kernel/controller enablement, hierarchy, swap policy, and unbounded workloads still affect the host.

```text
Pod/container memory.max=256MiB; application attempts sustained 400MiB use
-> reclaim/limit pressure -> allocation failure or OOM handling under kernel policy
# Memory limit != safe request size; page cache and other charged memory also matter.
# CPU/memory limits do not automatically constrain every node resource or network path.
```

!!! info "External Resources"
    - [cgroup v2 Controllers and Memory Events](https://docs.kernel.org/admin-guide/cgroup-v2.html) (Linux kernel)
    - [Resource Limit Scope and Exemptions](https://man7.org/linux/man-pages/man2/getrlimit.2.html) (Linux man-pages)

## Monitoring and IPC

Virtual memory separation allows deliberate sharing through IPC. For a Unix-domain connection, inspect peer credentials under the appropriate namespace model and authorize the requested operation. Knowing that a caller runs as a service UID does not authorize every file or action. Tracing/ptrace access is separately constrained by credentials, dumpability, LSM policy, and namespaces.

```bash
# Read-only inspection; no process environment or memory dump is printed.
ps -eo pid,ppid,user,group,comm
ss -lxnp  # Visibility depends on privilege and namespace.
# Tracing a live service can perturb it and expose secrets; use a controlled fixture.
strace -f -e trace=openat,connect,execve /usr/bin/true
```

Use [audit and monitoring](logging-monitoring.md) for event retention and limitations. A seccomp filter constrains reachable syscalls; a successful trace of one execution is not a sufficient regression suite for a production filter.

## Diagnose Container Termination

An exit code reported as 137 commonly represents termination by SIGKILL (`128 + 9`); it does not identify who sent the signal or prove an OOM kill. Compare the runtime's termination reason, cgroup memory-event counters, kernel/node logs, and orchestrator events in the same time interval. Counters aggregate events over a cgroup lifetime; a nonzero value from earlier is not proof about this particular restart.

Kubernetes memory **requests** influence scheduling and memory **limits** constrain applicable runtime usage; a request is not a hard cap. Node-pressure eviction is a separate kubelet decision from a process killed under a memory cgroup limit. Restart policy and controllers determine what runs afterward. Do not infer malicious resource use or a universally correct new limit from one numeric exit code.

```bash
# Authorized Kubernetes lab: inspect existing evidence, do not generate memory pressure.
kubectl -n study describe pod app
kubectl -n study logs app --previous
# On a Linux host, inspect the matching actual cgroup path and time window.
cat /sys/fs/cgroup/example/memory.events
cat /sys/fs/cgroup/example/pids.events
# Paths are placeholders; correlate with the intended container, not an unrelated group.
```

!!! info "External Resources"
    - [Kubernetes Resource Requests and Limits](https://kubernetes.io/docs/concepts/configuration/manage-resources-containers/) (Kubernetes)
    - [Node-Pressure Eviction](https://kubernetes.io/docs/concepts/scheduling-eviction/node-pressure-eviction/) (Kubernetes)
