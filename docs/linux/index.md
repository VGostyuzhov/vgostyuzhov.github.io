# Linux OS Security

Linux security depends on the authority of a running process, the objects it can reach, and the kernel/service decisions that enforce access. A username, container label, compliance score, or encrypted disk is evidence about one part of that system, not a complete security boundary.

## Learning Path and Prerequisites

Start with basic shell syntax, processes, files/directories, and numeric identities. Work through the sequence below, then use the [14 tricky interview questions](interview-questions.md). Half of the questions apply the mechanisms to cloud/container hosts.

| Stage | Topics | What to explain afterward |
|---|---|---|
| File decisions | [File systems](file-system.md), [access controls](access-control-systems.md), [system paths](system-paths.md) | Why one operation succeeds and another fails on the same path |
| Authority | [Kernel/userspace](kernel-userspace.md), [users/groups](user-management.md), [process security](process-security.md) | Which credentials, capabilities, and enforcement layers a process actually has |
| Login and data | [Authentication](authentication.md), [encryption](encryption.md) | What a credential/key protects and which existing access it does not revoke |
| Operations | [Packages](package-management.md), [hardening](hardening-compliance.md), [logging](logging-monitoring.md) | How a change becomes active and what evidence supports the result |
| Network paths | [Firewalls](firewalls.md), [network security](network-security.md) | How listener, route, state, and namespace affect exposure |
| Workload isolation | [VM/container security](vm-container-sec.md), [attack vectors](attack-vectors.md) | Which host interfaces remain reachable and under whose authority |
| Reference and practice | [CLI tools](cli-tools.md), [interview questions](interview-questions.md) | Which observation or rejection test distinguishes competing explanations |

## Trace One Service Operation

```text
Request reaches service -> process handles request under current credentials
 -> resolves data path in its mount namespace
 -> applicable DAC/ACL/capability/LSM checks permit or deny access
 -> existing descriptor, resource limits, and runtime state affect later operations
 -> application authorizes the requested user's action separately
 -> configured logs/collector may retain evidence, subject to coverage/loss controls
# Non-root execution does not replace every subsequent check.
```

A Kubernetes pod can use that same host kernel while receiving different mount/PID/user views, capabilities, seccomp filters, and volume access. Changing the image or account database does not automatically rewrite every active kernel, credential, connection, or key in the system. The lessons emphasize these lifecycles and their observable limits.

Use [networking](../networking/index.md) for packet paths and [authentication architecture](../authentication/index.md) for application/session protocols. Keep Linux file/process authority separate from cloud IAM and Kubernetes API authorization when combining those models.
