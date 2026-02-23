# Coding Fundamentals

## Data Structures for Security Engineers

Understanding core data structures helps when building security tooling, analysing logs at scale, and understanding algorithm behaviour in security contexts.

| Data Structure | Properties | Security use cases |
|----------------|-----------|-------------------|
| **Hash table / dict** | O(1) average lookup, insert, delete | IP reputation lookup, IOC matching, deduplication of alerts, caching DNS responses |
| **Set** | O(1) membership test, union/intersection | Allowlist/blocklist membership, comparing two IOC feeds, finding unique IPs in logs |
| **Array / list** | O(1) indexed access, O(n) search | Log entries, packet buffers, ordered event sequences |
| **Linked list** | O(1) insert/delete at head | LRU caches (eviction policy), connection tracking |
| **Stack (LIFO)** | O(1) push/pop | Call stack analysis in reverse engineering, bracket/tag matching in parsers |
| **Queue (FIFO)** | O(1) enqueue/dequeue | Event processing pipelines, rate limiting (sliding window), BFS traversal |
| **Tree (binary, B-tree)** | O(log n) search, insert, delete | File system indexing, database indexes, certificate chains (tree of trust) |
| **Trie (prefix tree)** | O(k) lookup where k is key length | IP prefix matching (routing tables), domain name matching, autocomplete for commands |
| **Bloom filter** | Probabilistic membership (no false negatives) | Fast malware hash pre-check, web filter URL lookup, CDN cache checking |
| **Graph** | Nodes + edges; various traversal algorithms | Network topology mapping, attack path analysis, trust relationship modelling, RBAC policy graphs |
| **Heap / priority queue** | O(1) min/max, O(log n) insert/extract | Alert prioritisation, top-N analysis, Dijkstra's algorithm for network paths |

---

## Algorithms and Complexity (Big O)

Big O notation describes how an algorithm's resource usage (time or space) scales with input size.

**Common complexity classes:**

| Big O | Name | Example | Scale at n=1M |
|-------|------|---------|---------------|
| **O(1)** | Constant | Hash table lookup, array index | 1 operation |
| **O(log n)** | Logarithmic | Binary search, balanced tree lookup | ~20 operations |
| **O(n)** | Linear | Scanning a log file, linear search | 1M operations |
| **O(n log n)** | Linearithmic | Merge sort, heap sort | ~20M operations |
| **O(n^2)** | Quadratic | Nested loop comparison, naive string matching | 1T operations (infeasible) |
| **O(2^n)** | Exponential | Brute-force subset enumeration | Infeasible |

**Security-relevant algorithm considerations:**

| Context | Algorithm choice | Why it matters |
|---------|-----------------|---------------|
| **Hash comparison** | Constant-time comparison (e.g., `hmac.compare_digest`) | Prevents timing side-channel attacks |
| **Password hashing** | bcrypt, scrypt, Argon2 (deliberately slow) | Resistance to brute force; tunable cost factor |
| **Log searching** | Indexed search (B-tree, inverted index) vs linear scan | Performance at scale; SIEM query optimisation |
| **Pattern matching** | Aho-Corasick (multi-pattern) | YARA-style IOC scanning across large datasets |
| **Regular expressions** | Avoid catastrophic backtracking (ReDoS) | Untrusted input + bad regex = denial of service |
| **Sorting alerts** | O(n log n) sort + priority queue | Triage thousands of alerts efficiently |

---

## Regular Expressions

Regular expressions (regex) are essential for log parsing, pattern matching, and data extraction in security operations.

**Core syntax:**

| Pattern | Meaning | Example |
|---------|---------|---------|
| `.` | Any single character | `a.c` matches `abc`, `a1c` |
| `*` | Zero or more of preceding | `ab*c` matches `ac`, `abc`, `abbc` |
| `+` | One or more of preceding | `ab+c` matches `abc`, `abbc` but not `ac` |
| `?` | Zero or one of preceding | `colou?r` matches `color`, `colour` |
| `^` | Start of line | `^ERROR` matches lines starting with ERROR |
| `$` | End of line | `\.exe$` matches strings ending in .exe |
| `[]` | Character class | `[0-9]` matches any digit |
| `[^]` | Negated character class | `[^a-z]` matches non-lowercase |
| `\d` | Digit (same as `[0-9]`) | `\d{3}` matches three digits |
| `\w` | Word character `[a-zA-Z0-9_]` | `\w+` matches one or more word characters |
| `\s` | Whitespace | `\s+` matches one or more spaces/tabs |
| `\b` | Word boundary | `\broot\b` matches "root" but not "rooted" |
| `(...)` | Capture group | `(\d+\.\d+\.\d+\.\d+)` captures an IPv4 address |
| `(?:...)` | Non-capturing group | Groups without capturing |
| `{n,m}` | Repetition range | `\d{1,5}` matches 1 to 5 digits |
| `\|` | Alternation (OR) | `error\|warning\|critical` matches any of the three |

**Security-relevant regex patterns:**

