# Forensic Fundamentals

## Evidence Volatility & Order of Collection

Digital evidence varies substantially in persistence. The Order of Volatility, as defined in RFC 3227, dictates that the most transient data sources must be collected first. Failure to follow this order results in irrecoverable data loss - once volatile memory is overwritten or a network connection terminates, the associated evidence ceases to exist.

**Order of Volatility (most volatile first):**

1. **CPU registers and cache** - nanoseconds of persistence; lost immediately on context switch
2. **Routing tables, ARP cache, process tables, kernel statistics** - maintained in memory, lost on reboot
3. **System memory (RAM)** - contains running processes, open network connections, decrypted data, and loaded malware; lost on power off
4. **Temporary file systems** - `/tmp`, swap space, pagefile; may survive reboot but are frequently cleared
5. **Disk storage** - hard drives, SSDs; persist across reboots but may be overwritten by normal system operation
6. **Remote logging and monitoring data** - SIEM entries, syslog servers, netflow collectors; persist until retention policies expire
7. **Archival media** - backup tapes, optical media; most durable but least current

**Volatility Categories by Evidence Type:**

| Category | Examples | Typical Persistence | Collection Priority |
|:---|:---|:---|:---|
| Network state | Active connections, DNS cache, ARP tables | Seconds to minutes | Immediate |
| Memory | Running processes, encryption keys, clipboard | Until power loss | Within minutes |
| Ephemeral disk | Temp files, swap, browser cache | Hours to days | Within hours |
| Persistent disk | File system, registry, event logs | Weeks to years | Within days |
| External logs | SIEM, centralized syslog, cloud logs | Per retention policy | As available |

**Practical Considerations:**

- Always collect memory before disk - a disk imaging tool running on a live system modifies memory
- Network connections and routing state should be captured before any system interaction
- On virtual machines, snapshots can preserve both memory and disk state simultaneously
- SSD TRIM operations actively destroy deleted data, making timely acquisition critical
- Cloud environments introduce additional complexity as evidence may span multiple jurisdictions and providers

