# Linux Encryption at Rest

Encryption protects selected data under a key-management and access model. Distinguish a stolen offline disk from a running host whose volume is already unlocked. An authorized or compromised process that can read plaintext is not stopped by the underlying block encryption.

## Block and File Encryption

| Mechanism | Boundary | Important limitation |
|---|---|---|
| dm-crypt + LUKS | Block-device mapping and keyslot metadata | Open mapping exposes decrypted blocks/files to authorized running processes |
| fscrypt | Filesystem-supported encrypted directory policies | Key availability, metadata coverage, and access checks differ from block encryption |
| GnuPG | Explicit encrypted file/message | Decrypted copies, recipient keys, and running-process exposure remain relevant |

LUKS keyslots protect access to a volume key using supported credentials and key-derivation function (KDF) parameters. A passphrase change normally changes how that key is unlocked, not every encrypted data block. The header contains critical metadata; losing it can make data unrecoverable even with a remembered passphrase. Backups must protect both availability and confidentiality.

```bash
# Read-only Linux templates for a known authorized LUKS device/mapping.
cryptsetup luksDump /dev/example-encrypted-device
cryptsetup status example-mapping
# Do not print passphrases, volume keys, or recovery material into notes/logs.
# luksFormat and erase are destructive operations, not inspection commands.
```

A header backup can retain an older keyslot capable of unlocking the same volume key. Removing a passphrase from the current header does not retroactively invalidate every historical header backup. Design credential revocation, backup retention, and any required volume-key reencryption together. Test restoration on disposable data, not the only production copy.

!!! info "External Resources"
    - [cryptsetup Key Management and LUKS Metadata](https://man7.org/linux/man-pages/man8/cryptsetup.8.html) (cryptsetup)
    - [LUKS Header Backup Security](https://man7.org/linux/man-pages/man8/cryptsetup-luksHeaderBackup.8.html) (cryptsetup)
    - [fscrypt Threat Model and Key Availability](https://docs.kernel.org/filesystems/fscrypt.html) (Linux kernel)

## Running Hosts, Cloud Volumes, and Secrets

```text
Stolen encrypted volume + no usable key -> offline contents can remain protected
Running host + open mapping + process permitted to read -> plaintext available
Cloud volume encryption + overly privileged workload IAM -> service reads can still succeed
# Storage encryption, OS permissions, workload identity, and key-policy scope differ.
```

Unlocked fscrypt directories are likewise subject to the key and filesystem threat model; encryption is not automatic isolation from every process under the same identity. Protect keys from process memory/core dumps, uncontrolled backups, and overly broad access. Cloud-managed encryption adds KMS/IAM/service trust rather than eliminating key management.

Use independent application/file encryption where the data-owner/recipient model requires it, but account for where plaintext is processed. Memory-resident secrets and already-copied plaintext cannot be revoked by changing only a disk passphrase.
