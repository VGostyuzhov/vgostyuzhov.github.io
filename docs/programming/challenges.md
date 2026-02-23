# Security Challenges

Hands-on coding exercises that reinforce security concepts. Each challenge includes the problem statement, key concepts, and a reference implementation outline.

## Challenge 1: Caesar Cipher Implementation

**Objective:** Implement encryption and decryption for the Caesar cipher, then extend to brute-force all 26 rotations.

**Key concepts:** Character encoding, modular arithmetic, brute-force attacks on weak encryption.

```python
def caesar_encrypt(plaintext: str, shift: int) -> str:
    result = []
    for ch in plaintext:
        if ch.isalpha():
            base = ord("A") if ch.isupper() else ord("a")
            result.append(chr((ord(ch) - base + shift) % 26 + base))
        else:
            result.append(ch)
    return "".join(result)


def caesar_decrypt(ciphertext: str, shift: int) -> str:
    return caesar_encrypt(ciphertext, -shift)


def caesar_bruteforce(ciphertext: str) -> list[tuple[int, str]]:
    return [(i, caesar_decrypt(ciphertext, i)) for i in range(26)]
```

**Extension ideas:**

- Implement frequency analysis to auto-detect the most likely shift
- Extend to Vigenere cipher (polyalphabetic substitution)
- Build a ROT13 detector for encoded strings in log files

---

## Challenge 2: Log Parser and Analyser

**Objective:** Parse Apache/Nginx access logs, extract key fields, and identify suspicious patterns (brute force, scanning, anomalous user agents).

**Key concepts:** Regex extraction, statistical analysis, anomaly detection, IOC extraction.

```python
import re
from collections import Counter

LOG_PATTERN = re.compile(
    r'^(\S+) \S+ \S+ \[([^\]]+)\] "(\S+) (\S+) \S+" (\d{3}) (\d+|-)'
)

def parse_log_line(line: str) -> dict | None:
    m = LOG_PATTERN.match(line)
    if not m:
        return None
    return {
        "ip": m.group(1),
        "timestamp": m.group(2),
        "method": m.group(3),
        "path": m.group(4),
        "status": int(m.group(5)),
        "size": int(m.group(6)) if m.group(6) != "-" else 0,
    }

def detect_bruteforce(entries: list[dict], threshold: int = 10) -> list[str]:
    """Find IPs with excessive 401/403 responses."""
    failed = Counter(
        e["ip"] for e in entries if e["status"] in (401, 403)
    )
    return [ip for ip, count in failed.items() if count >= threshold]

def detect_scanning(entries: list[dict], threshold: int = 50) -> list[str]:
    """Find IPs hitting many unique paths (directory scanning)."""
    ip_paths: dict[str, set] = {}
    for e in entries:
        ip_paths.setdefault(e["ip"], set()).add(e["path"])
    return [ip for ip, paths in ip_paths.items() if len(paths) >= threshold]
```

**Extension ideas:**

- Add time-window analysis (requests per minute per IP)
- Detect SQL injection patterns in request paths
- Output results in JSON or CSV for SIEM ingestion
- Add GeoIP lookup for suspicious source IPs

---

## Challenge 3: Port Scanner

**Objective:** Build a TCP port scanner that identifies open ports and grabs service banners.

**Key concepts:** TCP three-way handshake, socket programming, concurrency, service identification.

```python
import socket
from concurrent.futures import ThreadPoolExecutor

def scan_port(host: str, port: int, timeout: float = 1.0) -> dict | None:
    try:
        with socket.socket(socket.AF_INET, socket.SOCK_STREAM) as s:
            s.settimeout(timeout)
            s.connect((host, port))
            banner = ""
            try:
                s.send(b"HEAD / HTTP/1.0\r\n\r\n")
                banner = s.recv(1024).decode(errors="replace").strip()
            except (socket.timeout, OSError):
                pass
            return {"port": port, "state": "open", "banner": banner}
    except (socket.timeout, ConnectionRefusedError, OSError):
        return None

def scan_host(host: str, ports: range, max_workers: int = 100) -> list[dict]:
    results = []
    with ThreadPoolExecutor(max_workers=max_workers) as pool:
        futures = {pool.submit(scan_port, host, p): p for p in ports}
        for future in futures:
            result = future.result()
            if result:
                results.append(result)
    return sorted(results, key=lambda r: r["port"])
```

**Extension ideas:**

