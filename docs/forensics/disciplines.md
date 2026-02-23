# Forensic Disciplines

## Network Forensics

Network forensics involves the capture, recording, and analysis of network traffic to detect intrusions, reconstruct events, and gather evidence.

### DNS Logs

- DNS query logs record domain lookups by internal hosts - useful for identifying C2 callbacks, DGA domains, and data exfiltration
- **Passive DNS** aggregates DNS resolution data from sensors, building a historical record of domain-to-IP mappings without actively querying DNS servers
- Passive DNS enables retrospective analysis - determine what IP a domain resolved to at a specific point in time
- DNS log sources: recursive resolver logs, firewall DNS inspection, network tap on port 53, DNS sinkhole logs

### Netflow

- Netflow provides metadata about network conversations: source/destination IP, ports, protocol, byte count, packet count, timestamps
- Does not capture payload content - useful for identifying patterns, anomalous volumes, and lateral movement
- **Sampling rate** is a critical consideration: a 1:100 sampling rate means only 1 in 100 packets is used to generate flow records, reducing accuracy for low-volume connections
- Common formats: NetFlow v5/v9 (Cisco), IPFIX (IETF standard), sFlow (sampled by default)
- Retention periods for netflow are typically longer than full packet capture due to significantly smaller storage requirements

### Packet Capture

- Full packet capture (PCAP) preserves complete network traffic including headers and payload
- Storage-intensive: 1 Gbps link generates approximately 10 TB per day at full capture
- Tools: tcpdump, Wireshark/tshark, Zeek (Bro), Moloch/Arkime for indexed capture
- Capture points: network TAPs (passive, full-duplex), SPAN/mirror ports (may drop packets under load)

