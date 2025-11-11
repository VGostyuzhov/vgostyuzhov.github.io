I want to create an mkdocs site, I'm going to use as study material to prepare for the Infrastructure Security Engineer and Cloud Security Engineer roles.

I want to write concise material to cover all the relevant topics, but which is also expressive and covers the majority of the info I need to learn.

By default I don't need the basics of the topic - just link some good online reading material instead in an "info" note.

* Don't commit CLAUDE.md to the git. 
* Don't create excessive comments.
* Don't create all the content. I want to work on it step-by-step and will provide explicit instructions.

Let's start with organizing the file structure. Here is the first iteration on the list of topics:

## 1. Learning & Career Development

### Learning Tips
- Learning How To Learn course
- Track concepts ("To learn", "Revising", "Done")
- How to review concepts (spaced-repetition, recall)
- Target your learning
- Identify what you need to work on
- Mental health

### Interviewing Tips
- Interview questions (vague, ask questions, conversation, depth/breadth)
- Show comprehension (repeat question, clarify)
- State your assumptions
- When asked a question you're not sure of
- Say what you are thinking
- Reduce cognitive load (notes, pseudocode, tests)
- Prepare (checklist, questions for interviewer, snacks)
- Do practice interviews

## 2. Networking & Protocols

### Core Networking
- OSI Model
- Firewalls
- NAT
- DNS (requests, exfiltration, configs)
- ARP
- DHCP
- BGP
- Broadcast domains and collision domains
- CAM table overflow

### Network Protocols
- TCP/UDP
- ICMP
- HTTP/S
- SSL/TLS (handshakes, encryption, signing, CAs, vulnerabilities)
- Mail (SMTP, IMAP, POP3)
- SSH
- Telnet
- IRC
- FTP/SFTP
- RPC
- Service ports

### Network Security & Tools
- Network traffic tools (Wireshark, tcpdump, Burp Suite)
- Traceroute
- Nmap
- Intercepts (Person-in-the-Middle)
- VPN
- Tor
- Proxy
- HTTP Header
- HTTP Response Header (Status Codes)
- UDP Header
- Root stores

## 3. Web Application Security

### Web Security Fundamentals
- Same origin policy
- CORS
- HSTS
- Certificate transparency
- HTTP Public Key Pinning (HPKP)
- Cookies (httponly)

### Web Vulnerabilities
- CSRF
- XSS (Reflected, Persistent, DOM based)
- SQLi
- Directory traversal
- Local file inclusion
- Remote file inclusion
- SSRF
- Malicious redirects

### Web Technologies & Tools
- POST/GET methods
- APIs
- Beefhook
- User agents
- Browser extension take-overs
- Web vulnerability scanners
- SQLmap

## 4. Infrastructure & Cloud Security

### Virtualization & Cloud
- Hypervisors
- Hyperjacking
- Containers, VMs, clusters
- Escaping techniques
- Lateral movement and privilege escalation techniques (Cloud Service Accounts)
- Site isolation
- Side-channel attacks (Spectre, Meltdown)
- BeyondCorp
- Log4j vulnerability
- Cloud security models (Shared responsibility)
- IAM (Identity and Access Management)
- Cloud storage security (S3 buckets, ACLs)
- Serverless security
- Kubernetes security
- Container security (Docker, registry scanning)
- Cloud monitoring and logging
- Cloud compliance frameworks

## 5. Operating Systems & System Security

### OS Implementation
- Privilege escalation techniques and prevention
- Buffer Overflows
- Directory traversal (prevention)
- Remote Code Execution / getting shells
- Local databases (sqlite)

### Platform-Specific Security
- Windows (Registry, Active Directory, SMB, Samba, ROP)
- *nix (SELinux, Kernel/userspace, MAC vs DAC, /proc, /tmp, /shadow, LDAP)
- macOS (Gotofail, MacSweeper)

### Security Mitigations
- Patching
- Data Execution Prevention
- Address space layout randomization
- Principle of least privilege
- Code signing
- Compiler security features
- Encryption
- Mandatory Access Controls (MACs)
- "Insecure by exception"
- Do not blame the user

## 6. Cryptography, Authentication & Identity

### Cryptographic Fundamentals
- Encryption vs Encoding vs Hashing vs Obfuscation vs Signing
- Encryption standards + implementations (RSA, AES, ECC, ChaCha20/Salsa20)
- Asymmetric vs symmetric (perfect forward secrecy)
- Ciphers (Block vs stream, modes of operation, AES-GCM)
- Integrity and authenticity primitives (Hashing functions, MACs, HMAC)
- Entropy (PRNG)

### Authentication Systems
- Certificates
- Trusted Platform Module (TPM)
- OAuth (Bearer tokens)
- Auth Cookies
- Sessions
- Auth systems (SAML v2.0, OpenID, Kerberos)
- Biometrics
- Password management
- U2F/FIDO
- Multi-factor auth methods

