# Kernel and Userspace

The CPU and memory-management hardware separate ordinary process execution from privileged kernel execution. This boundary reduces direct access to protected resources; it does not guarantee that kernel bugs, unsafe device interfaces, or misconfigured privileges cannot compromise the host.

## From Application Call to Kernel Decision

A userspace function such as a library file-open routine eventually requests a kernel operation, often through a syscall. Some library calls are implemented partly in userspace/vDSO, and events also enter the kernel through interrupts/exceptions. Not every kernel interaction is one explicit application syscall.

```text
Application: openat(..., "/srv/app/config", O_RDONLY)
  -> CPU enters kernel through syscall mechanism
  -> resolve pathname under process root/mount namespace
  -> evaluate credentials, DAC/ACL, applicable LSM and mount checks
  -> return descriptor or error such as EACCES/ENOENT
Application: read(fd, ...) -> kernel checks/uses the open object
# A descriptor refers to an opened object, not merely the original path string.
```

Virtual address spaces separate normal process memory mappings. Explicit shared memory, tracing permissions, device mappings, and kernel-mediated IPC permit controlled sharing. Root in userspace is still userspace; it can request broader operations through credentials/capabilities but does not execute every instruction in kernel mode.

!!! info "External Resources"
    - [System Calls and Library Wrappers](https://man7.org/linux/man-pages/man2/syscalls.2.html) (Linux man-pages)
    - [Path Resolution](https://man7.org/linux/man-pages/man7/path_resolution.7.html) (Linux man-pages)

## Kernel Attack Surface and Mitigations

A reachable syscall/device/filesystem interface can contain a kernel vulnerability. Patch the running kernel, reduce exposed features and excessive capabilities, and layer memory-exploitation mitigations with confinement. ASLR changes address predictability; non-executable memory constrains code execution from data pages; neither proves a memory-corruption bug is harmless.

**Seccomp filtering** evaluates syscall numbers and accessible argument values. It can allow, reject, or otherwise handle selected calls, but it does not establish a complete sandbox or generally inspect pointed-to pathname contents. A syscall trace from one happy-path run is not an exhaustive allow-list: error handling, threads, updates, and architecture variants change requirements.

```bash
# Read-only observation of an authorized existing Linux process.
target_pid=1234
sed -n '/^Uid:/p; /^Gid:/p; /^Cap/p; /^NoNewPrivs:/p; /^Seccomp/p' /proc/$target_pid/status
# Seccomp: 2 indicates filter mode, not which operations the filter permits.
# CapEff reports effective capability bits, not proof of host-level authority.
```

!!! info "External Resources"
    - [Seccomp Filters and Limitations](https://docs.kernel.org/userspace-api/seccomp_filter.html) (Linux kernel)
    - [Process Status Fields](https://man7.org/linux/man-pages/man5/proc_pid_status.5.html) (Linux man-pages)
