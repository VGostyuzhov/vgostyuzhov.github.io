# AD & Windows Defensive Strategies: A Study Guide

This guide covers the key defensive strategies for securing Active Directory and Windows environments. The focus is on practical, high-impact controls that a security engineer should know how to implement and evaluate.

## Tiered Administration Model

The **Enterprise Access Model** (formerly the Tier Model) is Microsoft's recommended approach to preventing credential theft and lateral movement by segmenting administrative access into isolated tiers.

| Tier | Scope | Examples |
| :--- | :--- | :--- |
| **Tier 0 (Control Plane)** | Identity systems and forest/domain administration | Domain Controllers, AD CS, Azure AD Connect, ADFS, PKI |
| **Tier 1 (Management Plane)** | Server and application administration | Member servers, databases, application servers |
| **Tier 2 (User Access)** | Workstation and end-user device management | Workstations, laptops, help desk |

**Key Principles:**

- **No downward authentication**: Tier 0 credentials must never be used on Tier 1 or Tier 2 systems
- **No upward access**: Tier 2 systems cannot manage Tier 0 or Tier 1 resources
- **Separate accounts**: Administrators have distinct accounts per tier (e.g., `admin-t0-jsmith`, `admin-t1-jsmith`)
- **Dedicated workstations**: Privileged Access Workstations (PAWs) for each tier

## Privileged Access Workstations (PAW)

PAWs are hardened workstations dedicated exclusively to administrative tasks. They are isolated from internet access, email, and general productivity tools.

**PAW Hardening:**

- Clean OS installation from trusted media
- Application whitelisting (AppLocker/WDAC) - only administrative tools allowed
- No internet browsing or email
- Credential Guard enabled to protect LSASS
- Device Guard / HVCI for kernel integrity
- Local admin password managed by LAPS
- Network segmentation - PAWs can only reach their designated tier

## Credential Protection

### Credential Guard

**Windows Credential Guard** uses virtualization-based security (VBS) to isolate LSASS secrets in a protected memory space that even a compromised kernel cannot access.

**What It Protects:**

- NTLM hashes
- Kerberos TGTs and session keys
- Derived credentials

**What It Does NOT Protect:**

- Credentials stored by applications or third-party SSPs
- Local account hashes in SAM
- Kerberos Service Tickets

### LAPS (Local Administrator Password Solution)

LAPS automatically manages and rotates local administrator passwords on domain-joined machines, storing them securely in AD attributes.

**How It Works:**

1. A Group Policy extension on each machine generates a random password
2. The password is stored in a confidential AD attribute (`ms-Mcs-AdmPwd`) on the computer object
3. ACLs control which users/groups can read the password
4. Passwords are automatically rotated at a configurable interval

**Windows LAPS** (the updated version) additionally supports:

- Password encryption using a designated security principal
- Password history
- Azure AD (Entra ID) backed storage
- DSRM account password management on DCs

### Protected Users Security Group

Members of the **Protected Users** group have additional security restrictions:

- Kerberos TGTs have a 4-hour lifetime (non-renewable)
- NTLM authentication is blocked
- DES and RC4 encryption are not used for Kerberos pre-authentication
- Credentials are not cached
- No delegation (constrained or unconstrained) is allowed

**Best Practice**: Add all highly-privileged accounts (Domain Admins, Enterprise Admins) to the Protected Users group.

### Managed Service Accounts

**Group Managed Service Accounts (gMSAs)** address the problem of weak service account passwords (which enable Kerberoasting):

- Passwords are 240-character random values
- Automatically rotated every 30 days by AD
- Cannot be set or known by administrators
- Multiple servers can share the same gMSA

## GPO Hardening

### Authentication Policies

Key Group Policy settings for hardening authentication:

- **Restrict NTLM**: `Network Security: Restrict NTLM: NTLM authentication in this domain` - Audit then deny NTLM
- **LAN Manager Authentication Level**: Set to "Send NTLMv2 response only. Refuse LM & NTLM"
- **Kerberos Encryption Types**: Disable RC4 (DES is disabled by default), enforce AES
- **Kerberos Armoring (FAST)**: Protects pre-authentication exchanges from offline cracking

### Attack Surface Reduction

- **Disable LLMNR**: GPO > Computer Config > Admin Templates > Network > DNS Client > "Turn off multicast name resolution"
- **Disable NBT-NS**: Network adapter settings or via DHCP option
- **SMB Signing**: Required on all machines (`RequireSecuritySignature = 1`)
- **LDAP Signing**: Required on Domain Controllers (`LDAPServerIntegrity = 2`)
- **LDAP Channel Binding**: Set to "Always" on Domain Controllers
- **WDigest**: Ensure disabled (`UseLogonCredential = 0`) to prevent plaintext password caching
- **Disable Spooler Service**: On Domain Controllers and servers to prevent PrinterBug/SpoolSample abuse

### AppLocker / WDAC

Application control prevents execution of unauthorized code:

- **AppLocker**: Rule-based, supports path/publisher/hash rules for EXEs, DLLs, scripts, and packaged apps
- **Windows Defender Application Control (WDAC)**: Kernel-enforced, more resistant to bypass than AppLocker

**Minimum Policy**: Block execution from user-writable paths (`%TEMP%`, `Downloads`, `AppData`).

## AD Certificate Services Hardening

AD CS misconfigurations are among the most impactful vulnerabilities in modern AD environments.

**Hardening Steps:**

