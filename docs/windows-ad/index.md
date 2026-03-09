# AD & Windows Core Concepts: A Study Guide

Active Directory (AD) is the backbone of identity and access management in most enterprise environments. Understanding its architecture, authentication mechanisms, and common attack surfaces is essential for any security engineer. This guide covers AD fundamentals and Windows-specific security concepts - the building blocks required before exploring attack techniques or defensive strategies.

**Related Pages:**

- [Attack Techniques](attacks.md) - Credential harvesting, Kerberoasting, Pass-the-Hash, Golden Tickets, DCSync, AD CS abuse, lateral movement
- [Defensive Strategies](defenses.md) - Tiered administration, PAW/PAM, credential hygiene, GPO hardening, monitoring, LAPS, detection strategies

## Active Directory Architecture

### Domains, Trees, and Forests

Active Directory is organized in a hierarchical structure:

- **Domain**: The core unit of AD. All objects (users, computers, groups) belong to a domain. Each domain has its own security boundary and replication scope.
- **Tree**: One or more domains that share a contiguous DNS namespace (e.g., `corp.example.com` and `eu.corp.example.com`). Parent-child trust relationships are automatic and transitive.
- **Forest**: The outermost boundary. A forest contains one or more trees and represents the ultimate trust and schema boundary. The first domain created becomes the **Forest Root Domain**.
- **Global Catalog (GC)**: A distributed data repository containing a searchable, partial representation of every object in every domain within a forest. Used for cross-domain searches and logon processing.

### Domain Controllers

Domain Controllers (DCs) are the servers responsible for:

- Hosting a writable copy of the AD database (`NTDS.dit`)
- Authenticating users and computers via Kerberos or NTLM
- Replicating directory changes to other DCs
- Enforcing Group Policy

**Read-Only Domain Controllers (RODCs)** hold a read-only copy of the AD database and are used in branch offices or DMZs where physical security is limited. They do not cache passwords by default and cannot process authentication locally without contacting a writable DC.

### FSMO Roles

Five **Flexible Single Master Operations** roles exist per forest/domain. These roles handle operations that must be performed by a single DC to avoid conflicts:

| Role | Scope | Purpose |
| :--- | :--- | :--- |
| **Schema Master** | Forest-wide | Controls schema modifications |
| **Domain Naming Master** | Forest-wide | Manages adding/removing domains in the forest |
| **PDC Emulator** | Per domain | Password changes, time sync, account lockout, GPO coordination |
| **RID Master** | Per domain | Allocates RID pools for SID creation |
| **Infrastructure Master** | Per domain | Resolves cross-domain object references |

## Trust Relationships

Trusts allow users in one domain to access resources in another.

- **Parent-Child Trust**: Automatic, two-way transitive trust between parent and child domains
- **Tree-Root Trust**: Automatic, two-way transitive trust between tree roots in the same forest
- **Forest Trust**: Manually created between two forest root domains. Can be one-way or two-way, and is transitive at the forest level.
- **External Trust**: Manually created between domains in different forests. Non-transitive.
- **Shortcut Trust**: Manually created to optimize authentication paths within a forest

**Security Implication**: Trusts expand the attack surface. Compromising a trusted domain can lead to lateral movement into the trusting domain. SID Filtering is used on forest trusts to prevent SID history-based attacks.

## Authentication Protocols

### Kerberos

Kerberos is the default authentication protocol in AD (since Windows 2000). It uses symmetric-key cryptography and a trusted third party - the Key Distribution Center (KDC), which runs on every Domain Controller.

**Authentication Flow:**

1. **AS-REQ**: Client sends an Authentication Service Request to the KDC, encrypted with the user's password hash (derived key)
2. **AS-REP**: KDC validates credentials and returns a **Ticket Granting Ticket (TGT)**, encrypted with the `krbtgt` account's password hash
3. **TGS-REQ**: Client presents the TGT to request a **Service Ticket (TGS)** for a specific service
4. **TGS-REP**: KDC returns the Service Ticket, encrypted with the target service account's password hash
5. **AP-REQ**: Client presents the Service Ticket to the target service
6. **AP-REP** (optional): Service authenticates itself back to the client (mutual authentication)

**Key Components:**

- **TGT (Ticket Granting Ticket)**: Proves the user has authenticated; encrypted with `krbtgt` hash
- **TGS (Service Ticket)**: Grants access to a specific service; encrypted with the service account's hash
- **PAC (Privilege Attribute Certificate)**: Embedded in Kerberos tickets, contains the user's SID, group memberships, and privileges. Used for authorization decisions.
- **Pre-authentication**: By default, the KDC requires the client to prove knowledge of the password before issuing a TGT. Disabling this enables AS-REP Roasting.

### NTLM

NTLM is a legacy challenge-response protocol still present in many environments. It is weaker than Kerberos and should be minimized.

**Authentication Flow:**

1. Client sends username to the server
2. Server responds with a random **challenge** (nonce)
3. Client encrypts the challenge using the NT hash of the user's password and sends the **response**
4. Server forwards the response to the DC for validation (or validates locally)

**NTLM Weaknesses:**

- No mutual authentication - the server does not prove its identity
- Susceptible to relay attacks (NTLM Relay)
- NT hashes are equivalent to passwords - possessing the hash is enough to authenticate (Pass-the-Hash)
- NTLMv1 uses DES-based encryption and is trivially crackable