### Identity Management
- Access Control Lists (ACLs)
- Service accounts vs User accounts
- Impersonation (JWT)
- Federated identity

## 7. Malware & Threat Analysis

### Malware Analysis
- Interesting malware (Conficker, Stuxnet, WannaCry)
- Malware features (RCE, Domain-flux, Evasion, Process hollowing, RATs)
- Decompiling/reversing (Obfuscation, IDA Pro, Ghidra)
- Static/dynamic analysis (VirusTotal, Reverse.it, Hybrid-Analysis)

### Attack Vectors & Exploits
- Three ways to attack (Social, Physical, Network)
- Exploit Kits and drive-by download attacks
- Remote Control (RCE, Bind shell, Reverse shell)
- Spoofing (Email, IP, MAC, ARP, Biometric)
- Tools (Metasploit, ExploitDB, Shodan, Hak5)

### Attack Lifecycle (MITRE ATT&CK)
- Reconnaissance
- Resource development
- Initial access
- Execution
- Persistence
- Privilege escalation
- Defense evasion
- Credential access
- Discovery
- Lateral movement
- Collection
- Exfiltration
- Command and control
- Impact

## 8. Threat Modeling & Risk Assessment

### Threat Modeling Frameworks
- Threat Matrix
- Trust Boundaries
- Security Controls
- STRIDE framework
- MITRE ATT&CK framework
- PASTA (Process for Attack Simulation and Threat Analysis)
- DREAD risk assessment model

### Risk Management
- Risk assessment methodologies
- Vulnerability scoring (CVSS)
- Business impact analysis
- Risk treatment strategies
- Security metrics and KPIs

## 9. Detection & Monitoring

### Detection Systems
- IDS (Signature vs behavior, Snort, OSSEC)
- SIEM
- IOC (Indicators of Compromise)
- Signatures (Host-based, Network)
- Anomaly/Behavior-based detection
- Firewall rules (Brute force, port scanning)

### Monitoring & Analysis
- Things that create signals (Honeypots)
- Things that triage signals (SIEM)
- Things that will alert a human
- Honey pots (Canary tokens)
- Things to know about attackers
- Logs to look at (DNS, HTTP, traffic, execution)
- Detection-related tools (Splunk, Wireshark, Zeek, etc.)

## 10. Digital Forensics

### Forensic Fundamentals
- Evidence volatility
- Chain of custody

### Forensic Disciplines
- Network forensics (DNS logs, Netflow)
- Disk forensics (Imaging, filesystems, logs, data recovery, tools)
- Memory forensics (Acquisition, virtual vs physical, structures, tools)
- Mobile forensics (Jailbreaking, Android vs. iPhone)
- Anti-forensics (Timestomping, data wiping, steganography)

## 11. Incident Response

### Incident Management
- Privacy incidents vs information security incidents
- Know when to talk to legal, users, managers
- Run a scenario
- Good practices for running incidents (delegation, communication, risk)
- Important things to know (root cause, attack stages, timeline)
- Response models (SANS PICERL, Google IMAG)

## 12. Programming & Technical Skills

### Coding Fundamentals
- The basics (conditions, loops, dictionaries, lists, strings)
- Data structures (Dictionaries, arrays, stacks, SQL)
- Sorting (Quicksort, merge sort)
- Searching (Binary vs linear)
- Big O (space and time)
- Regular expressions
- Recursion
- Python (comprehensions, generators, slicing, types)

### Security-Themed Coding Challenges
- Ciphers/encryption algorithms
- Parse arbitrary logs
- Web scrapers
- Port scanners
- Botnets
- Password brute-forcer
- Scrape metadata from PDFs
- Recover deleted items
- Malware signatures

## 13. Compliance & Governance

### Regulatory Frameworks
- GDPR (General Data Protection Regulation)
- HIPAA (Health Insurance Portability and Accountability Act)
- PCI DSS (Payment Card Industry Data Security Standard)
- SOX (Sarbanes-Oxley Act)
- NIST Cybersecurity Framework
- ISO 27001/27002
- CIS Controls

### Security Governance
- Security policies and procedures
- Security awareness training
- Vendor risk management
- Security architecture review
- Change management
- Business continuity planning
- Disaster recovery planning

## 14. Emerging Technologies & Threats

### Modern Attack Techniques
- Supply chain attacks
- Living off the land (LOTL) techniques
- Fileless malware
- AI/ML-based attacks
- IoT security threats
- 5G security considerations

### Zero Trust Architecture
- Zero Trust principles
- Micro-segmentation
- Continuous verification
- Least privilege access
- Device trust
- Network trust

### DevSecOps
- Security in CI/CD pipelines
- Infrastructure as Code (IaC) security
- Container security in DevOps
- Security testing automation
- Static Application Security Testing (SAST)
- Dynamic Application Security Testing (DAST)
- Interactive Application Security Testing (IAST)