- **Audit templates**: Remove `CT_FLAG_ENROLLEE_SUPPLIES_SUBJECT` from templates unless strictly required (prevents ESC1)
- **Restrict enrollment permissions**: Only authorized groups should have `Enroll` permissions on sensitive templates
- **Disable web enrollment**: Prevent NTLM relay to the CA (ESC8)
- **Remove `EDITF_ATTRIBUTESUBJECTALTNAME2`** flag from CA configuration (prevents ESC6)
- **Enforce Manager Approval** on high-privilege certificate templates
- **Harden CA ACLs**: Only CA administrators should have `ManageCA` and `ManageCertificates` permissions
- **Monitor**: Track certificate enrollment events (Event IDs 4886, 4887)

## Monitoring and Detection

### Critical Event IDs

| Event ID | Source | Indicates |
| :--- | :--- | :--- |
| **4624** | Security | Successful logon (check logon type) |
| **4625** | Security | Failed logon attempt |
| **4648** | Security | Explicit credential logon (RunAs) |
| **4662** | Security | Operation performed on AD object (DCSync detection) |
| **4672** | Security | Special privileges assigned (admin logon) |
| **4720** | Security | User account created |
| **4728/4732/4756** | Security | Member added to security group |
| **4768** | Security | Kerberos TGT requested (AS-REQ) |
| **4769** | Security | Kerberos Service Ticket requested (TGS-REQ) |
| **4771** | Security | Kerberos pre-authentication failed |
| **5136** | Security | Directory service object modified |
| **5145** | Security | Network share access check |
| **7045** | System | New service installed |

### Detection Strategies

**Kerberoasting Detection:**

- High volume of Event ID 4769 with RC4 encryption (`0x17`) from a single source
- Service Ticket requests for unusual SPNs or from unexpected accounts

**DCSync Detection:**

- Event ID 4662 with `Replicating Directory Changes` rights from a non-DC source
- Network traffic with DRS (Directory Replication Service) RPC calls from non-DC IPs

**Golden Ticket Detection:**

- TGT with an abnormal lifetime
- TGT encryption type mismatch (RC4 when AES is enforced)
- Event ID 4769 without a preceding 4768 (Service Ticket request without TGT request)

**Pass-the-Hash Detection:**

- Event ID 4624 with Logon Type 9 (NewCredentials) or 3 (Network) from unusual sources
- NTLM authentication from accounts that should only use Kerberos

**Lateral Movement Detection:**

- Event ID 7045 (new service) on endpoints - indicates PsExec-style movement
- Event ID 4648 (explicit credential use) patterns across multiple machines
- WinRM/PSRemoting connections from unexpected sources

### Tools for AD Security Assessment

| Tool | Purpose |
| :--- | :--- |
| **BloodHound** | Map AD relationships and identify attack paths to Domain Admin |
| **PingCastle** | AD security assessment - generates a health score and prioritized findings |
| **Purple Knight** | AD security assessment with checks mapped to MITRE ATT&CK |
| **Certify / Certipy** | Enumerate and identify AD CS misconfigurations |
| **PlumHound** | BloodHound data analysis and reporting |
| **ADRecon** | Comprehensive AD reconnaissance and reporting |

## Defensive Measures Summary

| Attack | Primary Defense | Detection |
| :--- | :--- | :--- |
| **LSASS Credential Dump** | Credential Guard, LSA Protection (`RunAsPPL`), disable WDigest | Sysmon process access on LSASS, MDATP alerts |
| **Kerberoasting** | gMSAs, long and complex service account passwords, AES-only encryption | High-volume 4769 with RC4 from single source |
| **AS-REP Roasting** | Ensure pre-authentication is required on all accounts | 4768 with error for accounts that should have pre-auth |
| **Golden Ticket** | Rotate `krbtgt` password regularly (twice), enforce AES | TGT without preceding AS-REQ, anomalous ticket lifetimes |
| **Pass-the-Hash** | Credential Guard, Protected Users group, restrict NTLM | 4624 Type 3/9 with NTLM from unexpected sources |
| **NTLM Relay** | SMB/LDAP Signing, EPA, disable LLMNR/NBT-NS | Network detection of relay patterns, unexpected NTLM |
| **DCSync** | Audit and restrict Replicating Directory Changes ACLs | 4662 from non-DC, DRS RPC from non-DC IP |
| **AD CS Abuse** | Template hardening, disable web enrollment, CA ACL audit | 4886/4887 enrollment events, unusual certificate requests |
| **RBCD** | Monitor `msDS-AllowedToActOnBehalfOfOtherIdentity` changes | 5136 for delegation attribute modifications |
| **Lateral Movement** | Network segmentation, PAWs, firewall rules (block 445/135/5985 between workstations) | 7045, 4648 patterns, anomalous remote connections |

!!! info "External Resources"
    - [Microsoft Enterprise Access Model](https://learn.microsoft.com/en-us/security/privileged-access-workstations/privileged-access-access-model) (Microsoft Learn)
    - [Best Practices for Securing Active Directory](https://learn.microsoft.com/en-us/windows-server/identity/ad-ds/plan/security-best-practices/best-practices-for-securing-active-directory) (Microsoft Learn)
    - [PingCastle](https://www.pingcastle.com/) (Free AD security assessment tool)
    - [BloodHound](https://github.com/BloodHoundAD/BloodHound) (AD attack path mapping)
    - [Windows Security Event Log Encyclopedia](https://www.ultimatewindowssecurity.com/securitylog/encyclopedia/) (Event ID reference)
    - [ANSSI AD Security Guide](https://www.cert.ssi.gouv.fr/uploads/guide-ad.html) (French National Cybersecurity Agency - comprehensive AD hardening)
