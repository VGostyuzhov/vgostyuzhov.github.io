# Network Security

## Network Stack Security

### Basics and Core Concepts

The Linux network stack security primarily relies on the **Netfilter** framework, which facilitates packet filtering, network address translation (NAT), and port translation. Administrators typically interact with Netfilter via userspace tools like `iptables` or the modern successor `nftables`. Define initiation and return-path policy for the host's role; default deny at relevant boundaries requires explicit required-flow and recovery planning. INPUT, OUTPUT, and FORWARD cover different traffic paths, so do not apply a blanket policy without understanding router/container/node requirements. Understanding the packet flow is critical; for instance, the `PREROUTING` chain handles packets before routing decisions, while `POSTROUTING` handles them after.



Network isolation is achieved via **Network Namespaces**, a fundamental building block for containerization (e.g., Docker, Kubernetes). A network namespace provides a logical copy of the network stack, including its own routes, firewall rules, and network devices, while veth links, bridges, routes, host-network mode, and configured policy determine actual reachability. Namespace separation alone does not authorize or deny every cross-workload path; see [packet-path enforcement](../networking/segmentation.md#host-input-forwarding-and-nat).

**Port Binding** and service exposure must adhere to the principle of least privilege. Services should bind strictly to the necessary interfaces; internal services should bind to the loopback interface (`127.0.0.1`) or an appropriately restricted internal interface rather than `0.0.0.0` (all interfaces). Misconfigured binding exposes internal administration panels or databases to the public internet.

**Network Monitoring and Logging** provides the necessary visibility for incident detection. This involves capturing flow data (e.g., VPC Flow Logs, NetFlow) to analyze traffic patterns and full packet capture (PCAP) for deep inspection when anomalies are detected. Logging should occur at the kernel level (e.g., using the `LOG` target in iptables) or via userspace daemons, ensuring timestamps are synchronized via NTP to allow for accurate correlation during forensic analysis.

### Cheatsheet: Network Stack Security

| Component | Key Concept | Critical Configuration/Command | Security Implication |
| :--- | :--- | :--- | :--- |
| **Firewall** | Default Deny | `iptables -P INPUT DROP` | Prevents unauthorized access by default. |
| **Netfilter** | Connection Tracking | `-m state --state ESTABLISHED,RELATED` | Allows return traffic without opening all ports. |
| **Namespaces** | Isolation | `ip netns add <name>` | Segregates network stacks to contain breaches. |
| **Binding** | Interface Restriction | `bind-address = 127.0.0.1` | Limits the listener; proxies/tunnels/port publication can still expose a path. |
| **Logging** | Kernel Logging | `iptables -A INPUT -m limit --limit 5/second -j LOG --log-prefix "INPUT: "` | Example observation rule, not a drop action; placement and a separate verdict matter. |
| **Sysctl** | IP Forwarding | `net.ipv4.ip_forward = 0` | Disables applicable IPv4 forwarding; IPv6 and container/router roles require separate policy. |

!!! info "External Resources for Deep Dive"
    * **[Arch Linux Wiki: Iptables](https://wiki.archlinux.org/title/Iptables)** - A comprehensive technical reference for configuring the Netfilter framework.
    * **[Red Hat: Introduction to Linux Network Namespaces](https://developers.redhat.com/blog/2018/05/14/introduction-to-linux-interfaces-for-virtual-networking)** - Technical deep dive into how namespaces create isolation.
    * **[NIST SP 800-41 Rev. 1: Guidelines on Firewalls and Firewalls Policy](https://csrc.nist.gov/publications/detail/sp/800-41/rev-1/final)** - Official government standards for firewall architecture and policy.

---

## Service Hardening

### Basics and Core Concepts

**Service Configuration** begins with minimizing the attack surface by disabling unused features, modules, and HTTP methods (e.g., TRACE, TRACK). Configurations should disable version broadcasting (e.g., `ServerTokens Prod` in Apache or `server_tokens off` in Nginx) to reduce incidental version disclosure. This does not replace patching or access control; software can often be fingerprinted through behavior. Review distribution defaults against the service's exposure and required features; defaults and available hardening vary by platform/version.

**Daemon Security** focuses on privilege management. Prefer dedicated service identities and the minimum privileges required by the daemon. Some services need privileged startup or narrowly scoped capabilities; constrain and drop those privileges where supported. If a service requires binding to a privileged port (< 1024), capabilities such as `CAP_NET_BIND_SERVICE` should be granted specifically to the binary or systemd unit, rather than running the entire process as root. This reduces privileges available after remote code execution; capabilities, accessible credentials, and local escalation paths still determine impact.

**Network Service Isolation** can restrict filesystem views and other process resources. The filesystem directives below do not themselves define a syscall allow-list; seccomp/SystemCallFilter-style controls require separate configuration. Techniques include `chroot` jails (changing the root directory for the process) and modern `systemd` sandboxing directives (e.g., `ProtectSystem=strict`, `ProtectHome=true`, `PrivateTmp=true`). Their effect depends on the exact directives, writable exceptions, privileges, and kernel support. A chroot alone is not a security boundary against a privileged process; use layered access controls and validate the service's actual restrictions.

**SSL/TLS Configuration** ensures the confidentiality and integrity of data in transit. Hardening involves disabling obsolete protocols (SSLv2, SSLv3, TLS 1.0, TLS 1.1) and weak cipher suites (e.g., RC4, null ciphers). Prefer ephemeral key establishment for **forward secrecy**, and review resumption/early-data modes separately and evaluate **HTTP Strict Transport Security (HSTS)** for browser HTTPS deployment. HSTS is not applicable to every TLS protocol and needs staged scope/lifetime planning; see [TLS](../networking/tls.md).



### Cheatsheet: Service Hardening

| Component | Key Concept | Critical Configuration/Example | Security Implication |
| :--- | :--- | :--- | :--- |
| **Privileges** | Non-Root Execution | `User=www-data` (Systemd) | Limits impact of RCE to low-privilege user. |
| **Capabilities** | Granular Perms | `setcap 'cap_net_bind_service=+ep' /path/to/daemon` | Binds privileged ports without root. |
| **Sandboxing** | FS Isolation | `ProtectSystem=strict` | Makes the service's filesystem view largely read-only; managed writable exceptions and other mounts still matter. |
| **Encryption** | Protocol Version | `ssl_protocols TLSv1.2 TLSv1.3;` (Nginx) | Mitigates BEAST, POODLE, and other protocol exploits. |
| **Headers** | HSTS | `Strict-Transport-Security` | Applies learned/preloaded HTTPS policy in supporting user agents. |
| **Obfuscation** | Info Disclosure | `server_tokens off` | Hides software version to complicate recon. |

!!! info "External Resources for Deep Dive"
    * **[Mozilla SSL Configuration Generator](https://ssl-config.mozilla.org/)** - The industry standard tool for generating secure web server configurations.
    * **[OWASP Cheatsheet: Transport Layer Protection](https://cheatsheetseries.owasp.org/cheatsheets/Transport_Layer_Protection_Cheat_Sheet.html)** - Definitive guide on SSL/TLS implementation standards.
    * **[Systemd Security Manual (`man systemd.exec`)](https://www.freedesktop.org/software/systemd/man/systemd.exec.html)** - Official documentation on sandboxing directives for service units.