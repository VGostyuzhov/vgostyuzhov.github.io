# Access Control Systems

---

# Discretionary Access Control (DAC)

## Basics and Core Concepts

Discretionary Access Control (DAC) is an access control model where the **owner** of an object (e.g., a file or directory) has the discretion to determine the access permissions for other subjects (e.g., users or groups). This model is identity-centric, meaning access rights are bound to the identity of the subject. The creator of an object is its initial owner, and this owner can grant or revoke privileges to others.

Traditional Unix permissions are a classic implementation of DAC. Every file and directory has three sets of permissions defined for three classes of users: the **owner** (u), the **group** (g), and **others** (o). Each set defines three permissions: **read (r)**, **write (w)**, and **execute (x)**. These are commonly represented by a 9-character string (e.g., `rwxr-xr--`) or an octal number (e.g., `754`). The `chmod` command modifies these permissions, while `chown` and `chgrp` change the user and group ownership.

Special permission bits modify the standard behavior.

* The **SUID (Set User ID)** bit (`s` in the owner's execute field) on an executable file causes it to run with the privileges of the file's owner, not the user who executed it (e.g., the `passwd` command running as root). 
* The **SGID (Set Group ID)** bit (`s` in the group's execute field) on an executable makes it run with the file's group privileges. On a directory, SGID causes new files and subdirectories created within it to inherit the directory's group ownership, which is useful for shared project folders. 
* The **Sticky Bit** (`t` in the others' execute field), when set on a directory (like `/tmp`), allows users to delete or rename files *only* if they are the owner of the file, the owner of the directory, or root, even if they have write permission to the directory.

**Access Control Lists (ACLs)** extend the traditional Unix permission model. While standard permissions only define access for one owner, one group, and everyone else, ACLs provide a more granular mechanism. They allow the owner to grant or deny permissions for specific, additional users and groups. The `getfacl` command is used to view ACLs, and `setfacl` is used to modify them. A file with an ACL applied is often indicated by a `+` (plus sign) at the end of its permission string in the `ls -l` output.

| Component | Symbolic | Octal | Description |
| :--- | :--- | :--- | :--- |
| **Permissions** | | | |
| Read | `r` | `4` | View contents of a file; list contents of a directory. |
| Write | `w` | `2` | Modify or delete a file; create or delete files in a directory. |
| Execute | `x` | `1` | Run a file as a script/program; `cd` into a directory. |
| No Permission | `-` | `0` | No access. |
| **Special Bits** | | | |
| SUID | `---s--` | `4000` | (Set User ID) Executable runs with file owner's privileges. |
| SGID | `------s-` | `2000` | (Set Group ID) Executable runs with file group's privileges. / Files in dir inherit dir's group. |
| Sticky Bit | `--------t` | `1000` | On directories, only the owner (or root) can delete/rename files. |
| **Core Commands** | | | |
| `chmod` | `chmod [u/g/o/a][+/-][r/w/x]` | `chmod [octal]` | Change permissions. (e.g., `chmod 755 file`, `chmod g+w file`) |
| `chown` | `chown [user]:[group]` | | Change owner and/or group. (e.g., `chown user:group file`) |
| `chgrp` | `chgrp [group]` | | Change group ownership. |
| **ACL Commands** | | | |
| `getfacl` | `getfacl [file]` | | View the Access Control List for a file. |
| `setfacl` | `setfacl -m u:[user]:[rwx]` | | Modify (set) the ACL for a file. (e.Same syntax for `g:[group]`) |

!!! info "External Resources"
    - **Official Documentation/Guides:**
        * [Access Control Lists in Linux](https://documentation.suse.com/sles/12-SP5/html/SLES-all/cha-security-acls.html) (SUSE Documentation)
        * [What You Need To Know To Set Linux Permissions and Access Control Lists](https://www.comptia.org/en-us/blog/what-you-need-to-know-to-set-linux-permissions-and-access-control-lists) (CompTIA)
    - **In-Depth Blog Posts:**
        * [Linux File Permissions: Understanding Setuid, Setgid, and the Sticky Bit](https://www.cbtnuggets.com/blog/technology/system-admin/linux-file-permissions-understanding-setuid-setgid-and-the-sticky-bit) (CBT Nuggets)
    - **Video Tutorials:**
        * [YouTube - Linux ACLs Explained](https://www.youtube.com/watch?v=LnKoncbQBsM)

---

# Mandatory Access Control (MAC)

## Basics and Core Concepts

Mandatory Access Control (MAC) is an access control model where the operating system constrains the ability of a subject (e.g., a process) to access or perform operations on an object (e.g., a file). Unlike Discretionary Access Control (DAC), where owners can set permissions, MAC policies are centrally managed by an administrator and enforced by the kernel. Access decisions are made by a **Reference Monitor** (like the Linux Security Modules - LSM) by comparing security labels of the subject and object against a system-wide policy.

SELinux (Security-Enhanced Linux) is a label-based MAC implementation. Every subject and object on the system is assigned a **security context** (e.g., `user:role:type:level`). The core of SELinux policy is **Type Enforcement (TE)**, which defines allowed interactions between types (e.g., `allow httpd_t var_www_t:file { read }`). Its policy is monolithic, loaded into the kernel, and can operate in `enforcing`, `permissive`, or `disabled` modes. AppArmor, by contrast, is a path-based MAC system. It uses **profiles** (plain text files) that are bound to specific executable paths (e.g., `/usr/sbin/sshd`) and define what file system paths and capabilities that application can access.

The SELinux context often includes components for **Role-Based Access Control (RBAC)** and **Multi-Level/Category Security (MLS/MCS)**. RBAC limits user privileges based on their assigned role (e.g., `web_admin_r`). MLS provides a hierarchical security model (e.g., `Unclassified` < `Secret`) based on the Bell-LaPadula model, enforcing "no read up" and "no write down" rules. MCS is a non-hierarchical extension that uses categories (e.g., `s0:c1,c2`) to provide compartmentalization, which is widely used to isolate containers.

Policy development and debugging are critical skills. For SELinux, policy is written in TE or CIL (Common Intermediate Language). Debugging relies on parsing `type=AVC` (Access Vector Cache) denial messages from `/var/log/audit/audit.log` using `ausearch`. The `audit2allow` utility can read these denials and suggest new `allow` rules. For AppArmor, profiles are developed manually or with tools like `aa-genprof`. Debugging is done in `complain` mode, with denials logged to `kern.log` or `auditd`, and the `aa-logprof` tool is used to interactively review and approve or deny behaviors.

# #Cheatsheet: MAC Concepts & Implementations

| Concept | SELinux (Label-Based) | AppArmor (Path-Based) |
| :--- | :--- | :--- |
| **Core Unit** | Security Context (`user:role:type:level`) | Profile (per-executable path) |
| **Policy Logic** | Type Enforcement (TE). Rules on *types*. | Path-based rules. `allow /var/www/ r`. |
| **Enforcement** | Kernel (LSM). Enforces on all objects. | Kernel (LSM). Enforces on profiled apps. |
| **Modes** | `enforcing`, `permissive`, `disabled` | `enforce`, `complain`, `disabled` |
| **Denial Log** | `/var/log/audit/audit.log` (`type=AVC`) | `/var/log/kern.log` or `auditd` |
| **Debug Tool** | `ausearch`, `audit2allow`, `sealert` | `aa-logprof`, `aa-complain` |
| **Models** | Supports TE, RBAC, MLS, MCS | Focuses on application confinement |

## External Resources for Deep Dive

!!! info "External Resources for Deep Dive"
    * **Official Documentation & Wikis:**
        * [SELinux Project Wiki](https://selinuxproject.org/page/Main_Page): The central hub for SELinux documentation, architecture explanations, and news.
        * [AppArmor Wiki](https://gitlab.com/apparmor/apparmor/-/wikis/home): The official GitLab wiki for AppArmor, containing documentation on profile creation, kernel interfaces, and utilities.
        * [Red Hat Enterprise Linux (RHEL) SELinux Guide](https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/8/html/using_selinux): A comprehensive and practical guide covering SELinux policy, modes, and troubleshooting.

    * **In-Depth Articles & Tutorials:**
        * [Red Hat Blog: MCS and Containers](https://www.redhat.com/en/blog/multi-category-security-mcs-selinux-and-containers): A detailed technical blog post explaining the practical application of MCS with SELinux for container isolation.
        * [Ubuntu Tutorial: AppArmor Profile Development](https://ubuntu.com/tutorials/apparmor-profile-development): A hands-on tutorial for creating and debugging AppArmor profiles from scratch.