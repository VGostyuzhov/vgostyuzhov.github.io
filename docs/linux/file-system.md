# File System Security

Linux filesystem security combines pathname traversal, object permissions, process credentials, mount policy, and any active mandatory policy. A mode such as `0644` answers only part of the access question. Read [access-control evaluation](access-control-systems.md) for ACL and MAC interactions.

## File Permissions and Ownership

**Discretionary Access Control (DAC)** uses numeric owner/group IDs and permission classes. Ordinary checks select the applicable owner, group, or other permissions; they do not simply combine all three. ACLs extend that decision. Directory permissions have different meanings from regular-file permissions:

| Bit | Regular file | Directory |
|---|---|---|
| Read | Read contents | List entry names |
| Write | Modify contents | Create/remove/rename entries, normally requiring search permission too |
| Execute/search | Execute under supported program format and policy | Traverse the directory when resolving a pathname |

A user can read a file only if the path can be traversed and the final file/policy permits it. Removing a name is primarily a parent-directory operation: a read-only file can still be removed from a writable/searchable directory, subject to sticky-bit, immutable, mount, and other checks.

```bash
# Read-only Linux diagnosis; substitute a permitted lab path.
namei -l /srv/app/config/settings.conf  # Inspect every path component.
stat -c '%A %a %u:%g %n' /srv/app/config/settings.conf
getfacl /srv/app/config/settings.conf  # ACL mask/entries can affect access.
findmnt -T /srv/app/config/settings.conf -o TARGET,FSTYPE,OPTIONS
# A correct final-file mode does not repair an unsearchable parent directory.
```

**Creation defaults:** A program requests a mode, then the umask normally removes permission bits: `requested & ~umask`. Requested `0666` with umask `0027` yields `0640`; requested `0777` yields `0750`. Umask does not add execute bits. A parent default ACL changes inheritance behavior, so inspect it before predicting the result from umask alone. `chmod` changes an existing object; changing umask does not rewrite old files.

!!! info "External Resources"
    - [Path Resolution and Search Permission](https://man7.org/linux/man-pages/man7/path_resolution.7.html) (Linux man-pages)
    - [Unlink and Directory Permissions](https://man7.org/linux/man-pages/man2/unlink.2.html) (Linux man-pages)
    - [Umask and Default ACL Interaction](https://man7.org/linux/man-pages/man2/umask.2.html) (Linux man-pages)

## Special File Types and Attributes

**Set-user-ID (SUID) / set-group-ID (SGID)** executable bits can change effective user/group credentials during execution under applicable kernel rules. They do not mean that any marked file always gains privilege: `nosuid`, `no_new_privs`, tracing, executable format, and other rules can prevent the transition. Linux ignores set-ID bits on interpreter scripts. A setgid directory normally gives newly created children its group; it does not give every process that enters the directory that group's authority.

**Sticky directories**, such as a conventional `/tmp` with mode `1777`, restrict removal/renaming to the file owner, directory owner, or appropriately privileged callers. They do not make a world-readable temporary file confidential or stop an attacker precreating a predictable name. Use atomic exclusive creation through a maintained temporary-file API and restrictive permissions.

| Object | Meaning | Check |
|---|---|---|
| Symlink | A name referring to another pathname | Resolution can follow it; validate the opened object, not an earlier string check |
| Device node | Special file representing a device interface | Access and mount policy determine whether it can be used |
| Unix socket / FIFO | Local communication endpoint | Peer identity and operation authorization remain necessary |
| Immutable attribute | Blocks relevant modifications until cleared | A sufficiently privileged caller can clear it; support varies by filesystem |
| Append-only attribute | Restricts file modification operations | Does not create tamper-proof remote evidence or prevent privileged reconfiguration |

Do not set immutable flags on live authentication files as a generic hardening step: legitimate account changes, rotation, and recovery can break. Treat these attributes as operational controls with an explicit change procedure.

!!! info "External Resources"
    - [execve Credential Transitions](https://man7.org/linux/man-pages/man2/execve.2.html) (Linux man-pages)
    - [Sticky-Bit and Mode Semantics](https://man7.org/linux/man-pages/man2/chmod.2.html) (Linux man-pages)
    - [Filesystem Attributes](https://man7.org/linux/man-pages/man1/chattr.1.html) (e2fsprogs manual)

## Mount Points and Filesystem Options

A mount exposes a filesystem at a path in a particular mount namespace. Check the mount visible to the process: another service/container can see a different tree.

| Option | Restriction | Limitation |
|---|---|---|
| `nosuid` | Suppresses applicable set-ID/file-capability privilege gains | Does not remove existing process privilege |
| `nodev` | Disallows interpretation of device special files on that mount | Does not block devices exposed elsewhere |
| `noexec` | Prevents direct execution from this mount | An interpreter on another mount can read/run a script; not a complete code-execution boundary |
| `ro` | Prevents filesystem writes through this mount | Other writable mounts, descriptors/paths, or privileged remounts need separate analysis |

Separate `/var/log` or data filesystems can constrain capacity failure and permit different options, but need quotas/monitoring and application compatibility checks. A read-only `/boot` can interfere with updates unless the workflow deliberately handles it. A container read-only root does not automatically make mounted volumes read-only.

```text
/tmp (noexec): script.sh cannot be directly execve'd from that mount
/usr/bin/sh /tmp/script.sh: shell can read script data if permissions allow
# Preventing the first path does not imply the second is impossible.
```

!!! info "External Resources"
    - [Mount Flags and Filesystem-Specific Behavior](https://man7.org/linux/man-pages/man8/mount.8.html) (util-linux)
    - [Mount Namespaces](https://man7.org/linux/man-pages/man7/mount_namespaces.7.html) (Linux man-pages)

## File System Hardening Techniques

Define which identity must read/write each path, identify trusted parent directories, minimize writable executable/configuration paths, and check ACLs and mandatory policy. Establish a trusted file-integrity baseline and review changes against package updates and approved deployments. A hash recorded after compromise cannot prove the original file was legitimate. Back up keys/configuration and test recovery before tightening mounts or removing access.
