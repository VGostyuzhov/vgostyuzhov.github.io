# Core Networking

Study this page for packet paths and reference facts, then use [DNS](dns.md), [TLS](tls.md), and [segmentation and firewalls](segmentation.md) to understand their enforcement boundaries. [Network protocols](protocols.md), [attacks](attacks.md), and [tools](security-tools.md) connect those mechanisms to operational checks. Finish with the [tricky interview questions](interview-questions.md), which include separate answer and commented-example expanders.

## Follow One HTTPS Request

Assume a workstation `10.0.10.8/24` connects to `api.example`, resolved to documentation address `192.0.2.80`. Its default gateway is `10.0.10.1`. DNS has supplied a destination, not proof of that destination's application identity.

1. The host checks its routing table. Because `192.0.2.80` is outside its directly connected subnet, it selects the gateway rather than trying to resolve the remote server's MAC address.
2. On Ethernet/IPv4, ARP resolves the gateway's local IP to a MAC address. The outgoing frame's destination MAC is the gateway; its IP destination remains `192.0.2.80`.
3. Each router forwards toward the IP destination and uses the next link's framing. Firewalls along the path evaluate applicable packet/flow policy. NAT, if configured, rewrites the relevant address/port tuple.
4. With HTTP/1.1 or HTTP/2 over TLS/TCP, TCP establishes transport state, then TLS authenticates the service and establishes protected traffic keys. HTTP/3 instead uses QUIC over UDP with TLS 1.3 integrated into its handshake.
5. The receiving application authenticates/authorizes its own operations. A network rule allowing port 443 does not grant access to every API object.
6. Replies need valid routes and matching policy/state. A successful outbound packet does not prove that the reverse path works.

```text
Ethernet: src=workstation-MAC, dst=gateway-MAC
IPv4:     src=10.0.10.8,       dst=192.0.2.80
TCP:      src=51000,          dst=443
# MAC addresses describe this link. The destination IP describes the remote host.
# A NAT gateway may rewrite the source IP/port; it does not authorize the API call.
```

