# Directory Services (LDAP & Kerberos)

Directory services are used to manage identities, permissions, and resources within an internal network, most commonly in corporate environments.

## LDAP (Lightweight Directory Access Protocol)

LDAP is used to query and modify items in directory service providers like Microsoft Active Directory (AD) or OpenLDAP.

### How it Works (Auth)
1.  **Bind**: The client sends a "Bind Request" with a Distinguished Name (DN) and password.
2.  **Comparison**: The server verifies the credentials against its database.
3.  **Result**: If correct, the connection is authenticated.

### Threats and Mitigations
-   **LDAP Injection**: Unsanitized user input in filters can allow attackers to bypass auth. *Mitigation: Use parameterized queries/escaping.*
-   **Cleartext Credentials**: LDAP sends passwords in the clear by default. *Mitigation: Use **LDAPS** (LDAP over TLS) or **STARTTLS**.*

---

## Kerberos

Kerberos is a ticket-based authentication protocol designed for secure communication over untrusted networks. It is the default authentication method for Windows domains.

### How it Works
1.  **AS-REQ**: User requests a Ticket Granting Ticket (TGT) from the Authentication Service (AS).
2.  **TGT**: AS provides an encrypted TGT.
3.  **TGS-REQ**: User presents the TGT to the Ticket Granting Service (TGS) to request a service ticket.
4.  **Service Ticket**: TGS provides a ticket for a specific resource (e.g., a file share).
5.  **Access**: User presents the service ticket to the resource.

### Threats and Mitigations
-   **Kerberoasting**: Attackers request service tickets for accounts with SPNs and attempt to crack the password hashes offline. *Mitigation: Use long, complex passwords for service accounts.*
-   **AS-REP Roasting**: If "Pre-Authentication" is disabled, attackers can get encrypted data for any user and crack it offline. *Mitigation: Ensure "Do not require Kerberos preauthentication" is NOT checked.*
-   **Golden Ticket**: If the `krbtgt` account hash is stolen, attackers can create forged TGTs for any user. *Mitigation: Rotate the `krbtgt` password regularly.*

## Pros and Cons

| Pros | Cons |
|------|------|
| **Single Sign-On**: One login grants access to the entire domain. | **Complexity**: Kerberos requires strict time synchronization (NTP). |
| **Proven Security**: Strong cryptographic foundations. | **Network Dependent**: Requires direct line-of-sight to Domain Controllers. |
| **Centralized Control**: Admins can manage all users and permissions in one place. | **Legacy**: Difficult to adapt for modern web/cloud-native applications. |
