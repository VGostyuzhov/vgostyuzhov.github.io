# Network Attacks

## Person-in-the-Middle (PitM) Attacks

A PitM attack positions the adversary between two communicating parties, intercepting and potentially modifying traffic in real time.

**Common PitM techniques:**

| Technique | Layer | How it works |
|-----------|-------|-------------|
| ARP spoofing | 2 | Falsify ARP replies to redirect traffic through attacker's machine |
| DNS spoofing | 7 | Return forged DNS responses to redirect victim to malicious server |
| DHCP spoofing | 7 | Rogue DHCP server sets attacker as default gateway |
| BGP hijacking | 3 | Announce more-specific routes to attract traffic |
| SSL stripping | 7 | Downgrade HTTPS to HTTP between victim and attacker |
| Wi-Fi evil twin | 2 | Rogue access point mimics legitimate SSID |
| LLMNR/NBT-NS poisoning | 7 | Respond to multicast name resolution requests on Windows networks |

**Why PitM matters:**

- Credential theft (HTTP basic auth, form submissions, session tokens)
- Session hijacking and injection of malicious content
- TLS downgrade to enable plaintext interception
- Lateral movement within a network segment

**Defences:**

- TLS everywhere with HSTS preload and certificate pinning (mobile)
- PKI and mutual TLS for service-to-service communication
- DNSSEC and DNS-over-HTTPS/TLS
- Static ARP entries or Dynamic ARP Inspection (DAI)
- 802.1X port-based authentication
- Network segmentation to limit broadcast domains

