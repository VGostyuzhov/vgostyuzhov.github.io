# Common Linux Attack Vectors

Understanding how attackers compromise Linux systems is essential for building effective defenses. This section covers common techniques used to gain initial access, escalate privileges, and establish persistence.

## Initial Access

Attackers first need to get a foothold on the system. Common entry points include:

- **Exposed Network Services**: Unpatched or misconfigured services (e.g., web servers, databases, SSH) are a primary target. Attackers scan for known vulnerabilities and exploit them to gain remote code execution.
- **Brute-Force Attacks**: Weak or default credentials on services like SSH or web applications can be easily guessed by automated brute-force tools.
- **Phishing and Social Engineering**: Tricking a user into running a malicious command or script.
- **Supply Chain Attacks**: Compromising a software package or container image that is later downloaded and run by the victim.

## Privilege Escalation

Once on the system as a low-privileged user, the attacker's next goal is to become `root`.

### Misconfigured `sudo` Rules

Overly permissive `sudo` rules can allow a user to run privileged commands. For example, if a user can run a command like `find` or `vim` with `sudo`, they can easily leverage it to spawn a root shell.

```bash
# If 'sudo find' is allowed, an attacker can do this:
sudo find . -exec /bin/sh \; -quit
```

### SUID/SGID Abuse

SUID (Set User ID) and SGID (Set Group ID) are special permissions that allow an executable to run with the permissions of the file owner or group, respectively. If a vulnerable program has the SUID bit set and is owned by `root`, an attacker may be able to exploit it to gain root privileges.

```bash
# Find all SUID executables owned by root
find / -user root -perm -4000 -print 2>/dev/null
```

### Kernel Exploits

A vulnerability in the Linux kernel itself can be exploited to escalate privileges from a local user to `root`. These are among the most severe vulnerabilities. Keeping the kernel patched is the primary defense. The `uname -r` command shows the current kernel version.

### Leaked Credentials and Information

- **Readable `/etc/shadow`**: Although rare, if an attacker can read the shadow file, they can attempt to crack the password hashes offline.
- **Unprotected Backups or Scripts**: Plaintext passwords or sensitive information stored in world-readable backup files or scripts are a common finding.

### Container Escapes

If an attacker gains access to a container, they may try to "escape" to the underlying host system. This is often possible if:
- The container is run in **privileged mode** (`--privileged`).
- The Docker socket (`/var/run/docker.sock`) is mounted inside the container.
- The host's root filesystem is mounted as a volume.
- A kernel exploit is used to break out of the container's isolation.

## Persistence Mechanisms

After gaining root access, attackers want to ensure they can regain access to the system even if it is rebooted or their initial exploit is patched.

### Backdoors and Web Shells

- **Web Shell**: A malicious script uploaded to a web server that allows the attacker to execute arbitrary commands.
- **SSH Keys**: Adding the attacker's public key to `~/.ssh/authorized_keys` (especially for the `root` user) provides stealthy and persistent access.

### `systemd` or `init.d` Services

Creating a new `systemd` service or an `init.d` script is a reliable way to run a malicious process at boot time.

```
# A simple malicious systemd service file
[Unit]
Description=System-critical update service

[Service]
ExecStart=/bin/nc -lp 4444 -e /bin/sh
Restart=always

[Install]
WantedBy=multi-user.target
```

### Cron Job Manipulation

Attackers can add a new cron job or modify an existing one to execute a malicious script or command on a recurring schedule. Cron jobs can be system-wide (`/etc/crontab`, `/etc/cron.d/`) or user-specific (`crontab -e`).

### Library Injection (`LD_PRELOAD`)

The `LD_PRELOAD` environment variable can be used to force a process to load a malicious shared library. If an attacker can control the environment of a privileged process, they can use `LD_PRELOAD` to inject their code and gain control. This is often used to create a **rootkit**.

### Kernel Module Rootkits

The most advanced and stealthiest form of persistence is a **Loadable Kernel Module (LKM) rootkit**. This involves inserting malicious code directly into the kernel. An LKM rootkit can:
- Hide processes, files, and network connections.
- Intercept system calls to manipulate data.
- Create a hidden backdoor.

Detecting LKM rootkits is very difficult without specialized tools like `rkhunter` or `chkrootkit`.