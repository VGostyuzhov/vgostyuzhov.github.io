# Network Protocols

## Transport Layer Protocols

**TCP (Transmission Control Protocol):**

- Connection-oriented, reliable, ordered delivery
- Three-way handshake (SYN, SYN-ACK, ACK) establishes session
- Congestion control (slow start, congestion avoidance) throttles on packet loss
- Used by: HTTP/S, SSH, SMTP, FTP, database protocols
- See [Core Networking - TCP vs UDP](core-networking.md#tcp-vs-udp) for detailed comparison

**UDP (User Datagram Protocol):**

- Connectionless, best-effort delivery, no ordering guarantees
- Minimal 8-byte header: source port, destination port, length, checksum
- Used by: DNS (queries), DHCP, VoIP, streaming, gaming, traceroute, QUIC

**SCTP (Stream Control Transmission Protocol):**

- Multi-homed (multiple IP addresses per endpoint) and multi-streamed
- Message-oriented (preserves message boundaries unlike TCP's byte stream)
- Used in telecom signaling (SS7 over IP, Diameter)

**QUIC:**

- UDP-based transport developed by Google, used by HTTP/3
- Built-in TLS 1.3, reduces handshake latency (0-RTT resumption)
- Multiplexed streams without head-of-line blocking

!!! info "External Resources"
    - [RFC 793 - TCP](https://datatracker.ietf.org/doc/html/rfc793) (IETF)
    - [RFC 768 - UDP](https://datatracker.ietf.org/doc/html/rfc768) (IETF)
    - [RFC 9000 - QUIC](https://datatracker.ietf.org/doc/html/rfc9000) (IETF)

## Application Layer Protocols

Overview of major application-layer protocols. Each subsection below covers a protocol family in detail.

| Protocol | Port(s) | Transport | Primary Use |
|----------|---------|-----------|------------|
| HTTP/HTTPS | 80, 443 | TCP (QUIC for HTTP/3) | Web traffic |
| DNS | 53 | UDP (TCP for zone transfers) | Name resolution |
| SMTP | 25, 587, 465 | TCP | Email sending |
| IMAP | 143, 993 | TCP | Email retrieval |
| POP3 | 110, 995 | TCP | Email retrieval |
| FTP/SFTP | 21, 22 | TCP | File transfer |
| SSH | 22 | TCP | Remote access, tunneling |
| RDP | 3389 | TCP/UDP | Windows remote desktop |
| LDAP | 389, 636 | TCP/UDP | Directory services |
| SNMP | 161, 162 | UDP | Network management |
| NTP | 123 | UDP | Time synchronization |
| Syslog | 514 | UDP (TCP with TLS) | Log forwarding |

For HTTP/HTTPS details, see [Core Networking - HTTP/HTTPS](core-networking.md#httphttps). For DNS, see the dedicated [DNS page](dns.md).

!!! info "External Resources"
    - [Application Layer - Cloudflare](https://www.cloudflare.com/learning/ddos/glossary/open-systems-interconnection-model-osi/) (Cloudflare)
    - [IANA Protocol Numbers](https://www.iana.org/protocols) (IANA)
    - [RFC Index](https://www.rfc-editor.org/rfc-index.html) (IETF)

## Email Protocols (SMTP, IMAP, POP3)

**SMTP (Simple Mail Transfer Protocol):**

- **Port 25** - server-to-server relay (often blocked by ISPs to prevent spam)
- **Port 587** - client submission with STARTTLS (recommended)
- **Port 465** - implicit TLS (SMTPS, re-standardised in RFC 8314)
- Push-based: client sends to server, server relays to destination MX
- Security: SPF, DKIM, and DMARC records in DNS validate sender legitimacy

**IMAP (Internet Message Access Protocol):**

- **Port 143** (cleartext/STARTTLS), **Port 993** (implicit TLS)
- Messages remain on server; client syncs a local view
- Supports folders, search, flags, partial message fetch
- Preferred for multi-device access

**POP3 (Post Office Protocol v3):**

- **Port 110** (cleartext/STARTTLS), **Port 995** (implicit TLS)
- Downloads messages and typically deletes from server
- Simpler than IMAP; single-device use case

**Email security stack:**

| Mechanism | DNS Record | Purpose |
|-----------|-----------|---------|
| **SPF** | TXT | Lists authorised sending IPs for a domain |
| **DKIM** | TXT | Cryptographic signature on email headers/body |
| **DMARC** | TXT | Policy for handling SPF/DKIM failures; reporting |

!!! info "External Resources"
    - [RFC 5321 - SMTP](https://datatracker.ietf.org/doc/html/rfc5321) (IETF)
    - [RFC 8314 - Email Submission with TLS](https://datatracker.ietf.org/doc/html/rfc8314) (IETF)
    - [DMARC Overview](https://dmarc.org/overview/) (DMARC.org)

## File Transfer Protocols (FTP, SFTP, SCP)

**FTP (File Transfer Protocol):**

- **Port 21** (control), **Port 20** (data in active mode)
- Cleartext credentials and data - never use without TLS (FTPS)
- Active mode: server connects back to client (firewall-unfriendly)
- Passive mode: client initiates both connections (firewall-friendly)
- FTPS (FTP over TLS) adds encryption but retains FTP's port complexity

**SFTP (SSH File Transfer Protocol):**

- Runs over **SSH (port 22)** - single encrypted channel
- Not related to FTP despite the name; entirely different protocol
- Supports resume, directory listing, permissions
- Preferred over FTP/FTPS for secure file transfer

**SCP (Secure Copy Protocol):**

- Also runs over **SSH (port 22)**
- Simple file copy (no directory listing or resume)
- Being deprecated in favour of SFTP in modern OpenSSH

**RPC (Remote Procedure Call):**

- Predefined set of tasks that remote clients can execute
- Used internally within organisations for service-to-service communication
- Implementations: ONC RPC, gRPC (HTTP/2-based, Protocol Buffers), XML-RPC, JSON-RPC

!!! info "External Resources"
    - [SFTP vs FTPS - SSH.com](https://www.ssh.com/academy/ssh/sftp) (SSH.com)
    - [gRPC Documentation](https://grpc.io/docs/) (gRPC)
    - [RFC 959 - FTP](https://datatracker.ietf.org/doc/html/rfc959) (IETF)

## Remote Access Protocols (SSH, Telnet, RDP)

**SSH (Secure Shell) - Port 22:**

- Encrypted remote shell access, tunneling, and file transfer
- Handshake: asymmetric encryption negotiates a shared symmetric session key
- Authentication: public key (preferred), password, certificate-based, keyboard-interactive
- Port forwarding: local (`-L`), remote (`-R`), dynamic SOCKS proxy (`-D`)
- Hardening: disable root login, disable password auth, use `AllowUsers`/`AllowGroups`, change default port, use `fail2ban`

**Telnet - Port 23 (992 for TLS):**

- Cleartext remote terminal access
- All data including credentials transmitted unencrypted
- Never use in production; exists only on legacy systems
- Replaced entirely by SSH

**RDP (Remote Desktop Protocol) - Port 3389:**

- Microsoft proprietary protocol for graphical remote desktop
- Supports encryption (TLS), NLA (Network Level Authentication)
- Common brute-force target; mitigate with MFA, VPN gateway, account lockout
- BlueKeep (CVE-2019-0708) demonstrated critical RDP vulnerabilities

!!! info "External Resources"
    - [OpenSSH Manual](https://www.openssh.com/manual.html) (OpenSSH)
    - [SSH Hardening Guide - Mozilla](https://infosec.mozilla.org/guidelines/openssh) (Mozilla)
    - [RDP Security Best Practices - CISA](https://www.cisa.gov/news-events/cybersecurity-advisories/aa19-168a) (CISA)

## Directory & Authentication Protocols (LDAP, Kerberos, RADIUS)

**LDAP (Lightweight Directory Access Protocol):**

- **Port 389** (cleartext/STARTTLS), **Port 636** (LDAPS)
- Hierarchical directory for storing identity data (users, groups, OUs)
- Distinguished Names (DN): `cn=john,ou=users,dc=example,dc=com`
- Operations: bind (authenticate), search, add, modify, delete
- Security: always use LDAPS or STARTTLS; LDAP injection is a real threat

**Kerberos:**

- **Port 88** (TCP/UDP)
- Ticket-based authentication protocol; default for Active Directory
- Components: KDC (Key Distribution Center), TGT (Ticket Granting Ticket), service tickets
- Flow: client authenticates to KDC, gets TGT, uses TGT to request service tickets
- Attacks: pass-the-ticket, golden ticket (forged TGT), Kerberoasting (cracking service ticket hashes)
- For detailed coverage, see [Directory Services](../authentication/directory-services.md)

**RADIUS (Remote Authentication Dial-In User Service):**

- **Ports 1812** (auth), **1813** (accounting)
- Centralised AAA (Authentication, Authorization, Accounting) for network access
- Used by VPN concentrators, Wi-Fi controllers, network switches (802.1X)
- Client-server model: NAS (Network Access Server) proxies auth to RADIUS server
- Password encrypted with shared secret + MD5 (weak); use RADSEC (RADIUS over TLS) where possible

!!! info "External Resources"
    - [LDAP Overview - Microsoft](https://learn.microsoft.com/en-us/previous-versions/windows/desktop/ldap/lightweight-directory-access-protocol-ldap-api) (Microsoft)
    - [Kerberos Protocol - MIT](https://web.mit.edu/kerberos/) (MIT)
    - [RFC 2865 - RADIUS](https://datatracker.ietf.org/doc/html/rfc2865) (IETF)

## SSL/TLS

SSL (deprecated) and TLS provide confidentiality, integrity, and authentication for network communication. TLS 1.3 is the current standard.

**Version history:**

| Version | Status | Key Changes |
|---------|--------|-------------|
| SSL 2.0/3.0 | Deprecated, insecure | Known vulnerabilities (POODLE) |
| TLS 1.0/1.1 | Deprecated | BEAST, CRIME attacks; removed by modern browsers |
| TLS 1.2 | Supported | Configurable cipher suites; widely deployed |
| TLS 1.3 | Current standard | Removed weak ciphers, 1-RTT handshake, mandatory PFS |

**TLS 1.3 handshake (simplified):**

1. **ClientHello** - supported cipher suites, key shares, SNI
2. **ServerHello** - selected cipher suite, key share, certificate
3. **Finished** - both sides derive session keys; encrypted communication begins

TLS 1.3 completes in 1 round-trip (down from 2 in TLS 1.2). Supports 0-RTT resumption (with replay risk).

**Certificate validation chain:**

- Server presents certificate signed by Certificate Authority (CA)
- Client verifies signature chain up to a trusted root CA in its root store
- Checks: expiry, revocation (OCSP/CRL), hostname match, key usage

**Known TLS attacks:**

| Attack | Target | Mitigation |
|--------|--------|-----------|
| POODLE | SSL 3.0 CBC padding | Disable SSL 3.0 |
| BEAST | TLS 1.0 CBC IV | Upgrade to TLS 1.2+ |
| CRIME/BREACH | TLS compression / HTTP compression | Disable TLS compression; mitigate BREACH at app level |
| HEARTBLEED | OpenSSL heartbeat extension | Patch OpenSSL; revoke and reissue certificates |
| FREAK/Logjam | Export-grade and weak DH ciphers | Disable weak cipher suites |
| Downgrade | Forces older protocol version | TLS_FALLBACK_SCSV, use TLS 1.3 |

**Best practices:**

- Use TLS 1.3; allow TLS 1.2 as fallback with strong cipher suites only
- Enable HSTS with `includeSubDomains` and `preload`
- Use Certificate Transparency (CT) logs to detect mis-issuance
- Automate certificate management (Let's Encrypt, ACME protocol)
- Configure OCSP stapling to reduce revocation check latency

!!! info "External Resources"
    - [SSL/TLS Best Practices - Mozilla](https://wiki.mozilla.org/Security/Server_Side_TLS) (Mozilla)
    - [Dutch NCSC - TLS Guidelines](https://english.ncsc.nl/publications/publications/2021/january/19/it-security-guidelines-for-transport-layer-security-2.1) (NCSC-NL)
    - [Qualys SSL Labs - SSL Server Test](https://www.ssllabs.com/ssltest/) (Qualys)