- Add SYN scan using raw sockets (requires root/admin)
- Implement service version detection from banners
- Add UDP scanning support
- Compare results against known vulnerability databases

---

## Challenge 4: Web Scraper with Security Headers Check

**Objective:** Fetch a web page and analyse its security headers, reporting missing or misconfigured protections.

**Key concepts:** HTTP headers, browser security mechanisms, TLS certificate analysis.

```python
import requests
import socket

SECURITY_HEADERS = {
    "Strict-Transport-Security": "Enforces HTTPS connections",
    "Content-Security-Policy": "Controls resource loading, mitigates XSS",
    "X-Content-Type-Options": "Prevents MIME-type sniffing",
    "X-Frame-Options": "Prevents clickjacking",
    "Referrer-Policy": "Controls referrer information leakage",
    "Permissions-Policy": "Controls browser feature access",
    "X-XSS-Protection": "Legacy XSS filter (deprecated but still checked)",
}

def check_security_headers(url: str) -> dict:
    resp = requests.get(url, timeout=10, allow_redirects=True)
    results = {}
    for header, description in SECURITY_HEADERS.items():
        value = resp.headers.get(header)
        results[header] = {
            "present": value is not None,
            "value": value,
            "description": description,
        }
    return results

def check_tls(url: str) -> dict:
    """Basic TLS information extraction."""
    import ssl
    from urllib.parse import urlparse
    hostname = urlparse(url).hostname
    ctx = ssl.create_default_context()
    with ctx.wrap_socket(
        socket.socket(), server_hostname=hostname
    ) as s:
        s.connect((hostname, 443))
        cert = s.getpeercert()
        return {
            "subject": dict(x[0] for x in cert["subject"]),
            "issuer": dict(x[0] for x in cert["issuer"]),
            "notBefore": cert["notBefore"],
            "notAfter": cert["notAfter"],
            "serialNumber": cert["serialNumber"],
        }
```

---

## Challenge 5: Password Strength Analyser

**Objective:** Evaluate password strength against common criteria and known breached password lists.

**Key concepts:** Password entropy, dictionary attacks, hash lookup, breach databases.

```python
import hashlib
import math
import re
import requests

def calculate_entropy(password: str) -> float:
    charset_size = 0
    if re.search(r"[a-z]", password):
        charset_size += 26
    if re.search(r"[A-Z]", password):
        charset_size += 26
    if re.search(r"\d", password):
        charset_size += 10
    if re.search(r"[^a-zA-Z0-9]", password):
        charset_size += 32
    if charset_size == 0:
        return 0.0
    return len(password) * math.log2(charset_size)

def check_haveibeenpwned(password: str) -> int:
    """Check password against HIBP API using k-anonymity."""
    sha1 = hashlib.sha1(password.encode()).hexdigest().upper()
    prefix, suffix = sha1[:5], sha1[5:]
    resp = requests.get(
        f"https://api.pwnedpasswords.com/range/{prefix}", timeout=5
    )
    for line in resp.text.splitlines():
        hash_suffix, count = line.split(":")
        if hash_suffix == suffix:
            return int(count)
    return 0

def analyse_password(password: str) -> dict:
    entropy = calculate_entropy(password)
    breach_count = check_haveibeenpwned(password)
    return {
        "length": len(password),
        "entropy_bits": round(entropy, 1),
        "strength": (
            "weak" if entropy < 40 else
            "moderate" if entropy < 60 else
            "strong" if entropy < 80 else
            "very strong"
        ),
        "breach_count": breach_count,
        "breached": breach_count > 0,
    }
```

---

## Challenge 6: YARA Rule Scanner

**Objective:** Build a file scanner that matches files against YARA rules to detect malware signatures and suspicious patterns.

**Key concepts:** Signature-based detection, pattern matching, IOC scanning, malware triage.

```python
import yara
import os

def compile_rules(rules_dir: str) -> yara.Rules:
    """Compile all .yar files in a directory."""
    rule_files = {}
    for fname in os.listdir(rules_dir):
        if fname.endswith((".yar", ".yara")):
            namespace = fname.rsplit(".", 1)[0]
            rule_files[namespace] = os.path.join(rules_dir, fname)
    return yara.compile(filepaths=rule_files)

def scan_file(rules: yara.Rules, filepath: str) -> list[dict]:
    matches = rules.match(filepath)
    return [
        {
            "rule": m.rule,
            "namespace": m.namespace,
            "tags": m.tags,
            "strings": [
                {"offset": s[0], "identifier": s[1], "data": s[2]}
                for s in m.strings
            ],
        }
        for m in matches
    ]

def scan_directory(rules: yara.Rules, dirpath: str) -> dict:
    results = {}
    for root, _, files in os.walk(dirpath):
        for fname in files:
            path = os.path.join(root, fname)
            matches = scan_file(rules, path)
            if matches:
                results[path] = matches
    return results
```

