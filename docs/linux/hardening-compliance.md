# Linux Hardening and Compliance

A hardening baseline specifies selected configuration expectations for a distribution/version and workload role. A compliance result is evidence about those checks, not proof that software, identities, application logic, and every runtime path are secure.

## Tailor the Baseline to the Host Role

A Kubernetes node, developer workstation, database VM, and public web host need different services, privileges, mounts, and recovery paths. Record the versioned CIS/STIG or internal profile, control objective, expected evidence, and explicitly approved exceptions. Blindly disabling forwarding or writable paths can break a node while leaving the actual workload exposure unresolved.

**Mandatory policy** only confines operations covered by the loaded SELinux/AppArmor configuration and enforcement mode. Read [MAC diagnosis](access-control-systems.md#mandatory-access-control) before treating “package installed” or “SELinux enabled” as complete coverage. Likewise, filesystem/sysctl controls need actual mount/process/kernel verification.

!!! info "External Resources"
    - [CIS Benchmarks and Platform Profiles](https://www.cisecurity.org/cis-benchmarks/) (CIS)
    - [Security Content Automation and Assessment](https://www.open-scap.org/tools/openscap-base/) (OpenSCAP)

## Work One Finding Through to Evidence

```text
Finding: web worker can write a root-executed helper's configuration
Threat: service compromise changes what a privileged scheduled task executes
Correction: remove unintended write path; retain only required service state storage
Positive test: intended reporting job still completes
Negative test: web worker cannot modify helper/config/parent dependency paths
Rollout: canary, review denied operations, then expand
Exception: documented owner/scope/expiry if a temporary compatibility gap remains
# A scanner finding closes only when the intended boundary is tested, not just renamed.
```

For service confinement, compare the current unit, effective runtime credentials and mounts, and [systemd sandbox settings](process-security.md#sandboxing-a-systemd-service). `systemd-analyze security` provides a heuristic configuration review. A favorable score cannot establish whether the application should read a mounted credential, whether a network route is allowed, or whether external IAM grants are excessive.

## Fleet Rollout, Updates, and Reassessment

Use reproducible images/configuration, canary hosts, functional/negative tests, measured failures, and rollback/recovery. Track policy drift without automatically overwriting an intentional exception. Where remediation restarts a service or reboots a node, handle workload draining, quorum, and availability under the orchestrator's supported workflow.

Patch activation has multiple layers: update the node OS and running kernel, rebuild affected container images, deploy replacements, and restart still-running vulnerable processes as required. A kernel package installed on disk does not prove the active kernel was changed; a rebuilt image does not replace every running pod. See [package activation](package-management.md#verify-a-running-deployment).

Keep exceptions narrow, owned, time-bounded, and linked to compensating controls with observable evidence. Reassess after distribution/runtime/kernel changes rather than treating a once-passing report as permanent assurance.
