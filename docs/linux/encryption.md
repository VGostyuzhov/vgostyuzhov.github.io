# Linux Encryption

Encrypting data at rest is a critical security control that protects sensitive information from being accessed if the physical storage media is lost or stolen. Linux provides several powerful tools for both full-disk and file-based encryption.

## Full-Disk Encryption (FDE)

Full-disk encryption encrypts an entire block device (e.g., a hard drive, SSD, or partition). This is the most common and robust method for protecting data on laptops and servers.

### `dm-crypt` and LUKS

The standard for FDE on Linux is **`dm-crypt`**, which is a transparent encryption subsystem within the Linux kernel. `dm-crypt` itself just handles the encryption; it does not store metadata about the encrypted volume.

To make `dm-crypt` practical, it is almost always used with **LUKS (Linux Unified Key Setup)**. LUKS is a standard specification for disk encryption metadata. It stores all the necessary information on the block device itself, including:
- The master encryption key (which is itself encrypted).
- Up to 8 key slots, allowing multiple passphrases or keys to decrypt the same volume.
- Cipher information (e.g., AES-256).

**How it works:**
1. At boot time, the bootloader loads an initial RAM disk (`initramfs`).
2. The `initramfs` contains the necessary tools to prompt the user for a passphrase.
3. The user enters the passphrase, which unlocks one of the LUKS key slots.
4. This decrypts the master key, which is then loaded into the kernel's `dm-crypt` subsystem.
5. `dm-crypt` creates a virtual, unencrypted block device at `/dev/mapper/`.
6. The system then mounts the filesystem from this virtual device and continues the boot process.

This setup is transparent to the user after boot. All data written to the `/dev/mapper/` device is automatically encrypted by `dm-crypt` before being written to the physical disk.

Most Linux installers provide an easy, automated way to set up LUKS during system installation.

## File-Based and Filesystem-Level Encryption

Instead of encrypting an entire block device, it is also possible to encrypt individual files or directories.

### `eCryptfs`

`eCryptfs` is a stacked cryptographic filesystem that can be mounted on top of an existing directory. It encrypts files on a file-by-file basis. It was famously used for Ubuntu's "Encrypted Home Directory" feature.

- **How it works**: When a user logs in, their `eCryptfs` passphrase (often synced with their login password) is used to mount an encrypted view of their home directory. Each file in the underlying encrypted directory corresponds to an encrypted file in the upper-level decrypted directory.
- **Advantages**: Flexible, allows for encrypting specific directories.
- **Disadvantages**: Can have performance overhead. Metadata (filenames, permissions, file sizes) in the lower directory is not fully encrypted.

### `fscrypt`

`fscrypt` is a more modern approach to filesystem-level encryption, integrated directly into filesystems like `ext4`, `F2FS`, and `UBIFS`. It offers better performance than `eCryptfs` and is now the preferred method for per-file encryption on Android and ChromeOS.

- **How it works**: `fscrypt` is enabled on a directory. Once enabled, any new files or subdirectories created within it are automatically encrypted.
- **Advantages**: High performance, strong security, and better metadata protection than `eCryptfs`.

### `GnuPG` (GPG)

GPG is a complete and open-source implementation of the OpenPGP standard. While often used for signing and encrypting emails and communications, it is also a powerful tool for encrypting individual files.

- **Symmetric Encryption**: Encrypt a file with just a passphrase.
  ```bash
  # Encrypt a file
  gpg -c sensitive-file.txt
  
  # Decrypt the file
  gpg sensitive-file.txt.gpg
  ```
- **Asymmetric Encryption**: Encrypt a file using a recipient's public key. Only the recipient, with their corresponding private key, can decrypt it.
  ```bash
  # Encrypt a file for user 'bob'
  gpg -e -r bob@example.com sensitive-file.txt
  
  # Decrypt the file (if you are Bob)
  gpg -d sensitive-file.txt.gpg
  ```

GPG is ideal for securely storing or transferring individual files, but it is not suitable for transparent, on-the-fly encryption of entire directories or disks.
