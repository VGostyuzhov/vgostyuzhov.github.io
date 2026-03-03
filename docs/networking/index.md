# Core Networking

## OSI Model & TCP/IP Stack

The OSI model provides a layered abstraction for network communication. The TCP/IP model is the practical implementation used on the internet.

| OSI Layer | Name | Function | TCP/IP Equivalent | Key Protocols |
|-----------|------|----------|-------------------|---------------|
| 7 | Application | User-facing services, API interaction | Application | HTTP, DNS, SMTP, SSH, FTP |
| 6 | Presentation | Data formatting, encryption, compression | Application | SSL/TLS, MIME, JPEG |
| 5 | Session | Session establishment and management | Application | NetBIOS, RPC, SOCKS |
| 4 | Transport | End-to-end delivery, flow control | Transport | TCP, UDP, SCTP |
| 3 | Network | Logical addressing, routing | Internet | IP, ICMP, IGMP, IPsec |
| 2 | Data Link | Physical addressing, error detection, frame sync | Network Access | Ethernet, ARP, PPP, 802.11 |
| 1 | Physical | Bits over physical medium (fibre, copper, radio) | Network Access | Ethernet PHY, DSL, 802.11 PHY |

**Security relevance by layer:**

- **Layer 7** - Application-level attacks (XSS, SQLi, API abuse). WAFs and application firewalls operate here.
- **Layer 4** - SYN floods, port scanning. Stateful firewalls inspect TCP/UDP headers.
- **Layer 3** - IP spoofing, routing attacks (BGP hijack). ACLs and network firewalls filter here.
- **Layer 2** - ARP spoofing, CAM table overflow, VLAN hopping. Switch port security and 802.1X mitigate.