```
# IPv4 address
\b\d{1,3}\.\d{1,3}\.\d{1,3}\.\d{1,3}\b

# Email address (basic)
[\w.+-]+@[\w-]+\.[\w.-]+

# URL extraction
https?://[^\s"'<>]+

# Base64-encoded string (long block)
[A-Za-z0-9+/]{20,}={0,2}

# Common log format - extract IP, timestamp, request
^(\S+) \S+ \S+ \[([^\]]+)\] "(\S+ \S+ \S+)" (\d{3}) (\d+|-)

# AWS access key
(?:AKIA|ABIA|ACCA|ASIA)[0-9A-Z]{16}

# JWT token
eyJ[A-Za-z0-9_-]+\.eyJ[A-Za-z0-9_-]+\.[A-Za-z0-9_-]+
```

!!! warning "ReDoS - Regular Expression Denial of Service"
    Avoid patterns with nested quantifiers on overlapping character classes when processing untrusted input. Example of a vulnerable pattern: `(a+)+b` - exponential backtracking on input like `aaaaaaaaaaX`. Use atomic groups or possessive quantifiers where supported, or set timeout limits.

---

## Python Essentials

Python is the most common language for security scripting, automation, and tool development.

**Key standard library modules for security:**

| Module | Purpose | Example use |
|--------|---------|-------------|
| `hashlib` | Cryptographic hashing | File integrity checking, hash comparison |
| `hmac` | HMAC computation and constant-time comparison | API signature verification |
| `secrets` | Cryptographically secure random numbers | Token generation, password creation |
| `ssl` | TLS/SSL wrapper | Certificate validation, secure connections |
| `socket` | Low-level networking | Port scanning, custom protocol clients |
| `subprocess` | Run external commands | Invoking nmap, running system commands |
| `re` | Regular expressions | Log parsing, IOC extraction |
| `json` / `csv` | Data formats | Parsing API responses, log files |
| `struct` | Binary data packing/unpacking | Parsing binary protocols, PE headers |
| `ipaddress` | IP address manipulation | CIDR calculations, network membership checks |
| `logging` | Structured logging | Audit trails for security tools |
| `argparse` | CLI argument parsing | Building command-line security tools |
| `pathlib` | Filesystem paths | Safe path manipulation (avoiding traversal) |
| `base64` | Encoding/decoding | Decoding obfuscated payloads |

**Common patterns:**

```python
# Constant-time hash comparison (prevents timing attacks)
import hmac
hmac.compare_digest(hash_a, hash_b)

# Secure random token generation
import secrets
token = secrets.token_hex(32)  # 64-character hex string
api_key = secrets.token_urlsafe(32)  # URL-safe base64

# File hashing
import hashlib
def hash_file(path, algo="sha256"):
    h = hashlib.new(algo)
    with open(path, "rb") as f:
        for chunk in iter(lambda: f.read(8192), b""):
            h.update(chunk)
    return h.hexdigest()

# IP address operations
from ipaddress import ip_address, ip_network
addr = ip_address("10.0.1.5")
net = ip_network("10.0.0.0/16")
print(addr in net)  # True
print(addr.is_private)  # True
```

---

## Scripting for Security Automation

**Common automation tasks and approaches:**

| Task | Approach | Key libraries |
|------|----------|--------------|
| **Log parsing** | Read line by line, extract fields with regex or structured parsing | `re`, `json`, `csv`, `gzip` |
| **API interaction** | HTTP requests to security tools (SIEM, threat intel, ticketing) | `requests`, `httpx` |
| **Network scanning** | Socket connections, banner grabbing | `socket`, `scapy`, `nmap` (python-nmap) |
| **File analysis** | Hash calculation, metadata extraction, string search | `hashlib`, `pefile`, `yara-python` |
| **Report generation** | Aggregate findings into structured output | `jinja2`, `csv`, `json` |
| **Alert enrichment** | Query multiple sources to add context to an alert | `requests`, `asyncio` for parallel queries |
| **Configuration auditing** | Parse configs and check against baselines | `yaml`, `json`, `configparser` |

**Script structure best practices:**

```python
#!/usr/bin/env python3
"""Brief description of what this tool does."""

import argparse
import logging
import sys

logging.basicConfig(
    level=logging.INFO,
    format="%(asctime)s %(levelname)s %(message)s"
)
log = logging.getLogger(__name__)


def main():
    parser = argparse.ArgumentParser(description=__doc__)
    parser.add_argument("target", help="Target to analyse")
    parser.add_argument("-o", "--output", help="Output file path")
    parser.add_argument("-v", "--verbose", action="store_true")
    args = parser.parse_args()

    if args.verbose:
        logging.getLogger().setLevel(logging.DEBUG)

    log.info("Starting analysis of %s", args.target)
    # Tool logic here


if __name__ == "__main__":
    main()
```

**Error handling patterns for security tools:**

- Fail securely: deny access on error rather than allowing it
- Log all errors with context (timestamp, input, error type)
- Never expose stack traces or internal paths to users
- Validate all external input before processing
- Set timeouts on network operations to prevent hanging
- Use `try/except` with specific exception types, not bare `except`

!!! info "External Resources"
    - [Python Official Tutorial](https://docs.python.org/3/tutorial/) (Python.org)
    - [Automate the Boring Stuff with Python](https://automatetheboringstuff.com/) (Al Sweigart)
    - [Black Hat Python](https://nostarch.com/black-hat-python2E) (No Starch Press)
    - [Violent Python](https://github.com/tanc7/Violent-Python3) (Reference)
