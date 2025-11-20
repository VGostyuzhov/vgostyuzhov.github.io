# Linux Encryption: A Study Guide

This guide covers fundamental concepts and tools for data encryption at rest on Linux systems. A strong grasp of these technologies is essential for protecting sensitive data from unauthorized access in the event of physical theft or storage media compromise.

## Basics and Core Concepts

Data encryption on Linux can be broadly categorized into two types: **full-disk encryption (FDE)** and **file-based encryption**. FDE operates at the block device level, encrypting an entire partition or disk, making it the most comprehensive solution for protecting system and user data. In contrast, file-based methods provide more granular control, encrypting individual files or directories, which can be useful for isolating specific data sets without encrypting the entire OS.

The standard for FDE in the Linux world is the **`dm-crypt`** kernel subsystem used in conjunction with **LUKS (Linux Unified Key Setup)**. `dm-crypt` handles the transparent, on-the-fly encryption and decryption of data as it's read from or written to the disk. LUKS provides the critical metadata management, storing the master encryption key (itself encrypted by a user's passphrase) and other configuration details directly on the disk. This architecture ensures that without the correct passphrase, the entire underlying block device is unreadable.

For more granular, file-level encryption, **`fscrypt`** and **`GnuPG` (GPG)** are the leading solutions. `fscrypt` is a modern kernel feature integrated directly into filesystems like `ext4` and `F2FS`. It offers high-performance, transparent encryption for designated directories and is the preferred method for home directory encryption on many modern distributions. `GnuPG`, on the other hand, is a user-space tool based on the OpenPGP standard. It is used for manually encrypting and decrypting individual files, making it ideal for secure storage of specific documents or for secure file transfers, but not for transparent, system-level encryption.

Understanding the distinction between block-level and file-level encryption is crucial. FDE with LUKS protects the entire system when it is powered off, but once the system is booted and the LUKS volume is unlocked, the data is accessible. File-based encryption like `fscrypt` or GPG can provide an additional layer of security, keeping specific files or directories encrypted even when the user is logged in. A comprehensive data protection strategy often involves a combination of both.

### Linux Encryption Cheat Sheet

| Technology | Type | Granularity | Common Use Case | Key Tool(s) |
| :--- | :--- | :--- | :--- | :--- |
| **LUKS** | Full-Disk (FDE) | Block Device / Partition | Encrypting entire OS or data drives. | `cryptsetup` |
| **fscrypt** | Filesystem-Level | Per-directory | Encrypting user home directories (`/home/user`). | `fscrypt` |
| **GnuPG (GPG)** | File-Based | Per-file | Securely storing or transferring individual files. | `gpg` |
| **eCryptfs** | Filesystem-Level | Per-file (legacy) | Older home directory encryption schemes. | `ecryptfs-utils` |

!!! info "External Resources for Deep Dive"
    *   **Arch Wiki - dm-crypt/Encrypting an entire system:** [https://wiki.archlinux.org/title/Dm-crypt/Encrypting_an_entire_system](https://wiki.archlinux.org/title/Dm-crypt/Encrypting_an_entire_system) (A comprehensive, technical guide to setting up LUKS for full-disk encryption).
    *   **`fscrypt` Documentation:** [https://github.com/google/fscrypt](https://github.com/google/fscrypt) (The official documentation and source code for the `fscrypt` tool and library).
    *   **The GNU Privacy Handbook:** [https://www.gnupg.org/gph/en/gnupg.html](https://www.gnupg.org/gph/en/gnupg.html) (A detailed manual for using GnuPG for file encryption, signing, and more).