!!! info "External Resources"
    - [Passive DNS Replication](https://www.first.org/resources/papers/conference2005/passive-dns.pdf) (FIRST)
    - [Network Forensics with Zeek](https://docs.zeek.org/en/master/) (Zeek Documentation)
    - [Cisco NetFlow Overview](https://www.cisco.com/c/en/us/products/ios-nx-os-software/ios-netflow/index.html) (Cisco)

## Disk Forensics

### Imaging

- Always use write blockers (hardware preferred) before connecting evidence media
- Create a bit-for-bit forensic image, not a logical copy
- Verify image integrity with cryptographic hashes (SHA-256) at creation and before each analysis session
- Tools: **FTK Imager** (free, supports E01/raw), **EnCase** (commercial, E01 format), **dc3dd** (command-line with hashing)

### Filesystems

| Filesystem | OS | Key Forensic Features |
|:---|:---|:---|
| NTFS | Windows | MFT, $LogFile, $UsnJrnl, Alternate Data Streams, Volume Shadow Copies |
| ext2 | Linux (legacy) | No journaling - deleted inodes retain more data |
| ext3 | Linux | Journal recovery possible; inode timestamps |
| ext4 | Linux | Extents, nanosecond timestamps, journal checksums |
| APFS | macOS/iOS | Snapshots, clones, native encryption, nanosecond timestamps |
| HFS+ | macOS (legacy) | Catalog file, journal, resource forks |

- **NTFS Master File Table (MFT)**: Contains metadata for every file - timestamps, size, permissions, data location. Small files (< ~700 bytes) are stored resident within the MFT entry itself.
- **NTFS $UsnJrnl**: Change journal tracking file creation, deletion, modification, and rename operations
- **Volume Shadow Copies** (Windows): Point-in-time snapshots that may contain deleted or modified files

### Data Carving

- Recovers files from unallocated space based on file signatures (headers/footers), independent of filesystem metadata
- Effective even when file system structures are destroyed or overwritten
- Tools: **Scalpel**, **PhotoRec**, **Foremost**
- Limitations: fragmented files may not carve correctly; false positives are common with certain file types

### Timeline Tools

- **plaso/log2timeline**: Aggregates timestamps from multiple artifact sources (filesystem, registry, event logs, browser history) into a unified super-timeline
- Super-timelines enable correlation of events across disparate data sources

!!! info "External Resources"
    - [Sleuth Kit / Autopsy Documentation](https://www.sleuthkit.org/autopsy/docs.php) (Sleuth Kit)
    - [NTFS Forensic Analysis](https://www.sans.org/white-papers/36842/) (SANS Institute)
    - [plaso - log2timeline Documentation](https://plaso.readthedocs.io/en/latest/) (plaso Project)

## Memory Forensics

### Acquisition

- Memory acquisition must occur before system shutdown - RAM content is lost on power off
- **Acquisition footprint**: Any tool used to capture memory runs in memory itself, inevitably overwriting a small portion of evidence. Minimize footprint by using lightweight acquisition tools.
- **Memory smear**: Memory state changes during the acquisition process because the system continues executing. A 16 GB capture may take 30+ seconds, during which processes execute, network states change, and data structures are modified.
- **Hibernation files** (`hiberfil.sys` on Windows): Contain a compressed copy of RAM at hibernation time - can be analyzed as a memory image
- **Crash dumps**: Windows crash dumps (`MEMORY.DMP`) contain full or partial memory state at the time of a BSOD
- **Pagefile/swap**: `pagefile.sys` (Windows) and swap partitions (Linux) contain memory pages evicted from RAM - may hold evidence from processes no longer in memory

### Virtual vs Physical Memory

- **Physical memory**: Actual RAM chips; memory acquisition tools capture physical memory
- **Virtual memory**: Abstraction provided by the OS; each process has its own virtual address space mapped to physical frames via page tables
- The OS memory manager handles translation between virtual and physical addresses
- Forensic tools must reconstruct virtual address spaces from page tables to analyze process memory

### Memory Structures

- **Process list**: Linked list of `EPROCESS` structures (Windows) or `task_struct` (Linux); rootkits may unlink entries to hide processes
- **Loaded modules**: DLLs (Windows) and shared objects (Linux) loaded into process address space
- **Network connections**: Socket structures containing remote IP, port, and connection state
- **Registry hives**: Windows registry is partially loaded into memory; may contain more recent data than on-disk hives
- **Handles and tokens**: Open file handles, security tokens, and access permissions for each process

### Kernel Space vs User Space

- **Kernel space**: Operating system code, drivers, and kernel data structures. Contains the ground truth for system state. Rootkits target kernel space to manipulate what user-space tools report.
- **User space**: Application code and data. Each process is isolated in its own virtual address space.
- Memory forensics tools operate on the physical memory image and can inspect both spaces, bypassing rootkit hiding techniques

### Memory Forensics Tools

| Tool | Type | Key Capabilities |
|:---|:---|:---|
| **Volatility** (2/3) | Open source | Plugin-based framework; process listing, DLL analysis, network connections, malware detection, timeline |
| **Rekall** | Open source | Similar to Volatility; integrated into GRR Rapid Response for remote acquisition |
| **GRR Rapid Response** | Open source | Remote live forensics at scale; uses Rekall for memory analysis |
| **WinDbg** | Microsoft | Kernel debugging; analysis of crash dumps and live kernel memory |
| **LiME** | Open source | Linux Memory Extractor; kernel module for Linux/Android memory acquisition |

!!! info "External Resources"
    - [Volatility 3 Documentation](https://volatility3.readthedocs.io/en/latest/) (Volatility Foundation)
    - [The Art of Memory Forensics](https://www.wiley.com/en-us/The+Art+of+Memory+Forensics-p-9781118825099) (Wiley)
    - [GRR Rapid Response](https://grr-doc.readthedocs.io/en/latest/) (Google)

## Mobile Forensics

Mobile device forensics presents unique challenges due to device encryption, proprietary operating systems, and rapid hardware evolution.

**Android vs iOS:**

| Aspect | Android | iOS |
|:---|:---|:---|
| File system | ext4, F2FS (varies by vendor) | APFS (encrypted by default) |
| Encryption | Full-disk or file-based (varies) | File-based (Data Protection API) |
| Acquisition | ADB, JTAG, chip-off, bootloader exploits | iTunes backup, checkm8 (hardware), GrayKey |
| Root/Jailbreak | Root via unlocked bootloader or exploit | Jailbreak via software exploit chain |
| Fragmentation | High - many vendors, OS versions, skins | Low - Apple controls hardware and software |

**Jailbreaking and Rooting:**

- Jailbreaking (iOS) and rooting (Android) bypass manufacturer security restrictions, granting full filesystem access
- Required for full physical acquisition on many devices
- Introduces risk: may modify device state, potentially altering evidence
- Must be documented as part of forensic procedure and weighed against evidence integrity requirements

**Acquisition Levels:**

- **Manual**: Photographs and screen recordings of device content. Least invasive but incomplete.
- **Logical**: Extracts files and data accessible through the OS API (e.g., iTunes backup). Does not include deleted data.
- **File system**: Full file system extraction including databases, caches, and system files. Requires root/jailbreak.
- **Physical**: Bit-for-bit image of flash storage. Most complete but most difficult to obtain on modern encrypted devices.

!!! info "External Resources"
    - [NIST Mobile Device Forensics Guidelines](https://csrc.nist.gov/publications/detail/sp/800-101/rev-1/final) (NIST SP 800-101)
    - [Cellebrite UFED Documentation](https://cellebrite.com/en/ufed/) (Cellebrite)
    - [SANS Mobile Forensics Poster](https://www.sans.org/posters/smartphone-forensics/) (SANS Institute)

## Cloud Forensics

Cloud forensics introduces complications absent from traditional on-premises investigations: shared infrastructure, limited physical access, distributed data storage, and provider-specific logging mechanisms.

- **Shared responsibility**: The cloud provider manages infrastructure; the customer manages data and application-level logging. Forensic evidence depends on which layer is relevant.
- **Acquisition challenges**: Traditional disk imaging is not possible. Evidence collection relies on provider APIs, snapshots, and log exports.
- **Log sources**: Cloud provider logs (CloudTrail, Activity Log, Cloud Audit Logs), VPC Flow Logs, access logs, application-level logs
- **Ephemeral resources**: Auto-scaling groups, serverless functions, and containers may be destroyed before evidence can be collected. Enable persistent logging before incidents occur.
- **Multi-tenancy**: Investigators cannot access other tenants' data; evidence is limited to the organization's own resources and logs
- **Legal jurisdictional issues**: Data may be stored across multiple regions and subject to different legal frameworks

!!! info "External Resources"
    - [CSA Cloud Forensics Guidance](https://cloudsecurityalliance.org/research/topics/cloud-forensics/) (Cloud Security Alliance)
    - [AWS Security Incident Response Guide](https://docs.aws.amazon.com/whitepapers/latest/aws-security-incident-response-guide/welcome.html) (AWS)
    - [NIST Cloud Computing Forensic Science Challenges](https://csrc.nist.gov/publications/detail/nistir/8006/final) (NIST IR 8006)

## Forensic Tools Reference

| Tool | Category | License | Primary Use |
|:---|:---|:---|:---|
| FTK Imager | Disk imaging | Free | Forensic image creation (E01, raw), mounting, preview |
| EnCase | Disk forensics | Commercial | Full forensic examination suite, court-accepted reporting |
| Autopsy / Sleuth Kit | Disk forensics | Open source | File system analysis, timeline, keyword search, carving |
| plaso (log2timeline) | Timeline | Open source | Super-timeline generation from multiple artifact sources |
| Volatility | Memory | Open source | Memory image analysis via extensible plugin architecture |
| Rekall | Memory | Open source | Memory forensics framework, integrated with GRR |
| WinDbg | Memory / Debug | Free | Windows kernel debugging, crash dump analysis |
| LiME | Memory acquisition | Open source | Linux/Android RAM capture via kernel module |
| Wireshark | Network | Open source | Packet capture and protocol analysis |
| Zeek (Bro) | Network | Open source | Network traffic analysis and logging framework |
| Arkime (Moloch) | Network | Open source | Indexed full packet capture and search |
| Cellebrite UFED | Mobile | Commercial | Mobile device acquisition and analysis |
| GRR Rapid Response | Remote forensics | Open source | Enterprise-scale remote live forensics |
| KAPE | Triage | Free | Rapid artifact collection and processing |

!!! info "External Resources"
    - [SANS Digital Forensics Tools](https://www.sans.org/tools/) (SANS Institute)
    - [NIST Computer Forensics Tool Testing Program](https://www.nist.gov/itl/ssd/software-quality-group/computer-forensics-tool-testing-program-cftt) (NIST CFTT)
    - [DFIR Training Tool List](https://www.dfir.training/tools) (DFIR Training)
