# Access Control Systems

Linux can apply multiple checks to one operation. DAC/ACL permission, process capabilities, mandatory policy, mount restrictions, and syscall filtering answer different questions; an allow at one layer does not cancel a denial at another.

## Discretionary Access Control and ACLs

A process carries real/effective user and group IDs plus supplementary groups. Filesystem checks normally use filesystem credentials derived from them. Linux POSIX-style ACLs extend owner/group/other access with named users/groups and a **mask** limiting the group-class permissions. They are not an arbitrary ordered Windows-style deny/allow list.

For a typical access ACL: an owner match uses the owner entry; a named-user match uses that entry limited by the mask; applicable group entries are evaluated within the group class and mask; other applies when the earlier identity classes do not match. A named-user match does not fall back to more generous other permissions. Capabilities and MAC checks can further change the result.

```text
# Synthetic getfacl excerpt; Alice is a named user, NOT the file owner:
# owner: bob
# group: reports
user::rw-
user:alice:rw-          # effective:r-- because mask removes write
group::r--
mask::r--
other::---
# Alice's named rw- entry does not supply effective write access.
```

```bash
# Read-only inspection; getfacl displays effective mask restrictions.
getfacl /srv/reports/monthly.txt
id alice
namei -l /srv/reports/monthly.txt
# chmod of group-class bits on an extended ACL can change the ACL mask.
# setfacl can recalculate it unless its options/configuration say otherwise.
```

A directory **default ACL** supplies inherited access policy to newly created children; it does not retroactively update old files. Creation-mode restrictions still apply. Inspect both the parent's default ACL and the child's resulting access ACL.

!!! info "External Resources"
    - [ACL Evaluation, Mask, and Inheritance](https://manpages.debian.org/trixie/acl/acl.5.en.html) (Linux manual)
    - [Filesystem Credentials and Path Checks](https://man7.org/linux/man-pages/man7/path_resolution.7.html) (Linux man-pages)

## Mandatory Access Control

**Linux Security Modules (LSMs)** provide kernel hooks for security mechanisms. SELinux uses labels/types and a loaded policy; AppArmor profiles constrain programs with rules including paths and capabilities. These controls can restrict operations by privileged processes when the relevant policy covers them. Not every process is necessarily confined: inspect labels/profiles, mode, and policy exceptions.

| Concept | SELinux | AppArmor |
|---|---|---|
| Subject policy | Process security context/domain | Applied process profile |
| File policy | Object labels and type rules | Path-mediated profile rules under its semantics |
| Active enforcement | Enforcing with applicable domain policy | Enforce mode with applicable profile |
| Observation mode | Permissive logs denials without enforcing them | Complain mode generally logs would-be violations; explicit deny behavior differs |
| Investigation | Contexts, AVC denials, persistent labeling rules | Applied profile, logs, intended paths and permissions |

SELinux contexts include user, role, type, and optionally multi-level security (MLS) levels or multi-category security (MCS) categories. **Type enforcement** determines permitted domain/type operations. **MCS categories** can separate container labels where the policy and runtime configure them; a container label alone is not proof that all paths are covered.

## Investigate a Denial Before Widening Policy

Suppose a service can read Unix-mode permissions on a newly mounted data directory but receives `EACCES` under SELinux. Determine the actual process context, target label, operation, and Access Vector Cache (AVC) denial event. A wrong filesystem label may need a supported persistent labeling fix, rather than a new allow rule for an unintended type. DAC failure can also occur before an expected MAC denial appears.

```bash
# SELinux lab host only; read-only diagnosis (privileges may be required).
getenforce
ps -eZ
ls -Zd /srv/app /srv/app/data
ausearch -m AVC -ts recent
# Logs can include sensitive paths/arguments; restrict their distribution.
# Do not automatically install every rule suggested by audit2allow.
```

Review whether the operation belongs in the service contract, fix configuration/labels first, then add the narrow required policy if justified. Disabling enforcement globally can restore functionality by removing a boundary and is not a demonstrated fix. AppArmor requires the same reasoning with its applied profile and path rules.

!!! info "External Resources"
    - [SELinux Modes, Labeling, and Troubleshooting](https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/9/html/using_selinux/index) (Red Hat)
    - [AppArmor Profile Language](https://manpages.ubuntu.com/manpages/noble/en/man5/apparmor.d.5.html) (AppArmor manual)