!!! info "External Resources"
    - [Person-in-the-Middle Attack - OWASP](https://owasp.org/www-community/attacks/Manipulator-in-the-middle_attack) (OWASP)
    - [SSL Stripping - Moxie Marlinspike](https://moxie.org/2009/02/18/sslstrip.html) (Moxie.org)
    - [HSTS Preload List](https://hstspreload.org/) (Google)

## ARP Spoofing & Cache Poisoning

**How ARP works:**

- ARP (Address Resolution Protocol) maps IP addresses to MAC addresses on a local network segment
- Gratuitous ARP: a host announces its own IP-MAC mapping without being asked
- ARP is stateless and trusts all replies - no built-in authentication

**ARP spoofing attack:**

1. Attacker sends forged ARP replies to the victim, claiming the gateway's IP maps to the attacker's MAC
2. Attacker sends forged ARP replies to the gateway, claiming the victim's IP maps to the attacker's MAC
3. Traffic flows through the attacker (PitM position)
4. Attacker forwards packets to maintain connectivity (transparent interception)

**Tools:** `arpspoof` (dsniff suite), `ettercap`, `bettercap`, `Cain & Abel`

**ARP cache poisoning vs ARP spoofing:**

- ARP spoofing is the act of sending forged ARP packets
- ARP cache poisoning is the result - the victim's ARP table contains incorrect entries
- Both terms are often used interchangeably

**Mitigations:**

- **Dynamic ARP Inspection (DAI)** - switch validates ARP packets against DHCP snooping binding table
- **Static ARP entries** - manual MAC-IP mappings (does not scale)
- **VLAN segmentation** - limits ARP broadcast scope
- **ARP monitoring tools** - `arpwatch`, `XArp` detect changes in ARP tables
- **802.1X** - authenticates devices before granting network access

!!! info "External Resources"
    - [ARP Spoofing - MITRE ATT&CK](https://attack.mitre.org/techniques/T1557/002/) (MITRE)
    - [Dynamic ARP Inspection - Cisco](https://www.cisco.com/c/en/us/td/docs/switches/lan/catalyst4500/12-2/25ew/configuration/guide/conf/dynarp.html) (Cisco)
    - [Bettercap Documentation](https://www.bettercap.org/modules/) (Bettercap)

## DNS Attacks (Exfiltration, Poisoning, Hijacking)

DNS operates primarily over UDP port 53 and is inherently trusting, making it a frequent target.

**DNS exfiltration:**

- Data encoded as subdomains: `[encoded_data].attacker.com`
- Queries resolved by attacker's authoritative nameserver, which extracts the data
- Bypasses HTTP proxies and most firewalls since DNS is rarely blocked
- Does not appear in HTTP logs; only visible in DNS query logs
- Detection: monitor for unusually long subdomains, high query volume to single domains, entropy analysis

**DNS cache poisoning (Kaminsky attack):**

- Attacker floods recursive resolver with forged responses before legitimate response arrives
- If accepted, forged record is cached and served to all clients using that resolver
- DNSSEC prevents this by cryptographically signing DNS records
- Source port randomisation and transaction ID entropy reduce success probability

**DNS hijacking:**

- **Registrar-level** - attacker compromises domain registrar account and changes NS records
- **Router/DHCP-level** - modifies DNS settings on the victim's router or DHCP configuration
- **Local** - malware modifies `/etc/resolv.conf` or Windows DNS settings

**DNS rebinding:**

- Attacker's DNS server alternates between returning attacker IP and internal IP
- Browser's SOP check passes on domain name, but requests now hit internal services
- Mitigate by validating `Host` header, blocking private IPs in DNS responses, using network-level DNS filtering

**DNS tunneling:**

- Encapsulates arbitrary protocols (TCP, HTTP) within DNS queries and responses
- Tools: `iodine`, `dnscat2`, `dns2tcp`
- Detection: anomalous TXT record queries, unusual payload sizes, high query rates to single domain

**DNS sinkholing:**

- Defensive technique: redirect known malicious domains to a controlled server
- Infected hosts contacting C2 domains get redirected to the sinkhole
- Identifies compromised internal hosts; blocks C2 communication
- See [DNS page](dns.md) for additional DNS architecture details

!!! info "External Resources"
    - [DNS Exfiltration Detection - SANS](https://www.sans.org/blog/detecting-dns-tunneling/) (SANS)
    - [DNSSEC - How it Works](https://www.cloudflare.com/dns/dnssec/how-dnssec-works/) (Cloudflare)
    - [DNS Rebinding Attacks - Tavis Ormandy](https://lock.cmpxchg8b.com/rebinder.html) (lock.cmpxchg8b.com)

## BGP Hijacking

**How BGP works:**

- Border Gateway Protocol connects autonomous systems (AS) on the internet
- Routers exchange reachability information (IP prefix announcements)
- BGP has no built-in authentication - routers trust announcements by default

**BGP hijacking attack:**

1. Attacker's AS announces a more-specific prefix (e.g., `/25` instead of the legitimate `/24`)
2. BGP's longest-prefix-match routing causes traffic to flow to the attacker
3. Attacker can inspect, modify, or drop traffic before optionally forwarding it
4. Alternatively, announce the exact same prefix from a closer AS (AS path shorter)

**Real-world incidents:**

- **Pakistan/YouTube (2008)** - Pakistan Telecom announced YouTube's prefix, causing worldwide outage
- **MyEtherWallet/Amazon Route 53 (2018)** - BGP hijack redirected DNS traffic to steal cryptocurrency
- **China Telecom routing anomalies** - repeated incidents routing US/European traffic through Chinese networks

**Impact:** traffic interception, credential theft, route-based censorship, service disruption.

**Mitigations:**

| Defence | How it works |
|---------|-------------|
| **RPKI (Resource Public Key Infrastructure)** | Cryptographically signs route origin authorizations (ROAs); validators reject invalid announcements |
| **BGP Route Filtering** | Configure explicit prefix filters and maximum prefix limits with peers |
| **IRR (Internet Routing Registry)** | Publish routing policy; peers can build filters from IRR data |
| **BGPsec** | Path-level cryptographic validation (not widely deployed) |
| **BGP monitoring** | Services like RIPE RIS, BGPStream, Cloudflare Radar detect anomalous announcements |

!!! info "External Resources"
    - [BGP Hijacking - Cloudflare](https://www.cloudflare.com/learning/security/glossary/bgp-hijacking/) (Cloudflare)
    - [RPKI Documentation - RIPE NCC](https://www.ripe.net/manage-ips-and-asns/resource-management/rpki/) (RIPE NCC)
    - [BGPStream - CAIDA](https://bgpstream.caida.org/) (CAIDA)

## DDoS & Amplification Attacks

**DDoS (Distributed Denial of Service)** overwhelms a target with traffic from many sources, exhausting bandwidth, connection state, or application resources.

**Attack categories:**

| Category | Layer | Examples | Target |
|----------|-------|---------|--------|
| **Volumetric** | 3-4 | UDP flood, ICMP flood, amplification | Bandwidth saturation |
| **Protocol** | 3-4 | SYN flood, ACK flood, Ping of Death | Connection state tables |
| **Application** | 7 | HTTP GET/POST flood, Slowloris, R-U-Dead-Yet | Application resources |

**Amplification attacks:**

- Attacker sends small request with spoofed source IP (victim's IP) to a reflector
- Reflector sends large response to the victim
- Amplification factor = response size / request size

| Protocol | Amplification Factor | Port |
|----------|---------------------|------|
| DNS | ~28-54x | 53 |
| NTP (monlist) | ~556x | 123 |
| Memcached | ~51,000x | 11211 |
| SSDP | ~30x | 1900 |
| CLDAP | ~56-70x | 389 |
| CharGen | ~358x | 19 |

**SYN flood:**

- Sends massive volume of SYN packets with spoofed source IPs
- Server allocates resources for half-open connections, exhausting connection table
- Mitigation: SYN cookies (stateless SYN handling), increase backlog, rate limit SYN packets

**Slowloris:**

- Opens many connections to web server, sends partial HTTP headers slowly
- Keeps connections open indefinitely, exhausting server's connection pool
- Effective against servers with limited concurrent connection capacity (Apache)

**Mitigations:**

- **Edge/CDN** - Cloudflare, AWS Shield, Akamai absorb volumetric floods
- **Rate limiting** - per-IP and per-subnet throttling
- **BCP38/BCP84** - ingress filtering prevents IP spoofing at network edge
- **Anycast** - distributes attack traffic across many PoPs
- **Connection limits** - `max_conns`, timeout tuning, SYN cookies
- **Application-layer** - CAPTCHA, JavaScript challenges, bot detection

!!! info "External Resources"
    - [DDoS Attack Types - Cloudflare](https://www.cloudflare.com/learning/ddos/ddos-attack-tools/ddos-types/) (Cloudflare)
    - [NIST SP 800-189 - Routing Security](https://csrc.nist.gov/publications/detail/sp/800-189/final) (NIST)
    - [US-CERT - Understanding DDoS Attacks](https://www.cisa.gov/sites/default/files/publications/understanding-and-responding-to-distributed-denial-of-service-attacks_508c.pdf) (CISA)

## Port Scanning & Reconnaissance

Port scanning maps the attack surface by identifying open ports, running services, and operating system versions.

**Scanning phases in reconnaissance:**

1. **Host discovery** - identify live hosts (`ping` sweep, ARP scan)
2. **Port scanning** - determine open, closed, and filtered ports
3. **Service enumeration** - identify service versions (`-sV`)
4. **OS fingerprinting** - determine operating system (`-O`)
5. **Vulnerability scanning** - probe for known CVEs (Nmap NSE, Nessus, OpenVAS)

**Port states (Nmap terminology):**

| State | Meaning |
|-------|---------|
| **Open** | Application accepting connections |
| **Closed** | Reachable but no application listening (RST response) |
| **Filtered** | Firewall dropping or rejecting probes (no response or ICMP unreachable) |
| **Unfiltered** | Accessible but open/closed state undetermined (ACK scan) |
| **Open/Filtered** | Cannot determine if open or filtered (UDP, FIN, Xmas scans) |

**Evasion techniques:**

- **Fragmentation** (`-f`) - split probes into small fragments to bypass simple packet filters
- **Decoys** (`-D`) - mix real scan with decoy source IPs
- **Timing** (`-T0` to `-T5`) - slow scans avoid IDS rate-based detection
- **Source port manipulation** (`--source-port 53`) - some firewalls permit traffic from DNS port
- **Idle scan** (`-sI`) - use zombie host to scan without revealing attacker's IP

**Detection:**

- IDS rules for sequential port access patterns
- Connection rate thresholds per source IP
- Failed connection attempt monitoring
- Honeypots on unused IP addresses or ports

For tool details, see [Network Security & Tools - Network Scanning](security-tools.md#network-scanning).

!!! info "External Resources"
    - [Nmap Network Scanning](https://nmap.org/book/) (Nmap)
    - [SANS - Reconnaissance Techniques](https://www.sans.org/blog/pen-test-poster-white-board-reconnaissance/) (SANS)
    - [MITRE ATT&CK - Network Service Discovery](https://attack.mitre.org/techniques/T1046/) (MITRE)

## CAM Table Overflow

**How switches use CAM tables:**

- CAM (Content Addressable Memory) table maps MAC addresses to switch ports
- Switches learn MAC addresses dynamically by observing source MACs of incoming frames
- Frames to unknown MACs are flooded to all ports in the VLAN (unknown unicast flooding)

**CAM table overflow attack (MAC flooding):**

1. Attacker generates thousands of frames with random source MAC addresses
2. CAM table fills to capacity
3. Switch cannot learn new legitimate entries
4. All traffic is flooded to all ports, turning the switch into a hub
5. Attacker can now sniff all traffic on the VLAN segment

**Tools:** `macof` (dsniff suite), `yersinia`

**Mitigations:**

| Defence | How it works |
|---------|-------------|
| **Port security** | Limit maximum MAC addresses per port; shut down port on violation |
| **802.1X** | Authenticate devices before granting port access |
| **DHCP snooping + DAI** | Validate ARP and DHCP against binding table |
| **Private VLANs** | Restrict communication between ports in the same VLAN |
| **Storm control** | Rate-limit broadcast/multicast/unknown unicast traffic |

**Detection:**

- Monitor for rapid MAC address changes on a single port
- SNMP traps for port security violations
- Unusual broadcast traffic volume

!!! info "External Resources"
    - [CAM Table Overflow - MITRE ATT&CK](https://attack.mitre.org/techniques/T1557/) (MITRE)
    - [Port Security - Cisco](https://www.cisco.com/c/en/us/td/docs/switches/lan/catalyst4500/12-2/25ew/configuration/guide/conf/port_sec.html) (Cisco)
    - [Layer 2 Attack Mitigation - Cisco](https://www.cisco.com/c/en/us/td/docs/switches/lan/catalyst6500/ios/15-0SY/configuration/guide/15_0_sy_swcg/layer2_security.html) (Cisco)
