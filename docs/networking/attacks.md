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

- Correct TLS chain/service-identity validation; HSTS for supported browser HTTPS. Preload and application pinning require explicit rollout/recovery planning
- PKI and mutual TLS for service-to-service communication
- DNSSEC validates signed DNS data; DoH/DoT protects a resolver hop. Neither substitutes for service authentication
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
- ARP has no cryptographic sender authentication; hosts retain neighbor-cache state and acceptance behavior varies by implementation

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

- **Dynamic ARP Inspection (DAI)** - switch validates IPv4 ARP against trusted DHCP-snooping bindings or configured policy; correctly configure trust ports and static-host bindings
- **Static ARP entries** - manual MAC-IP mappings (does not scale)
- **VLAN segmentation** - limits ARP broadcast scope
- **ARP monitoring tools** - `arpwatch`, `XArp` detect changes in ARP tables
- **802.1X** - authenticates devices before granting network access

**TLS boundary:** Poisoning Alice's gateway ARP entry changes where her frames go. If her HTTPS client still validates the intended hostname and trusted chain, an attacker without the appropriate key/certificate cannot simply read or rewrite the protected application stream. The attacker can still deny service, collect metadata, or attack other protocols. Network redirection and cryptographic impersonation are separate steps.

!!! info "External Resources"
    - [ARP Spoofing - MITRE ATT&CK](https://attack.mitre.org/techniques/T1557/002/) (MITRE)
    - [Dynamic ARP Inspection - Cisco](https://www.cisco.com/c/en/us/td/docs/switches/lan/catalyst4500/12-2/25ew/configuration/guide/conf/dynarp.html) (Cisco)
    - [Bettercap Documentation](https://www.bettercap.org/modules/) (Bettercap)

## DNS Attacks (Exfiltration, Poisoning, Hijacking)

DNS commonly uses UDP/TCP port 53; acceptance depends on resolver matching, validation, and policy. See [DNS validation and transport](dns.md#dnssec-dot-and-doh) before treating every DNS message as trusted.

**DNS exfiltration:**

- Data encoded as subdomains: `[encoded_data].attacker.com`
- Queries resolved by attacker's authoritative nameserver, which extracts the data
- Can traverse an approved recursive resolver unless that resolver/egress policy detects or blocks the delegation; enforcement determines whether it succeeds
- Plain DNS queries are not HTTP requests; correlate resolver, network, and endpoint telemetry. Unauthorized DoH adds a different visibility/control path
- Detection: monitor for unusually long subdomains, high query volume to single domains, entropy analysis

**DNS cache poisoning (Kaminsky attack):**

- Attacker floods recursive resolver with forged responses before legitimate response arrives
- If accepted, forged record is cached and served to all clients using that resolver
- DNSSEC validation rejects forged covered data when a valid trust chain is required; unsigned delegations and compromised legitimate zone administration have different risks
- Source port randomisation and transaction ID entropy reduce success probability

**DNS hijacking:**

- **Registrar-level** - attacker compromises domain registrar account and changes NS records
- **Router/DHCP-level** - modifies DNS settings on the victim's router or DHCP configuration
- **Local** - malware modifies `/etc/resolv.conf` or Windows DNS settings

**DNS rebinding:**

- Attacker's DNS server alternates between returning attacker IP and internal IP
- Browser's SOP check passes on domain name, but requests now hit internal services
- Require service authentication and intended Host/origin handling; constrain reachable destinations and apply resolver/browser protections where supported. DNS-only filtering misses alternative resolvers, cached results, IPv6, and direct paths

**DNS tunneling:**

- Encapsulates arbitrary protocols (TCP, HTTP) within DNS queries and responses
- Tools: `iodine`, `dnscat2`, `dns2tcp`
- Detection: anomalous TXT record queries, unusual payload sizes, high query rates to single domain

**DNS sinkholing:**

- Defensive technique: redirect known malicious domains to a controlled server
- Infected hosts contacting C2 domains get redirected to the sinkhole
- Supplies investigation leads and disrupts covered DNS-dependent C2 paths; cached answers, direct IPs, alternate resolvers, and benign lookups limit the conclusion
- See [DNS page](dns.md) for additional DNS architecture details

!!! info "External Resources"
    - [DNS Exfiltration Detection - SANS](https://www.sans.org/blog/detecting-dns-tunneling/) (SANS)
    - [DNSSEC - How it Works](https://www.cloudflare.com/dns/dnssec/how-dnssec-works/) (Cloudflare)
    - [DNS Rebinding Attacks - Tavis Ormandy](https://lock.cmpxchg8b.com/rebinder.html) (lock.cmpxchg8b.com)

## BGP Hijacking

**How BGP works:**

- Border Gateway Protocol connects autonomous systems (AS) on the internet
- Routers exchange reachability information (IP prefix announcements)
- Ordinary BGP updates do not cryptographically prove prefix ownership or the full AS path; session authentication and route-acceptance policy are distinct

**BGP hijacking attack:**

1. An attacker announces a more-specific route, for example a `/24` within a legitimate `/23`, on a path whose peers accept that announcement. Many networks filter IPv4 prefixes longer than `/24`, so acceptance cannot be assumed
2. After routing policy accepts and installs the route, forwarding's longest-prefix match can direct traffic toward it
3. The attacker can observe/drop/redirect traffic; correctly authenticated encryption still protects application content unless its trust boundary is also defeated
4. Announcing the same prefix can compete under BGP selection policy; local preference and other attributes mean shorter AS path is not the only deciding factor

**Real-world incidents:**

- **Pakistan/YouTube (2008)** - Pakistan Telecom announced YouTube's prefix, causing worldwide outage
- **MyEtherWallet/Amazon Route 53 (2018)** - BGP hijack redirected DNS traffic to steal cryptocurrency
- **China Telecom routing anomalies** - repeated incidents routing US/European traffic through Chinese networks

**Impact:** traffic interception, credential theft, route-based censorship, service disruption.

**Mitigations:**

| Defence | How it works |
|---------|-------------|
| **RPKI origin validation** | Validators obtain validated ROA data; routers classify announcements and apply configured acceptance policy |
| **BGP Route Filtering** | Configure explicit prefix filters and maximum prefix limits with peers |
| **IRR (Internet Routing Registry)** | Publish routing policy; peers can build filters from IRR data |
| **BGPsec** | Path-level cryptographic validation (not widely deployed) |
| **BGP monitoring** | Services like RIPE RIS, BGPStream, Cloudflare Radar detect anomalous announcements |

### Origin Validation and Its Limits

A **Route Origin Authorization (ROA)** binds a prefix, permitted originating ASN, and maximum prefix length. A relying-party validator produces authenticated origin information; a router compares announcements against it. **Valid** means a covering authorization permits this origin/length; **Invalid** means covering authorization exists but does not permit this combination; **NotFound** means no covering authorization was found. These states do not themselves specify whether the router accepts traffic/routes.

```text
ROA: prefix=203.0.113.0/24, origin=AS64496, maxLength=25
203.0.113.128/25 originated by AS64496 -> Valid
203.0.113.128/25 originated by AS64497 -> Invalid (wrong origin)
203.0.113.128/26 originated by AS64496 -> Invalid (length exceeds maxLength)
198.51.100.0/24 with no covering ROA -> NotFound
# Local validation fixtures using documentation space; Internet import filters
# may reject these lengths independently. Invalid rejection requires policy.
```

Origin validation does not authenticate every AS-path hop or eliminate leaks from an authorized origin. Publishing an incorrect maxLength/origin can invalidate legitimate announcements. Test planned more-specific routes, validator/router connectivity, cached-data expiry, and outage policy. A validator reporting an error while a router keeps accepting Invalid routes does not enforce protection.

In RPKI-to-router distribution, successfully received data can be retained during a temporary outage until its expiry deadline. After expiry, the router removes that expired cache data and reevaluates routes against any remaining applicable data; a previously Invalid announcement can become NotFound if no covering authorization remains. The import policy determines the consequence. RFC 8210 starts/resets the expiry interval on an End of Data message, not merely on failed reconnect attempts. Use the negotiated/configured timers and multi-cache behavior rather than promising one universal stale-data lifetime.

!!! info "External Resources"
    - [BGP Prefix Origin Validation](https://www.rfc-editor.org/rfc/rfc6811.html) (IETF)
    - [RPKI-to-Router Data Distribution](https://www.rfc-editor.org/rfc/rfc8210.html) (IETF)

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

Amplification ratios depend on the request, response, deployment, and rate limits; the historical examples above are not universal current capacities. Reflection requires the attacker to make the reflector send replies to the victim, commonly through source-address spoofing. Source-address validation at originating networks reduces that capability; the victim's firewall cannot impose it on every remote network.

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

### Match the Defense to the Exhausted Resource

```text
Internet flood: 10 Gbit/s -> victim uplink capacity: 1 Gbit/s -> host firewall
# Even if the host drops every attack packet, legitimate packets already lost
# capacity upstream. Host rules cannot recover that saturated link.
```

Use upstream filtering/scrubbing or an appropriately provisioned edge for link saturation. SYN cookies reduce some half-open TCP state pressure, not all CPU/socket/application load. Authenticated bot requests can exhaust expensive database work despite successful TCP/TLS; use operation-aware quotas and capacity limits. Per-IP limits can penalize many users behind one NAT and miss a distributed attack. A CDN protects only traffic routed through it: restrict/monitor direct origin access under the provider's supported design.

!!! info "External Resources"
    - [SYN Floods and Countermeasures](https://www.rfc-editor.org/rfc/rfc4987.html) (IETF)
    - [Source Address Validation](https://www.rfc-editor.org/rfc/rfc2827.html) (IETF)

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
4. Unknown-destination unicast can be flooded within the VLAN according to switch behavior; known destinations may still use learned entries
5. An attacker may observe traffic delivered to their port, but table exhaustion does not universally expose every frame or defeat TLS encryption

**Tools:** `macof` (dsniff suite), `yersinia`

**Mitigations:**

| Defence | How it works |
|---------|-------------|
| **Port security** | Limit maximum MAC addresses per port; shut down port on violation |
| **802.1X** | Authenticate devices before granting port access |
| **DHCP snooping + DAI** | Complementary protection for DHCP/ARP manipulation, not a direct limit on arbitrary source-MAC flooding |
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