!!! info "External Resources"
    - [RFC 3227 - Guidelines for Evidence Collection and Archiving](https://www.rfc-editor.org/rfc/rfc3227) (IETF)
    - [Order of Volatility in Digital Forensics](https://www.sans.org/blog/the-order-of-volatility/) (SANS Institute)
    - [Digital Evidence Collection Best Practices](https://www.nist.gov/publications/guide-integrating-forensic-techniques-incident-response) (NIST SP 800-86)

## Chain of Custody

Chain of custody is the documented chronological history of the handling, transfer, and disposition of evidence. It establishes that evidence has been preserved in an unaltered state from the point of collection through presentation in legal proceedings. A broken chain of custody can render evidence inadmissible in court, regardless of its technical relevance.

**Chain of Custody Documentation Must Include:**

- **Identification**: Unique evidence identifier, case number, description of the item
- **Collection details**: Date, time (with timezone), location, name of collecting individual, method of collection
- **Handover records**: Every transfer of custody documented with signatures of both releasing and receiving parties, including date, time, and reason for transfer
- **Storage conditions**: Where evidence was stored, how it was secured (locked cabinet, evidence room), environmental conditions if relevant
- **Access log**: Every instance of evidence being accessed, by whom, for what purpose, and for how long
- **Integrity verification**: Hash values (MD5, SHA-256, or both) computed at collection and verified at every subsequent access point

**Handover Notes Best Practices:**

- Document the purpose of each evidence transfer (e.g., "transferred to forensic examiner for disk analysis")
- Record the condition of the evidence at each transfer point
- Use tamper-evident packaging where physical media is involved
- Photograph evidence before and after any interaction
- Maintain a continuous log with no temporal gaps - any unaccounted period weakens the chain

**Hash Verification Process:**

```bash
# Generate hash at collection time
sha256sum /dev/sda > evidence_hash_collection.txt

# Verify hash at each subsequent access
sha256sum /dev/sda | diff - evidence_hash_collection.txt
```

**Common Chain of Custody Failures:**

- Evidence left unattended or in an unsecured location
- Gaps in the transfer log where no individual can account for the evidence
- Failure to re-verify hashes after evidence access
- Unsigned or undated transfer forms
- Multiple individuals handling evidence without individual documentation

!!! info "External Resources"
    - [Chain of Custody in Digital Forensics](https://www.sciencedirect.com/topics/computer-science/chain-of-custody) (ScienceDirect)
    - [NIST Guidelines on Digital Evidence Handling](https://csrc.nist.gov/publications/detail/sp/800-86/final) (NIST)
    - [ACPO Good Practice Guide for Digital Evidence](https://www.digital-detective.net/digital-forensics-documents/ACPO_Good_Practice_Guide_for_Digital_Evidence_v5.pdf) (ACPO/UK)

## Forensic Imaging & Preservation

Forensic imaging creates a bit-for-bit copy of a storage device, including unallocated space, slack space, and deleted file remnants. The original evidence must never be directly examined - all analysis is performed on verified forensic copies.

**Imaging Approaches:**

- **Physical image**: Bit-for-bit copy of the entire physical device, including all partitions and unallocated space. This is the gold standard for forensic acquisition.
- **Logical image**: Copy of the logical file system only. Faster but misses deleted data in unallocated space, slack space, and hidden partitions.
- **Targeted collection**: Selective acquisition of specific files or directories. Used when full imaging is impractical (e.g., multi-terabyte cloud storage).

**Write Blockers:**

Write blockers are hardware or software devices that prevent any write operation to the evidence source during acquisition. Hardware write blockers are preferred as they operate independently of the operating system and are more defensible in court. Software write blockers depend on correct OS-level enforcement.

**Forensic Imaging Tools and Commands:**

```bash
# Using dd for raw imaging
dd if=/dev/sda of=/evidence/case001/disk.img bs=4096 conv=noerror,sync status=progress

# Using dc3dd (enhanced dd with built-in hashing)
dc3dd if=/dev/sda of=/evidence/case001/disk.img hash=sha256 log=imaging.log

# Using ewfacquire for Expert Witness Format (compressed, with metadata)
ewfacquire /dev/sda -t /evidence/case001/disk -C case001 -D "Suspect workstation" -e "Examiner Name"
```

**Image Formats:**

| Format | Extension | Compression | Metadata | Tool Support |
|:---|:---|:---|:---|:---|
| Raw (dd) | `.img`, `.raw` | None | None embedded | Universal |
| Expert Witness Format | `.E01` | Yes | Case info, hashes | EnCase, FTK |
| Advanced Forensic Format | `.aff` | Yes | Extensive metadata | Open source |
| SMART | `.s01` | Yes | Limited | ASR Data |

**Verification:**

After imaging, compare the hash of the source device against the hash of the image. Both must match exactly. Document the hashes, the tool used, and the operator. Re-verify before every analysis session.

!!! info "External Resources"
    - [Forensic Disk Imaging with dc3dd](https://www.sans.org/blog/forensic-imaging-with-dc3dd/) (SANS Institute)
    - [NIST Computer Forensic Tool Testing - Disk Imaging](https://www.nist.gov/itl/ssd/software-quality-group/computer-forensics-tool-testing-program-cftt/cftt-technical/disk) (NIST CFTT)
    - [FTK Imager Documentation](https://www.exterro.com/digital-forensics-software/ftk-imager) (Exterro)

## Anti-Forensics Techniques

Anti-forensics encompasses techniques used by adversaries to hinder, delay, or prevent forensic analysis. Understanding these techniques is essential for investigators to recognize when evidence has been tampered with and to employ countermeasures during examination.

**Timestomping:**

Timestomping is the modification of file system timestamps (creation, modification, access, and entry modification times) to conceal the true timeline of attacker activity. On NTFS, each file has two sets of timestamps - the `$STANDARD_INFORMATION` attribute and the `$FILE_NAME` attribute. Common timestomping tools modify only `$STANDARD_INFORMATION`, leaving `$FILE_NAME` intact. Forensic examiners compare both attribute sets to detect discrepancies.

```
# Detection: Compare $SI and $FN timestamps using Sleuth Kit
istat -o <partition_offset> image.raw <inode_number>
# If $SI timestamps are earlier than $FN timestamps, timestomping is indicated
# $FN timestamps can only be modified by the kernel
```

**Data Destruction and Wiping:**

- **Secure deletion tools**: Overwrite file content with random or zero data before unlinking (e.g., `shred`, `sdelete`)
- **Full disk wiping**: DBAN, nwipe for HDD; manufacturer secure erase for SSD
- **SSD-specific challenges**: Wear leveling and over-provisioning mean deleted data may persist in areas inaccessible to standard tools but recoverable with chip-off forensics

**Log Manipulation:**

- Clearing Windows Event Logs (leaves Event ID 1102 - "The audit log was cleared")
- Editing or truncating syslog/journald entries
- Disabling logging services or modifying audit policies
- Timestomping log entries to disrupt timeline analysis

**How Malware Hides:**

- **Process injection**: Injecting code into legitimate processes to avoid appearing as a separate suspicious process
- **Rootkits**: Kernel-level or userspace rootkits that hook system calls to hide files, processes, network connections, and registry entries from standard system tools
- **Fileless malware**: Operates entirely in memory, using legitimate system tools (PowerShell, WMI, .NET) and leaving minimal disk artifacts
- **Steganography**: Hiding data within image files, audio files, or network protocols
- **Packing and encryption**: Compressing or encrypting malware binaries to evade signature-based detection and complicate static analysis
- **Living-off-the-land binaries (LOLBins)**: Using legitimate system utilities (certutil, mshta, regsvr32) to execute malicious payloads

**Encrypted Volumes and Containers:**

Full-disk encryption (BitLocker, LUKS, FileVault) and encrypted containers (VeraCrypt) prevent access to evidence without the decryption key. Memory forensics may recover encryption keys from RAM if the system is still powered on. Investigators should never shut down an encrypted system before capturing memory.

!!! info "External Resources"
    - [Anti-Forensics Techniques - MITRE ATT&CK](https://attack.mitre.org/tactics/TA0005/) (MITRE)
    - [Timestomping Detection with NTFS Artifacts](https://www.sans.org/blog/digital-forensic-sifting-timestamp-analysis-filter-using-a-tool/) (SANS Institute)
    - [Fileless Malware and Anti-Forensics](https://www.crowdstrike.com/en-us/cybersecurity-101/malware/fileless-malware/) (CrowdStrike)

## Legal Considerations

Digital forensics operates at the intersection of technology and law. Investigators must understand the legal frameworks governing evidence collection, preservation, and presentation to ensure that their findings are admissible and that they do not expose their organization to legal liability.

**Jurisdictional Issues:**

- Digital evidence frequently spans multiple legal jurisdictions, especially in cloud environments
- International cooperation frameworks (MLATs - Mutual Legal Assistance Treaties) may be required for cross-border evidence collection
- The CLOUD Act (US) allows law enforcement to compel US-based providers to produce data regardless of where it is stored
- GDPR (EU) imposes restrictions on processing personal data, even during forensic investigations

**Legal Authority for Collection:**

- **Law enforcement**: Requires warrants, subpoenas, or court orders depending on the type of evidence and jurisdiction
- **Corporate investigations**: Authority derives from employment agreements, acceptable use policies, and corporate ownership of systems
- **Incident response**: Organizations generally have authority to investigate their own systems, but must respect employee privacy rights that vary by jurisdiction
- **Consent**: Voluntary consent from the data owner may authorize collection, but the scope of consent must be clearly documented

**Admissibility Standards:**

- **Daubert Standard (US)**: Expert testimony and scientific evidence must be based on sufficient facts, the product of reliable principles and methods, and applied reliably to the case
- **Federal Rules of Evidence (US)**: Rules 901 and 902 govern authentication of digital evidence; Rule 803(6) addresses business records exceptions to hearsay
- **Best Evidence Rule**: The original recording of data (or a verified forensic copy) is preferred over secondary descriptions

**Privacy and Data Protection:**

- Investigators must minimize collection of data not relevant to the investigation
- Personal data encountered during forensic analysis may be subject to data protection regulations
- Legal hold obligations require preservation of potentially relevant evidence once litigation is reasonably anticipated
- Attorney-client privilege and work product doctrine may protect certain forensic reports from disclosure

**Documentation for Legal Proceedings:**

- All forensic procedures must be documented in sufficient detail for independent reproduction
- Reports should distinguish between factual observations and expert opinion
- Investigators must be prepared to testify about their qualifications, methodology, tools used, and findings
- Peer review of forensic findings strengthens their credibility in court

!!! info "External Resources"
    - [DOJ Guidelines for Searching and Seizing Computers](https://www.justice.gov/criminal/file/442111/dl) (US Department of Justice)
    - [ISO/IEC 27037 - Digital Evidence Identification and Collection](https://www.iso.org/standard/44381.html) (ISO)
    - [GDPR and Digital Forensics](https://www.enisa.europa.eu/topics/incident-response/glossary/gdpr-and-digital-forensics) (ENISA)