!!! info "External Resources"
    - [IPv4 Router Forwarding Requirements](https://www.rfc-editor.org/rfc/rfc1812.html#section-5) (IETF)
    - [HTTP/3 and QUIC Transport](https://www.rfc-editor.org/rfc/rfc9114.html) (IETF)

## OSI Model & TCP/IP Stack

The OSI model provides a layered abstraction for network communication. The TCP/IP model describes the Internet protocol suite. Layer placement is a teaching approximation: TLS, ARP, tunnels, and application protocols do not always map neatly to a single OSI layer.

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
- CIDR notation: an IPv4 `/24` contains 256 addresses, ordinarily 254 host addresses; `/31` point-to-point and `/32` host routes have different semantics. IPv6 does not use the IPv4 broadcast/host-count convention.
- Broadcast address: all host bits set to 1
- Network address: all host bits set to 0

**Security implications:**

- Proper subnetting enables network segmentation and access control
- Micro-segmentation limits lateral movement after compromise
- Private addresses are not globally routed in the ordinary Internet model, but routed enterprise/VPN paths can still reach them. NAT changes visible tuples; it is not an access-control guarantee. See [segmentation](segmentation.md).

!!! info "External Resources"
    - [Subnetting Practice - Subnet Calculator](https://www.subnet-calculator.com/) (SubnetCalculator)
    - [IPv6 Addressing - RIPE NCC](https://www.ripe.net/publications/ipv6-info-centre/) (RIPE NCC)
    - [RFC 1918 - Address Allocation for Private Internets](https://datatracker.ietf.org/doc/html/rfc1918) (IETF)

## TCP vs UDP

| Property | TCP | UDP |
|----------|-----|-----|
| **Connection** | Connection-oriented (3-way handshake: SYN, SYN-ACK, ACK) | Connectionless |
| **Reliability** | Reliable ordered byte stream with retransmission; connection failure is still possible | Best-effort, no guarantees |
| **Flow control** | Sliding window, congestion control (throttles on packet loss) | No built-in flow/congestion control; applications/transports must implement suitable behavior |
| **Header size** | 20-60 bytes | 8 bytes (source port, dest port, length, checksum) |
| **Use cases** | HTTP/S, SSH, SMTP, FTP, database connections | DNS queries, VoIP, streaming, DHCP, traceroute, gaming |
| **Security notes** | SYN flood attacks exploit handshake; stateful firewalls track connections | Firewalls can track UDP tuples and idle timeouts despite the absence of a handshake |

**TCP handshake:**

1. Client sends `SYN` (sequence number)
2. Server responds `SYN-ACK` (acknowledges, sends own sequence)
3. Client sends `ACK` (connection established)

**TCP teardown:** `FIN` -> `ACK` -> `FIN` -> `ACK` (four-way close) or `RST` for abrupt termination.

A UDP application that sends without congestion control can saturate a shared link. UDP-based protocols such as QUIC implement their own congestion control and reliability. TCP reliability is not authentication: sequence/ACK handling does not prove application identity.

!!! info "External Resources"
    - [TCP vs UDP - Cloudflare](https://www.cloudflare.com/learning/ddos/glossary/tcp-ip/) (Cloudflare)
    - [RFC 9293 - TCP Specification](https://www.rfc-editor.org/rfc/rfc9293.html) (IETF)
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

**Boundary example:** VLAN 10 and VLAN 20 separate ordinary Ethernet broadcasts. If their gateway routes between them without an ACL/firewall restriction, hosts can still connect across the VLANs. VLAN membership is topology; the [permitted-flow policy](segmentation.md#from-requirements-to-permitted-flows) determines access.

!!! info "External Resources"
    - [BGP Explained - Cloudflare](https://www.cloudflare.com/learning/security/glossary/what-is-bgp/) (Cloudflare)
    - [VLAN Security Best Practices - Cisco](https://www.cisco.com/c/en/us/td/docs/switches/lan/catalyst4500/12-2/25ew/configuration/guide/conf/vlans.html) (Cisco)
    - [RPKI - Securing BGP Routing](https://www.cloudflare.com/learning/security/glossary/what-is-rpki/) (Cloudflare)

## NAT & DHCP

**NAT (Network Address Translation):**

- Rewrites addresses, often with ports; common deployments translate private IPv4 sources for Internet access
- **SNAT/PAT** - many internal hosts share one public IP (most common)
- **DNAT** - maps external IP/port to internal server (port forwarding)
- Hides internal network topology from external observers
- IPv6 commonly uses globally routable addressing without NAT; it still requires explicit ingress/egress policy

**DHCP (Dynamic Host Configuration Protocol):**

- Ports: UDP 67 (server), UDP 68 (client); IPv6 uses ports 546/547
- DHCPv4 assigns an IPv4 address and options such as subnet mask, default gateway, and DNS servers. DHCPv6 is a different exchange; IPv6 default-router discovery uses Router Advertisements

**DHCPv4 process (DORA):**

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

HTTP has stateless request/response semantics; HTTP/1.1 uses textual message syntax, while HTTP/2 and HTTP/3 use binary framing. HTTPS adds a protected authenticated connection, not automatic application authorization.

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
| 389 | TCP (UDP for CLDAP) | LDAP | Require authenticated TLS for password binds; StartTLS or LDAPS by policy |
| 443 | TCP/UDP | HTTPS | TCP for HTTP/1.1 and HTTP/2; UDP for HTTP/3 |
| 465, 587 | TCP | SMTP submission | 465 implicit TLS; 587 must enforce successful STARTTLS when required |
| 993 | TCP | IMAPS | Encrypted IMAP |
| 995 | TCP | POP3S | Encrypted POP3 |
| 3306 | TCP | MySQL | Bind to localhost; use TLS |
| 3389 | TCP/UDP | RDP | Brute-force target; use NLA + MFA |
| 5432 | TCP | PostgreSQL | Bind to localhost; use TLS |
| 8080, 8443 | TCP | HTTP/HTTPS alt | Common for proxies and dev servers |

**Port ranges:**

- **0-1023** - System/well-known ports; binding privilege is OS/configuration dependent (Linux can use capabilities or a configured unprivileged-port boundary)
- **1024-49151** - Registered; used for IANA-registered services
- **49152-65535** - IANA dynamic/private range; actual ephemeral ranges are OS/configuration dependent

!!! info "External Resources"
    - [IANA Service Name and Transport Protocol Port Number Registry](https://www.iana.org/assignments/service-names-port-numbers/service-names-port-numbers.xhtml) (IANA)
    - [Nmap - Common Ports](https://nmap.org/book/nmap-services.html) (Nmap)
    - [SANS - Common Ports Cheat Sheet](https://www.sans.org/blog/common-ports/) (SANS)
