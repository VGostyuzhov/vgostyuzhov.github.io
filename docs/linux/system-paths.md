# Critical System Directories

A path's security depends on the process's mount view, every traversed parent, the opened object, and policy. Directory names describe conventions, not automatic trust. Distribution layout and container/immutable-host design vary.

## Configuration, Executables, and Runtime State

| Path | Typical contents | Security question |
|---|---|---|
| `/etc` | Local configuration, account and service policy | Who can modify the file or replace it through its parent? |
| `/usr`, `/bin`, `/sbin` | Installed programs/libraries | Is this a merged-/usr layout, and are binaries/configuration trusted? |
| `/var/lib` | Persistent application state | Which service may read/write it, and what is backed up? |
| `/var/log` | Retained logs | Are capacity, access, forwarding, and rotation controlled? |
| `/run` | Volatile runtime data and sockets | Does a socket delegate privileged daemon authority? |
| `/proc`, `/sys` | Kernel/process interfaces, not ordinary disk data | Which namespace, visibility, capabilities, and write controls apply? |
| `/tmp`, `/var/tmp` | Temporary files under lifecycle conventions | Are creation, ownership, cleanup, and mount policies safe? |
| `/boot` | Boot artifacts | What validates boot integrity, and how are authorized updates performed? |

On a merged-/usr installation, `/bin` or `/sbin` can be symlinks into `/usr`. Inspect the resolved path rather than assuming separate protected partitions. Typical sensitive-file modes differ: `/etc/passwd` is generally readable; `/etc/shadow` is restricted; sudoers is commonly `0440` with owner/mode checks. Use distribution-supported settings instead of one blanket mode for `/etc`.

```bash
# Read-only Linux templates for permitted files.
namei -l /usr/local/sbin/report-helper
stat -c '%A %u:%g %n' /usr/local/sbin/report-helper
findmnt -T /usr/local/sbin/report-helper
# Also inspect any script, config, plugin, or working directory used by this helper.
```

!!! info "External Resources"
    - [Filesystem Hierarchy Standard 3.0](https://refspecs.linuxfoundation.org/FHS_3.0/fhs-3.0.html) (Linux Foundation; layout conventions)
    - [Path Resolution](https://man7.org/linux/man-pages/man7/path_resolution.7.html) (Linux man-pages)

## Temporary Files and Path Races

**Time of check to time of use (TOCTOU)** occurs when a program checks a name, an attacker changes what that name resolves to, and the program later uses the new object under old assumptions. Avoid predictable shared-directory files and independent check-then-open sequences; use supported atomic creation/opening APIs and trusted directory descriptors where required.

```python
# Runnable local fixture: exclusive temporary creation in a private directory.
from pathlib import Path
from tempfile import TemporaryDirectory
with TemporaryDirectory(prefix="study-private-") as directory:
    path = Path(directory) / "result.txt"
    with path.open("x") as output:  # Exclusive creation: fail if name already exists.
        output.write("harmless study data\n")
    print(path.read_text().strip())
# Private directory protects the parent path; 'x' alone is not every path defense.
```

Sticky mode limits eligible removals/renames; it does not stop reads of permissive files. `noexec` is not a complete interpreter-execution barrier. See [file and directory permissions](file-system.md).

!!! info "External Resources"
    - [Atomic Exclusive Creation and Path Flags](https://man7.org/linux/man-pages/man2/open.2.html) (Linux man-pages)
    - [Python Secure Temporary Files](https://docs.python.org/3/library/tempfile.html) (Python)

## Privileged Consumers and Boot Integrity

A root-run service/timer reading an unprivileged-writable script can execute attacker-controlled content without changing the root-owned service binary. Follow the full dependency chain, including drop-ins, environment/config files, executable search path, and runtime sockets. A socket under `/run` can be more powerful than a writable regular file if it accepts administrative requests.

Immutable attributes do not defeat a caller with the authority to clear them and can break account/update maintenance. Secure Boot checks the applicable boot trust chain; encrypting storage addresses a different confidentiality threat. Restrict boot configuration, signing authority, recovery access, and update paths according to the actual deployment.
