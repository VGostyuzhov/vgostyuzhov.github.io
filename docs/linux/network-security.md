# Network Security

## Network Stack Security

### Basics and Core Concepts

The Linux network stack security primarily relies on the **Netfilter** framework, which facilitates packet filtering, network address translation (NAT), and port translation. Administrators typically interact with Netfilter via userspace tools like `iptables` or the modern successor `nftables`. Effective firewall configuration requires a "default deny" policy (dropping all traffic by default) and explicitly allowing only necessary traffic through specific chains (INPUT, OUTPUT, FORWARD). Understanding the packet flow is critical; for instance, the `PREROUTING` chain handles packets before routing decisions, while `POSTROUTING` handles them after.



Network isolation is achieved via **Network Namespaces**, a fundamental building block for containerization (e.g., Docker, Kubernetes). A network namespace provides a logical copy of the network stack, including its own routes, firewall rules, and network devices, ensuring that processes within the namespace cannot interact with the host's network stack or other namespaces unless explicitly bridged or routed. This limits the blast radius of a compromised application by preventing lateral movement at the network layer.

**Port Binding** and service exposure must adhere to the principle of least privilege. Services should bind strictly to the necessary interfaces; internal services should bind to the loopback interface (`127.0.0.1`) or a private, non-routed internal interface rather than `0.0.0.0` (all interfaces). Misconfigured binding exposes internal administration panels or databases to the public internet.

**Network Monitoring and Logging** provides the necessary visibility for incident detection. This involves capturing flow data (e.g., VPC Flow Logs, NetFlow) to analyze traffic patterns and full packet capture (PCAP) for deep inspection when anomalies are detected. Logging should occur at the kernel level (e.g., using the `LOG` target in iptables) or via userspace daemons, ensuring timestamps are synchronized via NTP to allow for accurate correlation during forensic analysis.

### Cheatsheet: Network Stack Security

| Component | Key Concept | Critical Configuration/Command | Security Implication |
| :--- | :--- | :--- | :--- |
| **Firewall** | Default Deny | `iptables -P INPUT DROP` | Prevents unauthorized access by default. |
| **Netfilter** | Connection Tracking | `-m state --state ESTABLISHED,RELATED` | Allows return traffic without opening all ports. |
| **Namespaces** | Isolation | `ip netns add <name>` | Segregates network stacks to contain breaches. |
| **Binding** | Interface Restriction | `bind-address = 127.0.0.1` | Prevents public exposure of internal services. |
| **Logging** | Kernel Logging | `iptables -j LOG --log-prefix "DROP: "` | Provides audit trails for dropped packets. |
| **Sysctl** | IP Forwarding | `net.ipv4.ip_forward = 0` | Prevents the host from acting as a router. |

!!! info "External Resources for Deep Dive"
    * **[Arch Linux Wiki: Iptables](https://wiki.archlinux.org/title/Iptables)** - A comprehensive technical reference for configuring the Netfilter framework.
    * **[Red Hat: Introduction to Linux Network Namespaces](https://developers.redhat.com/blog/2018/05/14/introduction-to-linux-interfaces-for-virtual-networking)** - Technical deep dive into how namespaces create isolation.
    * **[NIST SP 800-41 Rev. 1: Guidelines on Firewalls and Firewalls Policy](https://csrc.nist.gov/publications/detail/sp/800-41/rev-1/final)** - Official government standards for firewall architecture and policy.

---

## Service Hardening

### Basics and Core Concepts

**Service Configuration** begins with minimizing the attack surface by disabling unused features, modules, and HTTP methods (e.g., TRACE, TRACK). Configurations should disable version broadcasting (e.g., `ServerTokens Prod` in Apache or `server_tokens off` in Nginx) to prevent attackers from easily mapping vulnerabilities to specific software versions. Default configurations provided by package managers are rarely secure; they prioritize compatibility over security and must be rigorously audited against benchmarks (e.g., CIS Benchmarks).

**Daemon Security** focuses on privilege management. Services should never run as the root user; instead, they should utilize dedicated service accounts with unique UIDs/GIDs. If a service requires binding to a privileged port (< 1024), capabilities such as `CAP_NET_BIND_SERVICE` should be granted specifically to the binary or systemd unit, rather than running the entire process as root. This prevents a remote code execution (RCE) vulnerability from immediately yielding root access to the host system.

**Network Service Isolation** restricts the file system and system call access of a running process. Techniques include `chroot` jails (changing the root directory for the process) and modern `systemd` sandboxing directives (e.g., `ProtectSystem=strict`, `ProtectHome=true`, `PrivateTmp=true`). These controls ensure that even if a daemon is compromised, the attacker's ability to persist, modify system files, or access user data is severely restricted.

**SSL/TLS Configuration** ensures the confidentiality and integrity of data in transit. Hardening involves disabling obsolete protocols (SSLv2, SSLv3, TLS 1.0, TLS 1.1) and weak cipher suites (e.g., RC4, null ciphers). Implementations must enforce **Perfect Forward Secrecy (PFS)** using ephemeral key exchanges (ECDHE) and enable **HTTP Strict Transport Security (HSTS)** to prevent protocol downgrade attacks.



### Cheatsheet: Service Hardening

| Component | Key Concept | Critical Configuration/Example | Security Implication |
| :--- | :--- | :--- | :--- |
| **Privileges** | Non-Root Execution | `User=www-data` (Systemd) | Limits impact of RCE to low-privilege user. |
| **Capabilities** | Granular Perms | `setcap 'cap_net_bind_service=+ep'` | Binds privileged ports without root. |
| **Sandboxing** | FS Isolation | `ProtectSystem=strict` | Makes the OS filesystem read-only for the service. |
| **Encryption** | Protocol Version | `FilesMatch "TLSv1.2|TLSv1.3"` | Mitigates BEAST, POODLE, and other protocol exploits. |
| **Headers** | HSTS | `Strict-Transport-Security` | Forces clients to connect over HTTPS only. |
| **Obfuscation** | Info Disclosure | `server_tokens off` | Hides software version to complicate recon. |

!!! info "External Resources for Deep Dive"
    * **[Mozilla SSL Configuration Generator](https://ssl-config.mozilla.org/)** - The industry standard tool for generating secure web server configurations.
    * **[OWASP Cheatsheet: Transport Layer Protection](https://cheatsheetseries.owasp.org/cheatsheets/Transport_Layer_Protection_Cheat_Sheet.html)** - Definitive guide on SSL/TLS implementation standards.
    * **[Systemd Security Manual (`man systemd.exec`)](https://www.freedesktop.org/software/systemd/man/systemd.exec.html)** - Official documentation on sandboxing directives for service units.