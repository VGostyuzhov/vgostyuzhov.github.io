# AD & Windows Attack Techniques: A Study Guide

This guide covers the most common and impactful attack techniques targeting Active Directory and Windows environments. Understanding these attacks is critical for building effective defenses and for security engineering interviews.

## Credential Harvesting

### LSASS Credential Dumping

The LSASS process stores credentials for active sessions. Tools like **Mimikatz** can extract:

- **Plaintext passwords** (if WDigest is enabled or on older systems)
- **NT hashes** (usable for Pass-the-Hash)
- **Kerberos tickets** (TGTs and Service Tickets)

```
# Mimikatz commands
privilege::debug
sekurlsa::logonpasswords    # Dump all credentials from LSASS
sekurlsa::tickets           # Extract Kerberos tickets
```

**Variations:**

- **Procdump/Task Manager**: Dump LSASS memory to disk, then extract offline
- **comsvcs.dll**: `rundll32 comsvcs.dll, MiniDump <lsass_pid> dump.bin full` - living-off-the-land technique
- **Direct syscalls**: Advanced tools use direct NT syscalls to bypass API hooks and EDR

### SAM Database Extraction

The local SAM database contains NT hashes for local accounts:

- **Registry extraction**: `reg save HKLM\SAM sam.bak` and `reg save HKLM\SYSTEM sys.bak`, then parse offline
- **Volume Shadow Copy**: Access locked SAM file via VSS snapshots
- **Secretsdump**: Impacket's tool can dump SAM, LSA secrets, and cached domain credentials remotely

### NTDS.dit Extraction

The AD database on Domain Controllers contains every domain account's password hash:

- **Volume Shadow Copy**: `vssadmin create shadow /for=C:` then copy `NTDS.dit`
- **ntdsutil**: `ntdsutil "activate instance ntds" "ifm" "create full C:\extract"` - creates an Install From Media backup
- **DCSync**: See below - does not require file access

## Kerberos Attacks

### Kerberoasting

Exploits the fact that Service Tickets are encrypted with the target service account's password hash. Any authenticated domain user can request a Service Ticket for any SPN-registered service.

**Attack Flow:**

1. Enumerate accounts with SPNs: `Get-ADUser -Filter {ServicePrincipalName -ne "$null"}`
2. Request Service Tickets for those SPNs
3. Extract the tickets and crack them offline

**Why It Works**: Service accounts often have weak passwords and are rarely rotated. The ticket encryption (RC4 by default) can be brute-forced offline without generating additional authentication events.

**Tools**: Rubeus, Impacket's `GetUserSPNs.py`, PowerView

### AS-REP Roasting

Targets accounts that have Kerberos **pre-authentication disabled**. Without pre-authentication, the KDC returns an AS-REP encrypted with the user's hash without verifying identity.

**Attack Flow:**

1. Enumerate accounts with `DONT_REQUIRE_PREAUTH` flag
2. Request AS-REP for those accounts
3. Crack the encrypted portion offline

**Tools**: Rubeus (`asreproast`), Impacket's `GetNPUsers.py`

### Golden Ticket

A forged TGT created using the **krbtgt** account's password hash. Since TGTs are encrypted with the krbtgt hash, possessing it allows an attacker to create tickets for any user with any group memberships.

**Requirements:**

- `krbtgt` account NT hash (obtained via DCSync or NTDS.dit extraction)
- Domain SID

**Impact:**

- Complete domain compromise
- Tickets can be created for non-existent users
- Persists until `krbtgt` password is changed **twice** (due to password history)
- Default TGT lifetime of 10 hours, but a Golden Ticket can specify any lifetime

### Silver Ticket

A forged Service Ticket created using a **service account's** password hash. Unlike Golden Tickets, Silver Tickets do not require communication with the KDC and target specific services.

**Requirements:**

- Target service account's NT hash
- Target SPN
- Domain SID

**Impact:**

- Access to the specific service only
- Does not touch the DC - harder to detect
- Common targets: CIFS (file shares), HTTP (web apps), MSSQL, HOST (WMI/PSRemoting)

### Diamond Ticket

A more stealthy variant of the Golden Ticket. Instead of forging a TGT from scratch, the attacker:

1. Requests a legitimate TGT
2. Decrypts it using the `krbtgt` hash
3. Modifies the PAC (adds privileged group SIDs)
4. Re-encrypts it

This produces a ticket with a legitimate structure that is harder for detection tools to flag.

## Lateral Movement

### Pass-the-Hash (PtH)

Uses the **NT hash** directly for NTLM authentication without knowing the plaintext password.

**How It Works**: The NT hash is the only value needed for the NTLM challenge-response. An attacker injects the hash into memory and authenticates as the user.

**Tools**: Mimikatz (`sekurlsa::pth`), Impacket (`psexec.py`, `wmiexec.py`), CrackMapExec

### Pass-the-Ticket (PtT)

Injects a stolen **Kerberos ticket** (TGT or Service Ticket) into the current session to access resources as the ticket's owner.

**Tools**: Mimikatz (`kerberos::ptt`), Rubeus (`ptt`)

### Overpass-the-Hash

Converts an NT hash into a Kerberos TGT, allowing Kerberos-based lateral movement using only the hash.

**Flow:**

1. Obtain NT hash
2. Use hash to request a TGT via AS-REQ (RC4 encryption)
3. Use the TGT for Kerberos authentication

This bypasses controls that block NTLM but allow Kerberos.

### NTLM Relay

Captures an NTLM authentication attempt and relays it to a different target service.

**Common Scenarios:**