## LDAP and DNS Integration

### LDAP

Lightweight Directory Access Protocol is the primary protocol for querying and modifying Active Directory. Every object in AD has a **Distinguished Name (DN)** that uniquely identifies it:

```
CN=John Smith,OU=Engineering,DC=corp,DC=example,DC=com
```

**Common LDAP Operations:**

- **Bind**: Authenticate to the directory (Simple Bind, SASL, etc.)
- **Search**: Query objects with filters like `(&(objectClass=user)(memberOf=CN=Admins,...))`
- **Modify**: Change attributes on objects

**Security Consideration**: LDAP traffic on port 389 is unencrypted by default. **LDAPS** (port 636) or **LDAP Channel Binding** and **LDAP Signing** should be enforced to prevent interception and relay attacks.

### DNS

Active Directory is tightly integrated with DNS. Domain Controllers register SRV records that clients use to locate services:

- `_ldap._tcp.dc._msdcs.<domain>` - Locating Domain Controllers
- `_kerberos._tcp.dc._msdcs.<domain>` - Locating KDCs
- `_gc._tcp.<forest>` - Locating Global Catalog servers

**AD-integrated DNS zones** store DNS data within the AD database itself, benefiting from AD replication and security.

## Group Policy

Group Policy Objects (GPOs) are the primary mechanism for centralized configuration management in AD.

**How GPO Processing Works:**

1. GPOs are linked to **Sites**, **Domains**, or **Organizational Units (OUs)**
2. They are processed in order: Local -> Site -> Domain -> OU (LSDOU)
3. Last applied policy wins (unless `Enforced` is set)
4. `Block Inheritance` on an OU prevents higher-level GPOs from applying (unless `Enforced`)

**Security-Relevant GPO Settings:**

- **Password Policy**: Complexity, length, history, lockout thresholds
- **Audit Policy**: Logon events, object access, privilege use, process tracking
- **User Rights Assignment**: Who can log on locally, remotely, act as part of the OS
- **Restricted Groups**: Enforce group membership (e.g., ensure only specific accounts are in local Administrators)
- **Software Restriction Policies / AppLocker**: Control which applications can execute
- **Windows Firewall Rules**: Network access control

## Windows Security Fundamentals

### Security Identifiers (SIDs)

Every security principal (user, group, computer) has a unique **SID**. SIDs follow the format:

```
S-1-5-21-<DomainID>-<RID>
```

**Well-Known SIDs:**

| SID | Identity |
| :--- | :--- |
| `S-1-5-21-<domain>-500` | Built-in Administrator |
| `S-1-5-21-<domain>-502` | krbtgt account |
| `S-1-5-21-<domain>-512` | Domain Admins |
| `S-1-5-21-<domain>-516` | Domain Controllers |
| `S-1-5-21-<domain>-519` | Enterprise Admins |
| `S-1-5-18` | Local System |
| `S-1-5-32-544` | Built-in Administrators |

### Access Tokens and Authorization

When a user logs on, Windows creates an **Access Token** containing:

- User SID
- Group SIDs (including nested group memberships)
- Privileges (e.g., `SeDebugPrivilege`, `SeImpersonatePrivilege`)
- Integrity level

When accessing a resource, the **Security Reference Monitor** compares the token against the resource's **Security Descriptor** (which includes the DACL - Discretionary Access Control List) to determine access.

### Windows Privilege Model

Key privileges that are commonly targeted or exploited:

| Privilege | Risk |
| :--- | :--- |
| `SeDebugPrivilege` | Read/write memory of any process - enables credential dumping |
| `SeImpersonatePrivilege` | Impersonate tokens - Potato family exploits escalate to SYSTEM |
| `SeBackupPrivilege` | Read any file regardless of ACL - can extract NTDS.dit |
| `SeRestorePrivilege` | Write any file regardless of ACL - can modify system files |
| `SeTakeOwnershipPrivilege` | Take ownership of any securable object |
| `SeLoadDriverPrivilege` | Load kernel drivers - can load vulnerable/malicious drivers |

### Local Security Authority (LSA)

The **Local Security Authority Subsystem Service (LSASS)** is the process responsible for:

- Authenticating users locally and against AD
- Generating access tokens
- Enforcing security policies
- Storing credentials in memory (targeted by tools like Mimikatz)

**Credential Storage:**

- **SAM Database**: Stores local account NT hashes (located at `C:\Windows\System32\config\SAM`)
- **LSASS Memory**: Contains plaintext passwords, NT hashes, Kerberos tickets for currently logged-on users
- **NTDS.dit**: The AD database on Domain Controllers, containing all domain account hashes
- **Credential Manager**: Stores saved credentials (web, Windows) for the current user

!!! info "External Resources"
    - [Microsoft Kerberos Documentation](https://learn.microsoft.com/en-us/windows-server/security/kerberos/kerberos-authentication-overview) (Microsoft Learn)
    - [Kerberos Explained](https://www.tarlogic.com/blog/how-kerberos-works/) (Tarlogic - practical walkthrough)
    - [Harmj0y's AD Security Blog](https://blog.harmj0y.net/) (In-depth AD research and attack techniques)
    - [Group Policy Fundamentals](https://learn.microsoft.com/en-us/previous-versions/windows/it-pro/windows-server-2012-R2-and-2012/hh831791(v=ws.11)) (Microsoft Learn)
