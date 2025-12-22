# DNS: A Study Guide

This guide covers the fundamentals of the Domain Name System (DNS), a critical component of the internet that translates human-readable domain names into machine-readable IP addresses. Understanding DNS is essential for network security, as it is often a target for and a medium of malicious activity.

## Basics and Core Concepts

DNS operates primarily on UDP port 53, providing fast, connectionless lookups. When a client requests a domain name that is too large for a single UDP packet, the DNS server can respond with a truncation flag, signaling the client to retry the request over the more reliable TCP protocol. The lookup process is recursive: a client asks its local resolver, which then queries a root server, a Top-Level Domain (TLD) server (e.g., for `.com`), and finally the authoritative name server for the specific domain to get the IP address. This result is then cached at various levels to speed up subsequent requests.

From a security perspective, DNS traffic can be a blind spot. While HTTP/S traffic logs show the full URL being accessed, standard DNS logs only show the domain name being queried.

### Data Exfiltration

Attackers exploit this by using DNS for **data exfiltration**. They can encode stolen data into a series of subdomains (e.g., `[encoded_data].attacker.com`) and send it out as DNS queries. These queries traverse the network and are resolved by the attacker's malicious name server, which logs the subdomains and reconstructs the data. This technique is effective because it bypasses many traditional network firewalls and web proxies.

### DNS Sinkhole

Another key security concept is the **DNS sinkhole**. This is a technique used by defenders to redirect traffic intended for malicious domains to a controlled server. By poisoning the DNS entry for a known command-and-control (C2) server, all infected hosts within a network that try to contact the C2 server will instead be sent to the sinkhole. This not only prevents the malicious traffic from reaching its destination but also provides a valuable source of threat intelligence by logging which internal hosts are compromised.

### DNS Record Types Cheat Sheet

| Record Type | Name | Purpose | Security Relevance |
| :--- | :--- | :--- | :--- |
| **A / AAAA** | Address Record | Maps a domain to an IPv4 (A) or IPv6 (AAAA) address. | The most common record type; used to direct traffic. |
| **CNAME** | Canonical Name | An alias that points one domain to another. | Can be used to obscure the true destination of traffic. |
| **MX** | Mail Exchanger | Specifies the mail server responsible for a domain. | Can be targeted to intercept or spoof emails. |
| **NS** | Name Server | Delegates a domain or subdomain to a set of authoritative name servers. | Hijacking NS records can give an attacker control over a domain. |
| **PTR** | Pointer Record | Used for reverse DNS lookups, mapping an IP address back to a domain name. | Often used in security logging to identify the source of traffic. |
| **SOA** | Start of Authority | Contains administrative information about the domain (e.g., primary name server, admin email). | Provides reconnaissance information to attackers. |
| **TXT** | Text Record | Holds arbitrary text; used for SPF, DKIM, and DMARC email authentication. | Misconfigured email authentication records can lead to spoofing. |

!!! info "External Resources for Deep Dive"
    *   **Cloudflare - What is DNS?**: [https://www.cloudflare.com/learning/dns/what-is-dns/](https://www.cloudflare.com/learning/dns/what-is-dns/) (A clear and comprehensive overview of the DNS process).
    *   **"DNS Exfiltration: Tunneling data over DNS" by Acunetix**: [https://www.acunetix.com/blog/web-security-zone/dns-exfiltration/](https://www.acunetix.com/blog/web-security-zone/dns-exfiltration/) (A technical article explaining how DNS exfiltration works).
    *   **SANS Institute - "What is a DNS Sinkhole?"**: [https://www.sans.org/blog/what-is-a-dns-sinkhole/](https://www.sans.org/blog/what-is-a-dns-sinkhole/) (An explanation of DNS sinkholing as a defensive technique).