- **LLMNR/NBT-NS Poisoning**: Respond to name resolution broadcasts to capture authentication
- **Coerced Authentication**: Force a machine to authenticate (e.g., PetitPotam, PrinterBug) and relay to AD CS, LDAP, or SMB
- **LDAP Relay**: Relay to LDAP to modify AD objects (add users to groups, configure RBCD)

**Mitigations**: SMB Signing, LDAP Signing, EPA (Extended Protection for Authentication), disable LLMNR/NBT-NS

## Domain Escalation

### DCSync

Mimics the replication protocol used between Domain Controllers to request password hashes from AD. Requires **Replicating Directory Changes** and **Replicating Directory Changes All** permissions (held by Domain Admins, Enterprise Admins, and Domain Controllers by default).

```
# Mimikatz DCSync
lsadump::dcsync /domain:corp.example.com /user:krbtgt
lsadump::dcsync /domain:corp.example.com /all /csv
```

**Impact**: Extracts any account's NT hash remotely, including `krbtgt` (enabling Golden Tickets).

### AD Certificate Services (AD CS) Abuse

AD CS provides PKI capabilities in AD. Misconfigurations can lead to domain compromise:

| Technique | Description |
| :--- | :--- |
| **ESC1** | Enrollee can specify a Subject Alternative Name (SAN), allowing certificate requests on behalf of any user |
| **ESC2** | Template allows any purpose EKU or no EKU restriction |
| **ESC3** | Certificate Request Agent template allows enrolling on behalf of others |
| **ESC4** | Vulnerable template ACLs allow modification of template settings |
| **ESC6** | CA has `EDITF_ATTRIBUTESUBJECTALTNAME2` flag enabled, allowing SAN in any request |
| **ESC7** | Vulnerable CA ACLs allow an attacker to become CA officer |
| **ESC8** | NTLM relay to the CA's HTTP enrollment endpoint (web enrollment) |

**Tools**: Certify, Certipy, ForgeCert

### Resource-Based Constrained Delegation (RBCD)

An attacker who can modify the `msDS-AllowedToActOnBehalfOfOtherIdentity` attribute on a target computer can configure it to accept delegation from an attacker-controlled account.

**Attack Flow:**

1. Create or control a computer account (any user can join up to 10 machines by default)
2. Set the RBCD attribute on the target to trust the attacker's computer account
3. Use S4U2Self and S4U2Proxy to obtain a Service Ticket as any user to the target

### Group Policy Abuse

If an attacker gains write access to a GPO linked to sensitive OUs:

- Deploy **Scheduled Tasks** that execute malicious code on all affected machines
- Modify **Restricted Groups** to add attacker accounts to local Administrators
- Push **logon scripts** or change security settings

## Persistence

### DCShadow

Registers a rogue Domain Controller in AD, allowing an attacker to inject arbitrary changes into the directory via replication. Changes made this way bypass standard security logging.

### AdminSDHolder & SDProp

The `AdminSDHolder` container's ACL is applied to all protected groups (Domain Admins, Enterprise Admins, etc.) every 60 minutes by the `SDProp` process. An attacker who modifies the AdminSDHolder ACL gains persistent access to all protected groups.

### Skeleton Key

A patch injected into LSASS on a Domain Controller that allows authentication with a master password alongside the legitimate password. All existing passwords continue to work.

### SID History Injection

Adding a privileged SID (e.g., Enterprise Admins SID) to a user's `sIDHistory` attribute grants those privileges without modifying group memberships directly. Effective for cross-domain escalation within a forest.

### Attack Techniques Summary

| Category | Technique | Prerequisite | Impact |
| :--- | :--- | :--- | :--- |
| **Credential Harvesting** | LSASS Dump | Local admin on target | Credentials for all active sessions |
| | DCSync | Replicating Directory Changes permissions | Any account's NT hash |
| | NTDS.dit Extraction | Domain Controller access | All domain password hashes |
| **Kerberos** | Kerberoasting | Any domain user | Service account password hashes |
| | AS-REP Roasting | Any domain user (or unauthenticated) | User password hashes |
| | Golden Ticket | krbtgt hash | Full domain compromise |
| | Silver Ticket | Service account hash | Access to specific service |
| **Lateral Movement** | Pass-the-Hash | NT hash | Authenticate as user (NTLM) |
| | Pass-the-Ticket | Kerberos ticket | Authenticate as user (Kerberos) |
| | NTLM Relay | Network position | Execute actions as relayed user |
| **Domain Escalation** | AD CS Abuse | Misconfigured templates/CA | Domain Admin or any user impersonation |
| | RBCD | Write to target's AD object | Service access as any user |
| **Persistence** | DCShadow | Domain Admin | Stealthy directory modifications |
| | Skeleton Key | Domain Controller access | Backdoor master password |
| | SID History | Privileged access | Cross-domain privilege escalation |

!!! info "External Resources"
    - [The Hacker Recipes - Active Directory](https://www.thehacker.recipes/ad/movement) (Comprehensive AD attack reference)
    - [Certified Pre-Owned - AD CS Whitepaper](https://specterops.io/wp-content/uploads/sites/3/2022/06/Certified_Pre-Owned.pdf) (SpecterOps - AD CS attack research)
    - [Harmj0y's Kerberoasting Overview](https://blog.harmj0y.net/powershell/kerberoasting-without-mimikatz/) (Original Kerberoasting research)
    - [Impacket](https://github.com/fortra/impacket) (Python collection of tools for AD attacks and network protocols)
    - [PayloadsAllTheThings - AD Attacks](https://github.com/swisskyrepo/PayloadsAllTheThings/blob/master/Methodology%20and%20Resources/Active%20Directory%20Attack.md) (Comprehensive cheat sheet)
