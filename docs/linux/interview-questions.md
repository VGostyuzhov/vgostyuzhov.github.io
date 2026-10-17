# Tricky Linux OS Security Interview Questions

Fourteen scenarios combine seven Linux foundations with seven cloud/container-host scenarios. Start with the linked lessons; explain the mechanism and failed assumption before opening the answer. Each worked example is a separate expandable block with comments and expected outcomes.

Synthetic traces, paths, images, and names are fixtures. Commands are inspection templates for authorized Linux/Kubernetes labs unless explicitly identified as a runnable local example. No host policy, cloud resource, or cluster is changed by this page.

## 1. Can a Read-Only File Be Deleted?

**Study first:** [File versus directory permissions](file-system.md#file-permissions-and-ownership).

**Scenario:** Alice owns a writable/searchable directory containing a file with mode 0444. She cannot modify its bytes through an ordinary write open, but can remove its name. Is DAC broken?

??? note "Answer guidance and common trap"
    No. Writing file contents and unlinking a name are different operations. Parent-directory write/search permissions normally govern removal, with sticky-bit, immutable, mount, and mandatory policy qualifications. Do not “fix” deletion risk by changing only the file's write bit.

    **Trap:** File write permission is treated as ownership of the directory entry.

    **Test:** In a private unprivileged lab directory, compare content-write and unlink behavior; separately examine a sticky shared directory's eligible owners.

??? example "Worked example"
    Runnable local filesystem fixture; only a newly created temporary directory is changed:

    ```python
    from pathlib import Path
    from tempfile import TemporaryDirectory
    with TemporaryDirectory(prefix="study-unlink-") as directory:
        path = Path(directory) / "read-only.txt"
        path.write_text("fixture")
        path.chmod(0o444)  # File is read-only; parent remains owned/writable/searchable.
        path.unlink()     # Removing the directory entry need not write the file contents.
        print(path.exists())  # False
    # Assumes ordinary Unix-like filesystem semantics and no overriding policy denial.
    ```

    This does not demonstrate that every user can remove every read-only file; the parent and caller are deliberately controlled.

**Follow-up:** Does a sticky bit make another user's readable temporary file secret? No; it restricts eligible removal/rename operations, not content reads.

## 2. Why Does a Named ACL Grant Still Fail?

**Study first:** [ACL mask and selection](access-control-systems.md#discretionary-access-control-and-acls).

**Scenario:** A file ACL grants user alice rw-, but its mask is r--. Alice cannot write even though a reviewer sees rw- next to her name. Which entry matters?

??? note "Answer guidance and common trap"
    The named-user entry is constrained by the ACL mask, yielding effective read-only access. POSIX ACL evaluation is not an ordered list whose last grant wins. Inspect the applicable identity class, mask, parent traversal, and mandatory policy before widening permissions.

    **Trap:** An ACL entry's requested bits are equated with its effective access.

    **Test:** Confirm effective permissions in getfacl and check the actual named user's operation; do not test only as a privileged administrator.

??? example "Worked example"
    Synthetic getfacl excerpt, not a rule to install. Bob owns the file; Alice matches the named-user entry:

    ```text
    # owner: bob
    # group: reports
    user::rw-
    user:alice:rw-         # effective:r--
    group::r--
    mask::r--
    other::---
    ```

    ```bash
    getfacl /srv/reports/monthly.txt
    id alice
    namei -l /srv/reports/monthly.txt
    # chmod of the group-class bits can change the extended ACL mask.
    # Inspect default ACLs separately when diagnosing newly created files.
    ```

**Follow-up:** Would a default ACL repair an already-created child file automatically? No; distinguish creation-time inheritance from existing-object changes.

## 3. Why Is EACCES Possible When Unix Permissions Look Correct?

**Study first:** [Mandatory policy](access-control-systems.md#mandatory-access-control) and [denial investigation](access-control-systems.md#investigate-a-denial-before-widening-policy).

**Scenario:** A daemon can traverse a path and its DAC/ACL permissions permit a read, but SELinux denies the operation after a data mount was changed. Should the team disable enforcement?

??? note "Answer guidance and common trap"
    No. Identify the actual domain, target label, operation, and denial evidence. A wrong label or unintended path can explain the failure. Correct supported configuration/labeling before adding a narrowly justified policy rule. An allow at DAC does not override MAC.

    **Trap:** A functionality failure is resolved by removing the security boundary globally.

    **Test:** The intended labeled data path works after correction while an unrelated sensitive path remains denied.

??? example "Worked example"
    Synthetic AVC-style fields for analysis; exact policy/types depend on the system:

    ```text
    type=AVC ... denied { read } ...
    scontext=system_u:system_r:httpd_t:s0
    tcontext=system_u:object_r:default_t:s0 tclass=file
    # Identify why web-service code reached a default-labeled file.
    # This is evidence to investigate, not an automatic new allow-rule specification.
    ```

    ```bash
    getenforce
    ls -Zd /srv/app/data
    ausearch -m AVC -ts recent
    # Check persistent labeling intent rather than blindly accepting audit2allow output.
    ```

**Follow-up:** If no matching AVC is visible, does that prove MAC was uninvolved? No; consider visibility/logging and other earlier failures, including DAC.

## 4. Does Removing a Group Membership Revoke an Existing Process?

**Study first:** [Running credentials](user-management.md#accounts-and-running-credentials).

**Scenario:** A user is removed from a privileged Unix group. A service or shell started before removal still accesses a group-readable file. Why?

??? note "Answer guidance and common trap"
    Existing processes can retain their earlier supplementary group list. Updating NSS/local/directory membership changes what future sessions may obtain, not every running credential. Define and enforce the required lifecycle for sessions, workers, privileged shells, external keys/tokens, and any open descriptors.

    **Trap:** An identity-database change is mistaken for immediate rewriting of all process credentials.

    **Test:** Compare a new authorized login with an existing process's Groups field and access result; coordinate termination or restart under the actual revocation contract.

??? example "Worked example"
    Synthetic identity timeline:

    ```text
    09:00  worker starts with supplementary GID 2200
    09:10  operator removes user's GID 2200 membership from directory
    09:11  new session lacks GID 2200; old worker still reports it
    # Database lookup and running credentials are separate evidence.
    ```

    ```bash
    id alice
    getent group reports
    target_pid=1234
    sed -n '/^Uid:/p; /^Gid:/p; /^Groups:/p' /proc/$target_pid/status
    # Commands are read-only templates on an authorized Linux host.
    ```

**Follow-up:** Will locking the password terminate an old root shell or revoke an SSH key? Not by itself; identify the credential and session being accepted.

## 5. Why Does SSH Still Accept a Password-Like Prompt?

**Study first:** [PAM flow](authentication.md#pam-and-local-authentication) and [SSH methods](authentication.md#ssh-user-authentication-and-host-identity).

**Scenario:** PasswordAuthentication is set to no, but a keyboard-interactive PAM path still accepts a password. Was the directive ignored?

??? note "Answer guidance and common trap"
    Not necessarily. SSH authentication methods and PAM backend behavior are distinct. Evaluate the effective Match-specific configuration, keyboard-interactive settings, UsePAM, AuthenticationMethods, and module control flow. Public key alone is not automatically MFA; an intended second factor must be required on every relevant successful path.

    **Trap:** One directive is treated as a proof covering all password or recovery paths.

    **Test:** Try each permitted method in an authorized lab, including missing-first-factor and missing-second-factor cases, and retain a controlled recovery path.

??? example "Worked example"
    Read-only configuration checks; sufficient privilege and the intended config are assumed:

    ```bash
    sshd -t
    sshd -T -C user=alice,host=client.example,addr=192.0.2.8
    # Review passwordauthentication, kbdinteractiveauthentication, usepam,
    # authenticationmethods, and applicable Match/include behavior.
    ```

    ```text
    PasswordAuthentication no       -> disables that SSH method
    Keyboard-interactive + PAM auth -> can still accept a password-like response
    Earlier sufficient PAM success  -> may skip a later auth module
    # Conceptual control flow, not an MFA configuration to paste into production.
    ```

**Follow-up:** Does correct user authentication prove server identity? No; the client still needs trusted host-key checking.

## 6. Can a Correctly Signed Repository Deliver Malicious Software?

**Study first:** [Repository trust chains](package-management.md#repository-trust-chains).

**Scenario:** APT verifies repository metadata and the downloaded package hash. The trusted publisher was compromised and intentionally signed an unwanted update. Which promise failed?

??? note "Answer guidance and common trap"
    Signature/hash verification can succeed: it establishes integrity under an accepted publisher trust chain, not benign intent. Scope keys/repositories, protect publishing/build authority, review privileged installation paths, and detect/recover from signer compromise. HTTPS alone neither replaces the trust chain nor fixes a compromised publisher.

    **Trap:** Authenticated origin is equated with safe software.

    **Test:** Distinguish tampering with a package after signing (hash/chain failure) from an authentically published malicious package (trust-policy/incident problem).

??? example "Worked example"
    APT trust-chain fixture:

    ```text
    Accepted repository key -> signed InRelease -> Packages index hash -> .deb hash
    Compromised signer publishes malicious package + matching metadata -> checks can pass
    Altered .deb with unchanged trusted metadata                      -> hash mismatch
    # APT normally does not require a separate publisher signature on every .deb.
    ```

    ```bash
    apt-cache policy openssl
    # Inspect candidate origin/version; importing a global third-party key can widen trust.
    # This read-only command does not install, approve, or independently audit a package.
    ```

**Follow-up:** Does installing a fixed package prove running processes use the fixed code? No; verify activation, restarts, and running-kernel state.

## 7. Does a New LUKS Passphrase Invalidate an Old Header Backup?

**Study first:** [LUKS keys and backups](encryption.md#block-and-file-encryption).

**Scenario:** A volume passphrase is replaced, but an attacker has an earlier header backup and its old passphrase. The volume key was not changed. Are the old unlock materials necessarily useless?

??? note "Answer guidance and common trap"
    No. Changing a keyslot's unlocking material normally does not rewrite the volume key. An old header can preserve a route to that same key and data. Treat header/recovery backups as sensitive key-management artifacts, and plan revocation/reencryption according to the required threat model.

    **Trap:** Changing a passphrase is assumed to replace every historical copy of the data key.

    **Test:** Use disposable encrypted fixtures to test the supported keyslot/recovery lifecycle. Do not experiment with the only copy of real data or publish unlock material.

??? example "Worked example"
    Conceptual cryptographic lifecycle; no live passphrases/keys or destructive commands:

    ```text
    Volume data encrypted under key K
    Old header: keyslot protects K using old passphrase P1
    New header: keyslot protects K using new passphrase P2
    Attacker retains old header + P1 -> old material can still recover K
    # Current keyslot removal does not revoke independently retained historical headers.
    ```

    ```bash
    cryptsetup status example-mapping
    # Read-only template: an open mapping means plaintext access is governed by runtime policy.
    ```

**Follow-up:** Does disk encryption stop root or an authorized application from reading an unlocked volume? No; distinguish offline theft from running-host access.

## 8. Can a Non-Root Container Control a Privileged Runtime Daemon?

**Study first:** [Container authority boundaries](vm-container-sec.md#controls-that-must-be-combined) and [runtime sockets](system-paths.md#privileged-consumers-and-boot-integrity).

**Scenario:** A pod runs as UID 10001 with dropped capabilities, but a mounted runtime-management socket accepts privileged requests from it. Is host control prevented by non-root execution?

??? note "Answer guidance and common trap"
    No. The process can exercise authority delegated by the socket's daemon/API. Its own UID/capability restrictions do not automatically constrain what a more privileged daemon performs on its behalf. Remove unnecessary management-socket exposure and constrain the actual daemon's authorization; do not demonstrate this by creating privileged containers on a live host.

    **Trap:** Local process identity is treated as the maximum authority available through all reachable services.

    **Test:** An ordinary application workload cannot reach/use the administrative runtime API, while intended platform management still works.

??? example "Worked example"
    Synthetic mount/access inventory, not a deployment recommendation:

    ```text
    Process: UID=10001, CapEff=0
    Mount:   /run/runtime-admin.sock -> privileged node daemon
    Mode:    socket group grants UID10001's group access
    Effect:  daemon accepts administrative operation from this process
    # Zero effective capabilities does not remove a separately delegated daemon API.
    ```

    ```bash
    target_pid=1234
    sed -n '/^Uid:/p; /^Gid:/p; /^Groups:/p; /^Cap/p' /proc/$target_pid/status
    # Inspect actual mount/socket permissions under the intended namespace too.
    # Do not invoke destructive daemon APIs as a discovery step.
    ```

**Follow-up:** Would mounting the socket read-only necessarily make its API read-only? No; filesystem mount write policy is not the daemon's operation-level authorization.

## 9. Do no_new_privs, Dropped Capabilities, and Seccomp Mean the Same Thing?

**Study first:** [Process privilege controls](process-security.md#credentials-capabilities-and-execution) and [Kubernetes settings](vm-container-sec.md#kubernetes-workload-settings-and-host-exposure).

**Scenario:** A workload sets allowPrivilegeEscalation=false, drops capabilities, and uses RuntimeDefault seccomp. Which distinct risks do these controls address?

??? note "Answer guidance and common trap"
    The no_new_privs setting constrains exec-based privilege gains; capability configuration removes specified privileged operations; seccomp filters reachable syscalls under the runtime profile. They do not revoke all mounted-file permissions, available cloud credentials, network destinations, or application authority. Inspect actual enforcement rather than equating one field with all three.

    **Trap:** One “restricted” label is assumed to summarize every runtime authority.

    **Test:** Verify relevant runtime status and intended denied operations under the actual runtime version, while required application behavior still succeeds.

??? example "Worked example"
    Container securityContext fragment; pod identity/mounts and the rest of the manifest are omitted:

    ```yaml
    securityContext:
      allowPrivilegeEscalation: false
      capabilities:
        drop: [ALL]
      seccompProfile:
        type: RuntimeDefault
    # no_new_privs does not remove privileges already held.
    # RuntimeDefault is runtime-dependent, not one fixed Kubernetes syscall list.
    ```

    ```text
    Sensitive readable volume remains mounted -> app can still read it if policy permits
    Reachable cloud credential source         -> IAM authority needs separate restriction
    # The three kernel controls do not automatically deny these application capabilities.
    ```

**Follow-up:** Does Restricted Pod Security admission guarantee a read-only root or NetworkPolicy? No; these are separately configured requirements.

## 10. Does Container Exit 137 Prove an OOM Attack?

**Study first:** [Container termination diagnosis](process-security.md#diagnose-container-termination).

**Scenario:** A Kubernetes container exits with code 137. An operator doubles its memory limit and reports a memory-exhaustion attack. What evidence is missing?

??? note "Answer guidance and common trap"
    137 commonly encodes SIGKILL, not its cause or attacker intent. Correlate termination reason, cgroup event deltas, kernel/node logs, pod events, workload behavior, and time. A cgroup OOM, manual/runtime kill, and node-pressure eviction are not interchangeable. Choose remediation after identifying the constrained resource and legitimate demand.

    **Trap:** One exit code is treated as both root-cause analysis and an attack verdict.

    **Test:** Use synthetic or controlled resource fixtures and establish which evidence distinguishes the alternatives, without generating node pressure in production.

??? example "Worked example"
    Synthetic correlated evidence, not live command output:

    ```text
    Container: exitCode=137, reason=OOMKilled, finishedAt=10:00
    Matching cgroup: memory.events oom_kill increased 0 -> 1 near 10:00
    Node/pod evidence: no node-pressure eviction for this event
    # Together these support this container's OOM path, not necessarily malicious intent.
    # A counter already nonzero yesterday would not establish today's cause.
    ```

    ```bash
    kubectl -n study describe pod app
    kubectl -n study logs app --previous
    # Missing logs can mean retention/access or restart behavior; not proof of no incident.
    ```

**Follow-up:** Would increasing the memory request impose a hard allocation cap? No; distinguish scheduling requests from runtime limits and node-pressure handling.

## 11. Why Does a Container Hit a PID Limit When ulimit Looks High?

**Study first:** [rlimits versus cgroups](process-security.md#resource-limits-and-cgroup-v2).

**Scenario:** A workload cannot create another thread although its shell reports a high process limit. Its cgroup task limit is already reached. Which limit governs?

??? note "Answer guidance and common trap"
    Independent limits can reject task creation. RLIMIT_NPROC is inherited and evaluated with real-UID task accounting/exemptions; a cgroup pids limit constrains its hierarchy. Threads count as tasks under the cgroup controller. A high shell limit does not override a lower cgroup boundary or an exhausted ancestor.

    **Trap:** One process's ulimit output is treated as every host/container policy.

    **Test:** Compare actual workload/ancestor cgroup state, event changes, and per-process limits before changing the runtime contract.

??? example "Worked example"
    Synthetic cgroup v2 snapshot:

    ```text
    Workload rlimit: high enough for next task
    pids.current: 64
    pids.max:     64
    pids.events:  max increased after clone attempt
    # Another thread/process can fail even though the shell's ulimit is not reached.
    # Parent cgroups can impose further aggregate restrictions.
    ```

    ```bash
    target_pid=1234
    cat /proc/$target_pid/limits
    cat /proc/$target_pid/cgroup
    # Inspect the resolved cgroup hierarchy; do not demonstrate with a fork bomb.
    ```

**Follow-up:** Does a cgroup namespace automatically change the actual task limit? No; it changes the exposed view; controller settings enforce resource limits.

## 12. Why Can a Pod with a Read-Only Root Still Modify a Volume?

**Study first:** [Mount policy](file-system.md#mount-points-and-filesystem-options) and [workload volume permissions](vm-container-sec.md#kubernetes-workload-settings-and-host-exposure).

**Scenario:** A container sets readOnlyRootFilesystem=true but writes to a separately mounted writable data volume. Its UID/group has permission there. Did Kubernetes ignore the setting?

??? note "Answer guidance and common trap"
    No. The root filesystem policy does not make every independent mount read-only. Trace the process mount view, volume flags, numeric UID/group mapping, ACL/MAC policy, and driver behavior. fsGroup can grant applicable volume access; non-root status does not make all mounted data inaccessible.

    **Trap:** Read-only root is treated as a global no-write or no-secret-access policy.

    **Test:** An intended state path is writable while protected root paths and unrelated sensitive volumes are denied under the actual process credentials.

??? example "Worked example"
    Container-spec fragment; the named volume is defined elsewhere in the pod:

    ```yaml
    name: app
    image: example.invalid/study-app:fixture
    securityContext:
      readOnlyRootFilesystem: true
      runAsUser: 10001
    volumeMounts:
      - name: data
        mountPath: /data
        readOnly: false  # Separate mount; UID/group/ACL/MAC still decide file operations.
    ```

    ```text
    / (container root): ro
    /data (volume):     rw, accessible to UID10001
    /etc/app write -> rejected by root-mount policy
    /data/state write -> can succeed under the separate mount and file policy
    # A writable hostPath would require an additional host-data exposure review.
    ```

**Follow-up:** Would UID 0 inside a user namespace always be host root on that volume? No; inspect mappings, filesystem/idmap support, and actual permissions.

## 13. Does Rebuilding an Image Patch the Kubernetes Node Kernel?

**Study first:** [Patch activation](package-management.md#verify-a-running-deployment) and [shared-kernel isolation](vm-container-sec.md#namespaces-and-user-ids).

**Scenario:** A team rebuilds its application image after a Linux kernel advisory. The running node kernel is unchanged. Is a reachable node-kernel issue necessarily remediated?

??? note "Answer guidance and common trap"
    No. Containers ordinarily use the node kernel, not a replacement kernel embedded in the application image. Update/activate the node fix under the provider/distribution's workflow, verify the running kernel or supported live-patch coverage, and separately rebuild/deploy affected image userland. New images do not replace already-running replicas automatically.

    **Trap:** One artifact's update is assumed to patch all execution layers.

    **Test:** Compare the relevant vendor advisory/package revision with running node state, image digest, and actual rollout; account for draining/quorum/availability requirements.

??? example "Worked example"
    Layered deployment fixture:

    ```text
    Image digest D1 -> D2: app libraries replaced in new replicas
    Node OS packages:     kernel fix installed on disk
    Running node kernel:  old code until required activation/live patch/reboot
    # Image rebuild, package installation, and active-code replacement are separate events.
    ```

    ```bash
    uname -r  # Run at the intended authorized node observer, not an assumed image layer.
    kubectl -n study get pods -o wide
    # Match distribution backports/advisory revision; version-string age alone is insufficient.
    # No node reboot/drain or rollout mutation is performed by these inspection templates.
    ```

**Follow-up:** If a service uses an old library mapping after package upgrade, what remains? A controlled restart/activation check under the deployment's availability plan.

## 14. Does ProtectSystem=strict Prevent a Cloud Daemon from Reading Secrets?

**Study first:** [systemd confinement](process-security.md#sandboxing-a-systemd-service) and [baseline evidence](hardening-compliance.md#work-one-finding-through-to-evidence), and [log delivery limits](logging-monitoring.md#journald-forwarding-and-retention).

**Scenario:** A cloud-hosted daemon has NoNewPrivileges=yes and ProtectSystem=strict, and its sandbox score improves. It can still read a secret file permitted to its service UID. Did the read-only setting fail?

??? note "Answer guidance and common trap"
    Not necessarily. Read-only filesystem protection primarily restricts writes in the service's view, not all permitted reads. no_new_privs does not revoke current file permissions or external cloud authority. Review which secrets the daemon should read, its identity/mount/LSM policy, writable exceptions, and recovery requirements. A heuristic score is not an application data-access contract.

    **Trap:** A stronger sandbox score is treated as proof that every sensitive operation is denied.

    **Test:** The daemon can read only required configuration, write only intended state, and is denied an unrelated sensitive fixture without breaking required operations.

??? example "Worked example"
    Drop-in fragment for an existing lab service; identity/executable are configured separately:

    ```ini
    [Service]
    NoNewPrivileges=yes
    ProtectSystem=strict
    ProtectHome=yes
    StateDirectory=study-app
    # Does not automatically hide /etc/app/secret from an otherwise permitted reader.
    # StateDirectory supplies managed writable state, not a globally read-only host.
    ```

    ```bash
    systemctl cat app.service
    systemctl show app.service -p User -p ProtectSystem -p ReadWritePaths -p NoNewPrivileges
    systemd-analyze security app.service
    # Compare effective credentials and allowed/denied operations, not the score alone.
    ```

**Follow-up:** Does remote forwarding guarantee that no host event is missing? No; verify source coverage, collector receipt, queues/loss, and the trust boundary around already received records.

## Self-Assessment

A supported answer identifies the actual process/identity, object, operation, enforcement layer, retained state, and test. Change one assumption: add an ACL, retain an old session, expose a daemon socket, use a user namespace, replace a mount, or lose log delivery. Explain which conclusion changes and why.

These exercises cover the Linux section's mechanisms. Cloud IAM, application authorization, and Kubernetes networking require their own teaching and checks; this bank is not a claim of complete site-wide interview coverage.
