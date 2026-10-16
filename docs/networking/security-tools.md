# Network Security & Tools

## Packet Capture & Analysis

**Wireshark:**

- GUI-based packet analyser; captures and decodes all layers
- Display filters: `tcp.port == 443`, `http.request.method == "POST"`, `dns.qry.name contains "evil"`
- Capture filters (BPF syntax): `host 10.0.0.1`, `port 80`, `tcp and not port 22`
- Follow TCP/HTTP streams to reconstruct conversations
- Export objects (files transferred over HTTP, SMB, etc.)
- With endpoint-supported key logging, decrypt matching captures using traffic-secret logs (`SSLKEYLOGFILE`); a passive capture or server private key alone cannot decrypt ordinary ECDHE TLS traffic

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
tcpdump -i eth0 'port 53'  # Include TCP DNS as well as UDP
```

**tshark:**

- CLI version of Wireshark; supports display filters and field extraction
- Useful for scripted analysis: `tshark -r capture.pcap -T fields -e http.host -e http.request.uri`

!!! info "External Resources"
    - [Wireshark User Guide](https://www.wireshark.org/docs/wsug_html_chunked/) (Wireshark)
    - [tcpdump Manual](https://www.tcpdump.org/manpages/tcpdump.1.html) (tcpdump.org)
    - [SANS - Packet Analysis Cheat Sheet](https://www.sans.org/blog/tips-for-getting-the-most-out-of-wireshark/) (SANS)

### Observation Point and Encryption

A **capture filter** decides which packets are retained; a **display filter** selects packets already present in the capture. Filtering out a packet at capture time cannot be reversed by changing the display filter. A host, SPAN/mirror port, tap, gateway, and VPN interface can observe different tuples and encapsulation. Mirroring may drop packets under load; checksum offload can make host captures show apparent checksum errors that are not transmitted wire errors.

```bash
# Capture a narrow authorized lab flow; -n avoids resolver lookups during capture.
tcpdump -ni eth0 -s 0 -w lab.pcap 'host 192.0.2.10 and port 443'
# Inspect the saved packets; a filter does not decrypt TLS application data.
tcpdump -nn -r lab.pcap
# Wireshark display filter for TCP handshake/retransmission investigation:
# ip.addr == 192.0.2.10 && tcp.port == 443
```

Encrypted traffic still exposes some flow/timing metadata. Key logs provide sensitive decryption capability for matching sessions and must be handled like credentials. Endpoint or authorized proxy telemetry can supply application context that a passive encrypted packet capture cannot. Missing replies can mean loss, filtering, asymmetric observation, or capture loss; combine evidence rather than claiming one cause.

!!! info "External Resources"
    - [Wireshark Capture and Display Filters](https://www.wireshark.org/docs/wsug_html_chunked/ChapterWork.html) (Wireshark)
    - [TLS Decryption and Key Logs](https://wiki.wireshark.org/TLS) (Wireshark)

## Interpret Results Before Changing Policy

A port state is a measurement of this probe from this observer at this time. A SYN/ACK normally supports “open” for a TCP SYN scan; an RST supports “closed”; silence or particular unreachable errors can support “filtered.” For UDP, silence can indicate an open service that does not respond to this payload or a dropped probe. Firewalls, NAT, host policy, protocol response rules, and rate limiting all affect results.

Use a controlled positive path and denied counterpart. Correlate route lookup, host listener, rule counters, server logs, and captures on both sides. A successful ping does not prove TCP/443 works; a missing ping does not prove TCP/443 fails. Do not open broad ports merely to compensate for uncertainty.

!!! info "External Resources"
    - [Nmap Port-State Definitions](https://nmap.org/book/man-port-scanning-basics.html) (Nmap)

## Network Scanning

**Nmap (Network Mapper):**

- Port scanning, service detection, OS fingerprinting, scripting engine (NSE)
- Scan types:

| Scan type | Flag | Observation | Interpretation limit |
|---|---|---|---|
| TCP connect | `-sT` | OS attempts a TCP connection | Reachability from this vantage, not authorization or application health |
| SYN | `-sS` | SYN/ACK, RST, or missing/error response | Privilege/platform requirements; detectable even without full handshake |
| UDP | `-sU` | Application response or ICMP errors | Silence often means open-or-filtered, not confirmed closed |
| FIN/Xmas/Null | `-sF/-sX/-sN` | Response to unusual flags | OS/filter behavior varies; not a guaranteed stealth technique |
| ACK | `-sA` | Response/filter behavior for ACK probes | Does not classify service open versus closed |
| Host discovery | `-sn` | Responses to discovery probes | Missing responses do not prove a host is absent |

```bash
# Documentation target: substitute only an explicitly authorized lab host.
nmap -sT -p 22,443 --reason 192.0.2.10
# Host-discovery filtering can hide a reachable service; -Pn skips discovery.
nmap -Pn -sT -p 443 --reason 192.0.2.10
# Service probing sends additional traffic; banner guesses can be wrong.
nmap -sV -p 443 192.0.2.10
```


**Masscan:**

- Asynchronous high-rate scanner; capacity/results depend on scope, link, limits, and environment
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
- Observed TTL is reduced by forwarding; initial TTL conventions are weak OS hints, not authenticated identity
- Routers may return ICMP Time Exceeded, but filtering/rate limits can hide hops; destination behavior depends on the UDP/ICMP/TCP probe

!!! info "External Resources"
    - [Zeek Documentation](https://docs.zeek.org/en/current/) (Zeek)
    - [Arkime Documentation](https://arkime.com/learn) (Arkime)
    - [NetFlow Overview - Cisco](https://www.cisco.com/c/en/us/products/ios-nx-os-software/ios-netflow/index.html) (Cisco)

## VPN & Tunneling

**VPN types:**

| Type | Protocol | Use Case | Notes |
|------|----------|----------|-------|
| **IPsec** | ESP/AH | Site-to-site, remote access | Operates at Layer 3; transport and tunnel modes |
| **OpenVPN** | Control/data-channel VPN over UDP or TCP | Remote access, site-to-site | TLS authenticates/negotiates its control channel; data-channel protection is separate |
| **WireGuard** | UDP | Remote access, site-to-site | Minimal codebase, modern cryptography, high performance |
| **L2TP/IPsec** | L2TP + IPsec | Legacy remote access | L2TP provides tunneling, IPsec provides encryption |
| **SSH tunneling** | SSH | Ad-hoc port forwarding | `-L` local, `-R` remote, `-D` dynamic SOCKS |

**Security considerations:**

- A VPN protects the configured path to its gateway; the gateway sees routed traffic/destination metadata, while correctly validated HTTPS can still protect application content beyond it
- Split tunneling selects which routes use the VPN; assess the remaining local/Internet paths and endpoint trust instead of assigning a universal security rating
- Full tunneling intends to route relevant traffic through the gateway; actual routes, DNS, IPv6, exceptions, and disconnect policy determine containment
- Always verify VPN kill-switch behaviour to prevent leaks on disconnect

### Trace Tunnel Boundaries

```text
Client -> encrypted VPN tunnel -> corporate gateway -> destination
# Encryption of the VPN ends at the gateway; HTTPS can continue end-to-end.
# A route to 10.0.0.0/8 does not automatically capture DNS or all IPv6 traffic.
```

```bash
# Linux read-only inspection; compare connected and disconnected lab states.
ip route get 192.0.2.80
ip -6 route get 2001:db8::80
resolvectl status  # On systemd-resolved hosts; resolver behavior is OS-specific.
# Check policy-selected DNS and IPv6 paths, not just the IPv4 default route.
```

Test failure behavior: when the tunnel drops, should protected traffic stop or use the ordinary default route? A kill switch requires enforced rules covering the intended address families, DNS, exceptions, and startup/disconnect races. Gateway membership still does not authorize every internal service. Define device posture, user/workload authentication, and per-service authorization separately.

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
| WPA2 | Commonly AES-CCMP data protection; Personal PSK or Enterprise EAP authentication | Strong credentials and patched implementations required; KRACK and offline guessing have different prerequisites |
| WPA3-Personal | SAE authentication/key establishment plus negotiated data protection | Resists passive offline password guessing under intended protocol assumptions; patch implementations and assess transition modes |

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
- Enterprise: 802.1X/EAP with provisioned authentication-server trust/name validation and an appropriate client method, such as EAP-TLS. Matching SSID or accepting any server certificate is insufficient
- Wireless IDS to detect rogue APs and deauth floods
- Disable WPS (Wi-Fi Protected Setup) - known PIN brute-force vulnerability

### Enterprise Wi-Fi Server Validation

In 802.1X, the supplicant (client), authenticator (AP/switch), and authentication server exchange EAP authentication before network admission. A rogue AP can copy a legitimate SSID. The client must validate the intended authentication server's certificate under its provisioned trust roots and server-name policy; users should not accept an unexpected certificate to “fix Wi-Fi.” With tunneled password methods, validate that server before releasing inner credentials. EAP-TLS also authenticates the client by its certificate/key, but admission still needs a configured authorization policy.

```text
Provisioned server names: radius.example.com
Provisioned trust:       enterprise-authentication CA
Observed server:         radius.attacker.example, another CA
Decision:                reject before inner credentials/admission
# Correct SSID does not override the trust/name mismatch.
```

WPA3-Personal's SAE is not the same as Enterprise EAP-TLS and is not the data cipher. Protected management frames reduce certain forged deauthentication paths where negotiated/enforced; they do not prevent radio interference or every availability attack. Configure supported modes and patch endpoints/APs rather than treating a protocol label as complete protection.

!!! info "External Resources"
    - [EAP Server Certificate Validation](https://learn.microsoft.com/en-us/windows-server/networking/technologies/extensible-authentication-protocol/network-access#server-certificate-validation) (Microsoft)

!!! info "External Resources"
    - [Aircrack-ng Documentation](https://www.aircrack-ng.org/documentation.html) (Aircrack-ng)
    - [Wi-Fi Alliance - WPA3](https://www.wi-fi.org/discover-wi-fi/security) (Wi-Fi Alliance)
    - [KRACK Attacks](https://www.krackattacks.com/) (KU Leuven)
