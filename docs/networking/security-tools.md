# Network Security & Tools

## Packet Capture & Analysis

**Wireshark:**

- GUI-based packet analyser; captures and decodes all layers
- Display filters: `tcp.port == 443`, `http.request.method == "POST"`, `dns.qry.name contains "evil"`
- Capture filters (BPF syntax): `host 10.0.0.1`, `port 80`, `tcp and not port 22`
- Follow TCP/HTTP streams to reconstruct conversations
- Export objects (files transferred over HTTP, SMB, etc.)
- Decrypt TLS with pre-master secret log file (`SSLKEYLOGFILE`)

**tcpdump:**

- CLI packet capture tool; useful on servers without GUI
- Common usage:

```bash
# Capture on interface eth0, write to file
tcpdump -i eth0 -w capture.pcap

# Filter by host and port
tcpdump -i any host 10.0.0.1 and port 443

# Read capture file with verbose output
tcpdump -r capture.pcap -vvv

# Capture only DNS traffic
tcpdump -i eth0 udp port 53
```

**tshark:**

- CLI version of Wireshark; supports display filters and field extraction
- Useful for scripted analysis: `tshark -r capture.pcap -T fields -e http.host -e http.request.uri`

!!! info "External Resources"
    - [Wireshark User Guide](https://www.wireshark.org/docs/wsug_html_chunked/) (Wireshark)
    - [tcpdump Manual](https://www.tcpdump.org/manpages/tcpdump.1.html) (tcpdump.org)
    - [SANS - Packet Analysis Cheat Sheet](https://www.sans.org/blog/tips-for-getting-the-most-out-of-wireshark/) (SANS)

## Network Scanning

**Nmap (Network Mapper):**

- Port scanning, service detection, OS fingerprinting, scripting engine (NSE)
- Scan types:

| Scan Type | Flag | Description | Stealth |
|-----------|------|-------------|---------|
| TCP Connect | `-sT` | Full 3-way handshake | Low (logged) |
| SYN (Half-open) | `-sS` | Sends SYN, analyses response, no full connection | Medium |
| UDP | `-sU` | Sends UDP probes; slow due to ICMP rate limiting | Medium |
| FIN/Xmas/Null | `-sF/-sX/-sN` | Sends unusual flag combinations | Higher (evades some firewalls) |
| ACK | `-sA` | Maps firewall rules (filtered vs unfiltered) | N/A (reconnaissance) |
| Ping sweep | `-sn` | Host discovery only, no port scan | Low |

- Common usage:

```bash
# Fast scan of top 1000 ports
nmap -sS -T4 10.0.0.0/24

# Service version detection + OS fingerprint
nmap -sV -O -A target.com

# Scan specific ports with scripts
nmap -sS -p 22,80,443 --script=vuln target.com
```

**Masscan:**

- Asynchronous scanner; can scan the entire internet in minutes
- Useful for large-scale reconnaissance; less accurate than Nmap for service detection

**Zmap:**

- Single-packet probe scanner for internet-wide surveys
- Sends one packet per target; stateless design for speed

!!! info "External Resources"
    - [Nmap Reference Guide](https://nmap.org/book/man.html) (Nmap)
    - [Nmap NSE Scripts](https://nmap.org/nsedoc/) (Nmap)
    - [SANS - Nmap Cheat Sheet](https://www.sans.org/blog/nmap-cheat-sheet/) (SANS)

## Traffic Monitoring

**Network flow analysis:**

- **NetFlow / IPFIX** - Cisco-originated; records metadata about network conversations (source/dest IP, ports, bytes, packets, duration)
- **sFlow** - sampled packet headers; lower overhead, works across vendors
- Useful for: bandwidth monitoring, anomaly detection, forensic timeline reconstruction

**IDS/IPS placement:**

- Inline (IPS) or passive tap/SPAN port (IDS)
- See [Detection Systems](../detection-monitoring/detection-systems.md) for detailed IDS/IPS coverage

**Traffic monitoring tools:**

| Tool | Type | Primary Use |
|------|------|-------------|
| **ntopng** | Flow analyser | Real-time network traffic monitoring |
| **Zeek (Bro)** | Network security monitor | Protocol analysis, connection logging, file extraction |
| **Arkime (Moloch)** | Full packet capture | Large-scale indexed packet capture and search |
| **Suricata** | IDS/IPS + NSM | Rule-based detection with protocol logging |
| **Nagios / Zabbix** | Infrastructure monitoring | Uptime, bandwidth, service health |

**ICMP and traceroute:**

- `ping` uses ICMP Echo Request/Reply - basic reachability test
- `traceroute` typically uses UDP (Linux) or ICMP (Windows `tracert`); increments TTL to map path
- Initial TTL: 64 (*nix), 128 (Windows) - useful for OS fingerprinting
- Each hop returns ICMP Time Exceeded; destination returns ICMP Port Unreachable or Echo Reply

!!! info "External Resources"
    - [Zeek Documentation](https://docs.zeek.org/en/current/) (Zeek)
    - [Arkime Documentation](https://arkime.com/learn) (Arkime)
    - [NetFlow Overview - Cisco](https://www.cisco.com/c/en/us/products/ios-nx-os-software/ios-netflow/index.html) (Cisco)

## VPN & Tunneling

**VPN types:**

| Type | Protocol | Use Case | Notes |
|------|----------|----------|-------|
| **IPsec** | ESP/AH | Site-to-site, remote access | Operates at Layer 3; transport and tunnel modes |
| **OpenVPN** | TLS over UDP/TCP | Remote access, site-to-site | Open source; highly configurable |
| **WireGuard** | UDP | Remote access, site-to-site | Minimal codebase, modern cryptography, high performance |
| **L2TP/IPsec** | L2TP + IPsec | Legacy remote access | L2TP provides tunneling, IPsec provides encryption |
| **SSH tunneling** | SSH | Ad-hoc port forwarding | `-L` local, `-R` remote, `-D` dynamic SOCKS |

**Security considerations:**

- VPN hides traffic from ISP but exposes it to the VPN provider
- Split tunneling sends only corporate traffic through VPN; reduces bandwidth but increases risk
- Full tunneling routes all traffic through VPN; more secure but higher latency
- Always verify VPN kill-switch behaviour to prevent leaks on disconnect

**Tor:**

- Onion routing through three relays (guard, middle, exit)
- Traffic pattern is distinctive on the network - easily identified even though content is encrypted
- Exit node sees unencrypted traffic if destination is HTTP (not HTTPS)
- Law enforcement investigation techniques: traffic correlation, compromised nodes, application-layer deanonymisation

!!! info "External Resources"
    - [WireGuard Documentation](https://www.wireguard.com/) (WireGuard)
    - [OpenVPN Documentation](https://openvpn.net/community-resources/) (OpenVPN)
    - [Tor Project - How Tor Works](https://www.torproject.org/about/overview.html.en) (Tor Project)

## Proxy & Interception Tools

**Forward proxies:**

- Client configures proxy; proxy forwards requests on client's behalf
- Use cases: content filtering, caching, access control, anonymisation
- Multiple proxy hops do not guarantee anonymity - traffic correlation, timing analysis, and compromised nodes defeat chaining

**Reverse proxies:**

- Sits in front of servers; clients connect to proxy, unaware of backend
- Use cases: load balancing, TLS termination, WAF, caching, DDoS mitigation

**Interception proxies (for security testing):**

| Tool | Primary Use |
|------|-------------|
| **Burp Suite** | Web application security testing; intercepts, modifies, replays HTTP/S requests |
| **OWASP ZAP** | Open-source web app scanner and proxy |
| **mitmproxy** | CLI/scriptable HTTP/S proxy for testing and debugging |
| **Fiddler** | Windows HTTP debugging proxy |

**TLS interception:**

- Proxy generates on-the-fly certificates signed by a local CA
- Client must trust the proxy's CA certificate
- Used in corporate environments for DLP and threat inspection
- Breaks certificate pinning; can introduce security risks if proxy is compromised

!!! info "External Resources"
    - [Burp Suite Documentation](https://portswigger.net/burp/documentation) (PortSwigger)
    - [OWASP ZAP](https://www.zaproxy.org/) (OWASP)
    - [mitmproxy Documentation](https://docs.mitmproxy.org/) (mitmproxy)

## Wireless Security Tools

**Wi-Fi security protocols:**

| Protocol | Encryption | Status |
|----------|-----------|--------|
| WEP | RC4 (24-bit IV) | Broken; crackable in minutes |
| WPA | TKIP (RC4-based) | Deprecated; known weaknesses |
| WPA2 | AES-CCMP | Current standard; vulnerable to KRACK and PMKID attacks |
| WPA3 | SAE (Dragonfly handshake) | Current; resistant to offline dictionary attacks |

**Wireless security tools:**

- **Aircrack-ng** - suite for monitoring, attacking, testing, and cracking Wi-Fi (capture handshakes, deauth attacks, WEP/WPA cracking)
- **Kismet** - wireless network detector, sniffer, and IDS
- **Wifite** - automated wireless auditing tool (wraps aircrack-ng)
- **Wireshark** - can capture wireless frames in monitor mode

**Common wireless attacks:**

- **Evil twin** - rogue access point mimicking legitimate SSID
- **Deauthentication** - forces client disconnection to capture handshake on reconnect
- **KRACK** - key reinstallation attack against WPA2 four-way handshake
- **PMKID capture** - extracts hash from AP without client interaction (WPA2)
- **Rogue AP / karma attack** - responds to all probe requests

**Mitigations:**

- Use WPA3 where supported; WPA2 with strong PSK as minimum
- Enterprise: 802.1X with RADIUS and certificate-based authentication
- Wireless IDS to detect rogue APs and deauth floods
- Disable WPS (Wi-Fi Protected Setup) - known PIN brute-force vulnerability

!!! info "External Resources"
    - [Aircrack-ng Documentation](https://www.aircrack-ng.org/documentation.html) (Aircrack-ng)
    - [Wi-Fi Alliance - WPA3](https://www.wi-fi.org/discover-wi-fi/security) (Wi-Fi Alliance)
    - [KRACK Attacks](https://www.krackattacks.com/) (KU Leuven)
