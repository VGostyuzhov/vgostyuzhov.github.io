# Secure Package Management

Package trust establishes which publisher/repository the system accepts, not whether every signed package is safe. A package installed with administrative authority can run privileged maintainer scripts and replace trusted components.

## Repository Trust Chains

**APT** normally verifies signed repository Release/InRelease metadata, follows authenticated hashes to package indexes, and checks downloaded package hashes against those indexes. It does not generally require every `.deb` to carry an independently verified publisher signature. An arbitrary downloaded `.deb` does not inherit a repository trust chain merely by being passed to a local installer.

**RPM/DNF** can verify package signatures and repository metadata according to configured policy and enabled checks. Do not infer one manager's protection from another's defaults. Trust keys, repository scope, expiry/freshness checks, and configuration are part of the decision.

```text
Trusted APT repository key -> signed Release/InRelease
                          -> authenticated Packages index hash
                          -> downloaded .deb hash
# HTTPS protects transport too; it does not replace this publisher trust chain.
# A compromised trusted publisher can produce a correctly signed malicious update.
```

Scope third-party keys to their repository with supported `Signed-By` configuration, verify the expected key fingerprint through a trusted distribution/vendor channel, and review repository/package precedence. Adding a broad globally trusted key can authorize more packages than intended. Do not disable signature or freshness failures simply to make installation succeed.

!!! info "External Resources"
    - [APT Repository Authentication](https://manpages.debian.org/bookworm/apt/apt-secure.8.en.html) (Debian/APT)
    - [APT Sources and Signed-By](https://manpages.debian.org/bookworm/apt/sources.list.5.en.html) (Debian/APT)
    - [RPM Signature Verification](https://rpm.org/docs/latest/man/rpmkeys.8) (RPM)

## Verify a Running Deployment

```bash
# Read-only Debian-family inventory; installed version is distribution-specific.
dpkg-query -W -f='${Package} ${Version}
' openssl
apt-cache policy openssl
uname -r  # Running kernel can differ from a newly installed kernel package.
# RPM-family alternatives:
rpm -q kernel
rpm -V openssl  # Compares against local RPM DB metadata, not an independent baseline.
```

Distribution vendors backport security fixes, so a lower-looking upstream version string alone does not prove vulnerability. Match vendor advisory/package revision and running code. Conversely, updating a library/package does not necessarily replace code already mapped by a process. Determine required service restarts, kernel reboot/live-patch coverage, and verification after deployment.

Container images are separate artifacts: patching the node does not rebuild userland libraries inside old images. Rebuild from updated trusted inputs, deploy new image digests, and verify running replicas. A digest pins content identity; it does not prove that content has no vulnerabilities.

## Patch Rollout and Recovery

Use inventory, risk-based deadlines, canaries, and a rollback/recovery plan. Security-only update selection depends on the distribution's repositories/advisory metadata; `apt` does not provide a universal `--security` upgrade switch. Configure unattended upgrades against the actual allowed origins and restart/reboot policy. Automation without monitoring can leave failures unnoticed or disrupt availability.

Do not run remote installation scripts as root solely because the transport is HTTPS. Verify provenance, content, privilege, and update path. Signed-but-unwanted software and compromised build dependencies remain supply-chain risks beyond package transport verification.