!!! info "External Resources"
    - [OSI Model - Cloudflare](https://www.cloudflare.com/learning/ddos/glossary/open-systems-interconnection-model-osi/) (Cloudflare)
    - [TCP/IP Guide](http://www.tcpipguide.com/free/t_toc.htm) (TCPIPGuide.com)
    - [Cisco - Internetworking Basics](https://www.cisco.com/E-Learning/bulk/public/tac/cim/cib/using_cisco_ios_software/linked/tcpip.htm) (Cisco)

## IP Addressing & Subnetting

**IPv4** - 32-bit addresses (4.3 billion total). Private ranges defined in RFC 1918:

| Range | CIDR | Usable hosts |
|-------|------|-------------|
| `10.0.0.0` - `10.255.255.255` | `10.0.0.0/8` | ~16.7M |
| `172.16.0.0` - `172.31.255.255` | `172.16.0.0/12` | ~1M |
| `192.168.0.0` - `192.168.255.255` | `192.168.0.0/16` | ~65K |

**IPv6** - 128-bit addresses. Link-local (`fe80::/10`), global unicast (`2000::/3`).

**Subnetting essentials:**

- Subnet mask divides IP into network and host portions
- CIDR notation: `/24` = 256 addresses (254 usable), `/16` = 65536 addresses
- Broadcast address: all host bits set to 1
- Network address: all host bits set to 0

**Security implications:**

- Proper subnetting enables network segmentation and access control
- Micro-segmentation limits lateral movement after compromise
- Private addressing combined with NAT hides internal topology

!!! info "External Resources"
    - [Subnetting Practice - Subnet Calculator](https://www.subnet-calculator.com/) (SubnetCalculator)
    - [IPv6 Addressing - RIPE NCC](https://www.ripe.net/publications/ipv6-info-centre/) (RIPE NCC)
    - [RFC 1918 - Address Allocation for Private Internets](https://datatracker.ietf.org/doc/html/rfc1918) (IETF)

## TCP vs UDP

| Property | TCP | UDP |
|----------|-----|-----|
| **Connection** | Connection-oriented (3-way handshake: SYN, SYN-ACK, ACK) | Connectionless |
| **Reliability** | Guaranteed delivery, ordering, retransmission | Best-effort, no guarantees |
| **Flow control** | Sliding window, congestion control (throttles on packet loss) | None - sends at application rate |
| **Header size** | 20-60 bytes | 8 bytes (source port, dest port, length, checksum) |
| **Use cases** | HTTP/S, SSH, SMTP, FTP, database connections | DNS queries, VoIP, streaming, DHCP, traceroute, gaming |
| **Security notes** | SYN flood attacks exploit handshake; stateful firewalls track connections | UDP floods harder to filter; no connection state to track |

**TCP handshake:**

1. Client sends `SYN` (sequence number)
2. Server responds `SYN-ACK` (acknowledges, sends own sequence)
3. Client sends `ACK` (connection established)

**TCP teardown:** `FIN` -> `ACK` -> `FIN` -> `ACK` (four-way close) or `RST` for abrupt termination.

Streaming media over UDP can saturate shared network links, degrading TCP connections on the same path because TCP backs off under congestion while UDP does not.

!!! info "External Resources"
    - [TCP vs UDP - Cloudflare](https://www.cloudflare.com/learning/ddos/glossary/tcp-ip/) (Cloudflare)
    - [RFC 793 - TCP Specification](https://datatracker.ietf.org/doc/html/rfc793) (IETF)
    - [RFC 768 - UDP Specification](https://datatracker.ietf.org/doc/html/rfc768) (IETF)

## Routing & Switching Fundamentals

**Switching (Layer 2):**

- Forwards frames based on MAC addresses using a CAM (Content Addressable Memory) table
- Learns MAC-to-port mappings dynamically
- Broadcast domains bounded by VLANs; collision domains bounded by switch ports

**Routing (Layer 3):**

- Forwards packets based on IP addresses using routing tables
- **Static routes** - manually configured
- **Dynamic routing protocols** - OSPF, BGP, RIP, EIGRP
- BGP (Border Gateway Protocol) connects autonomous systems and holds the internet together; BGP hijacking is a serious threat

**VLANs:**

- Logically segment a physical switch into separate broadcast domains
- VLAN tagging (802.1Q) allows trunking between switches
- VLAN hopping attacks exploit misconfigured trunk ports or double-tagging

!!! info "External Resources"
    - [BGP Explained - Cloudflare](https://www.cloudflare.com/learning/security/glossary/what-is-bgp/) (Cloudflare)
    - [VLAN Security Best Practices - Cisco](https://www.cisco.com/c/en/us/td/docs/switches/lan/catalyst4500/12-2/25ew/configuration/guide/conf/vlans.html) (Cisco)
    - [RPKI - Securing BGP Routing](https://www.cloudflare.com/learning/security/glossary/what-is-rpki/) (Cloudflare)

## NAT & DHCP

**NAT (Network Address Translation):**

- Translates private IP addresses to public IPs for internet access
- **SNAT/PAT** - many internal hosts share one public IP (most common)
- **DNAT** - maps external IP/port to internal server (port forwarding)
- Hides internal network topology from external observers
- IPv6 was designed to eliminate NAT, though NAT66 exists for edge cases

**DHCP (Dynamic Host Configuration Protocol):**

- Ports: UDP 67 (server), UDP 68 (client); IPv6 uses ports 546/547
- Assigns IP address, subnet mask, default gateway, DNS servers

**DHCP process (DORA):**

1. `DHCPDISCOVER` - client broadcasts request
2. `DHCPOFFER` - server offers an address
3. `DHCPREQUEST` - client accepts the offer
4. `DHCPACK` - server confirms the lease

**DHCP modes:**

- **Dynamic** - leases IP addresses temporarily
- **Automatic** - remembers MAC-IP pairing in a table, reassigns same IP
- **Manual/Static** - administrator-assigned fixed IP per MAC

**Security concerns:**

- DHCP starvation attacks exhaust the address pool
- Rogue DHCP servers can redirect traffic (PitM)
- DHCP snooping on switches validates DHCP messages and builds trusted binding tables

!!! info "External Resources"
    - [NAT Explained - Cloudflare](https://www.cloudflare.com/learning/network-layer/what-is-nat/) (Cloudflare)
    - [DHCP Snooping - Cisco](https://www.cisco.com/c/en/us/td/docs/switches/lan/catalyst4500/12-2/25ew/configuration/guide/conf/dhcp.html) (Cisco)
    - [RFC 2131 - DHCP](https://datatracker.ietf.org/doc/html/rfc2131) (IETF)

## HTTP/HTTPS

**HTTP (Port 80) / HTTPS (Port 443):**

HTTP is a stateless, text-based request-response protocol. HTTPS wraps HTTP in TLS for confidentiality and integrity.

**Request structure:**

```http
GET /api/users HTTP/1.1
Host: example.com
Accept: application/json
Accept-Encoding: gzip
Connection: keep-alive
```

**Key request headers:** `Host`, `Accept`, `Accept-Language`, `Accept-Charset`, `Accept-Encoding`, `Connection`, `Referer`, `Authorization`, `Cookie`, `User-Agent`.

**Response status codes:**

| Range | Category | Examples |
|-------|----------|---------|
| 1xx | Informational | 100 Continue, 101 Switching Protocols |
| 2xx | Success | 200 OK, 201 Created, 204 No Content |
| 3xx | Redirection | 301 Moved Permanently, 302 Found, 304 Not Modified |
| 4xx | Client Error | 400 Bad Request, 401 Unauthorized, 403 Forbidden, 404 Not Found, 429 Too Many Requests |
| 5xx | Server Error | 500 Internal Server Error, 502 Bad Gateway, 503 Service Unavailable |

**HTTP/2** - binary framing, multiplexed streams, header compression (HPACK), server push.

**HTTP/3** - uses QUIC (UDP-based), reduces handshake latency, built-in encryption.

**Security headers:** `Strict-Transport-Security`, `Content-Security-Policy`, `X-Content-Type-Options`, `X-Frame-Options`, `Referrer-Policy`. See the [Web Security Fundamentals](../web-security/index.md) section for details.

!!! info "External Resources"
    - [HTTP Overview - MDN](https://developer.mozilla.org/en-US/docs/Web/HTTP/Overview) (MDN)
    - [HTTP/2 Explained](https://http2-explained.haxx.se/) (Daniel Stenberg)
    - [OWASP Secure Headers Project](https://owasp.org/www-project-secure-headers/) (OWASP)

## Common Service Ports

| Port | Protocol | Service | Security Notes |
|------|----------|---------|---------------|
| 20, 21 | TCP | FTP (data, control) | Cleartext credentials; use SFTP instead |
| 22 | TCP | SSH / SFTP / SCP | Key-based auth preferred over passwords |
| 23 | TCP | Telnet | Cleartext; never use in production |
| 25 | TCP | SMTP | Cleartext without STARTTLS; relay abuse |
| 53 | TCP/UDP | DNS | Exfiltration vector; DNSSEC for integrity |
| 67, 68 | UDP | DHCP | Rogue server attacks; use DHCP snooping |
| 80 | TCP | HTTP | Unencrypted; redirect to 443 |
| 110 | TCP | POP3 | Cleartext; use 995 (POP3S) |
| 143 | TCP | IMAP | Cleartext; use 993 (IMAPS) |
| 389 | TCP/UDP | LDAP | Cleartext; use 636 (LDAPS) |
| 443 | TCP | HTTPS / TLS | Primary secure web traffic |
| 465, 587 | TCP | SMTPS / SMTP Submission | Encrypted email submission |
| 993 | TCP | IMAPS | Encrypted IMAP |
| 995 | TCP | POP3S | Encrypted POP3 |
| 3306 | TCP | MySQL | Bind to localhost; use TLS |
| 3389 | TCP/UDP | RDP | Brute-force target; use NLA + MFA |
| 5432 | TCP | PostgreSQL | Bind to localhost; use TLS |
| 8080, 8443 | TCP | HTTP/HTTPS alt | Common for proxies and dev servers |

**Port ranges:**

- **0-1023** - Well-known (privileged); require root/sudo to bind
- **1024-49151** - Registered; used for IANA-registered services
- **49152-65535** - Dynamic/ephemeral; used for client-side connections

!!! info "External Resources"
    - [IANA Service Name and Transport Protocol Port Number Registry](https://www.iana.org/assignments/service-names-port-numbers/service-names-port-numbers.xhtml) (IANA)
    - [Nmap - Common Ports](https://nmap.org/book/nmap-services.html) (Nmap)
    - [SANS - Common Ports Cheat Sheet](https://www.sans.org/blog/common-ports/) (SANS)