**Example YARA rule:**

```yara
rule SuspiciousPowerShell {
    meta:
        description = "Detect obfuscated PowerShell execution"
        severity = "medium"
    strings:
        $enc_cmd = "-EncodedCommand" ascii nocase
        $bypass = "-ExecutionPolicy Bypass" ascii nocase
        $hidden = "-WindowStyle Hidden" ascii nocase
        $download = "DownloadString" ascii nocase
        $iex = "IEX" ascii nocase
    condition:
        2 of them
}
```

---

## Challenge 7: Metadata Extractor

**Objective:** Extract metadata from files (images, PDFs, Office documents) that could reveal sensitive information.

**Key concepts:** EXIF data, document properties, information leakage, OSINT.

```python
from PIL import Image
from PIL.ExifTags import TAGS, GPSTAGS

def extract_image_metadata(filepath: str) -> dict:
    img = Image.open(filepath)
    exif_data = img._getexif()
    if not exif_data:
        return {"format": img.format, "size": img.size, "exif": None}

    metadata = {}
    for tag_id, value in exif_data.items():
        tag = TAGS.get(tag_id, tag_id)
        if tag == "GPSInfo":
            gps = {}
            for gps_id, gps_val in value.items():
                gps[GPSTAGS.get(gps_id, gps_id)] = gps_val
            metadata[tag] = gps
        else:
            metadata[tag] = str(value)
    return {
        "format": img.format,
        "size": img.size,
        "exif": metadata,
    }

# Security-relevant metadata fields to flag:
# - GPS coordinates (location disclosure)
# - Camera make/model (device identification)
# - Software (tool fingerprinting)
# - Author/Creator (identity disclosure)
# - Timestamps (activity timeline)
# - Embedded thumbnails (may contain unredacted original)
```

**Extension ideas:**

- Add PDF metadata extraction (author, creator, timestamps)
- Extract Office document properties (last saved by, revision count)
- Strip metadata from files before sharing (sanitisation tool)
- Scan a directory and report all files with GPS coordinates

---

## Challenge 8: Network Packet Analyser

**Objective:** Capture and analyse network packets, extracting key information for security monitoring.

**Key concepts:** Packet structure, protocol dissection, traffic analysis, anomaly detection.

```python
from scapy.all import sniff, IP, TCP, DNS, DNSQR, Raw

def packet_callback(pkt):
    if IP in pkt:
        src = pkt[IP].src
        dst = pkt[IP].dst

        if TCP in pkt:
            sport = pkt[TCP].sport
            dport = pkt[TCP].dport
            flags = pkt[TCP].flags
            print(f"TCP {src}:{sport} -> {dst}:{dport} [{flags}]")

            # Detect SYN scan (SYN without ACK)
            if flags == "S":
                print(f"  [!] Possible SYN scan from {src}")

        if DNS in pkt and pkt.haslayer(DNSQR):
            query = pkt[DNSQR].qname.decode()
            print(f"DNS query: {src} -> {query}")

            # Detect DNS tunneling (unusually long subdomains)
            if len(query) > 60:
                print(f"  [!] Possible DNS tunneling: {query}")

# Capture 100 packets on default interface
# sniff(count=100, prn=packet_callback, store=False)
```

**Extension ideas:**

- Detect ARP spoofing by tracking MAC-IP mappings
- Extract and reassemble HTTP requests/responses
- Identify cleartext credentials in unencrypted protocols
- Build a simple IDS rule engine with configurable signatures

!!! info "External Resources"
    - [Automate the Boring Stuff with Python](https://automatetheboringstuff.com/) (Al Sweigart)
    - [Black Hat Python](https://nostarch.com/black-hat-python2E) (No Starch Press)
    - [YARA Documentation](https://yara.readthedocs.io/) (VirusTotal)
    - [Scapy Documentation](https://scapy.readthedocs.io/) (Scapy)
