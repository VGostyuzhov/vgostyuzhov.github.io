# SSL/TLS

SSL (Secure Sockets Layer) and its successor TLS (Transport Layer Security) provide confidentiality, integrity, and authentication for network communication. SSL is deprecated. Prefer TLS 1.3 and securely configured TLS 1.2 where compatibility requires it. Encryption, peer identity validation, application authorization, and replay handling are separate requirements.

## Glossary of Acronyms

| Acronym | Full Name | One-liner |
|---------|-----------|-----------|
| **TLS** | Transport Layer Security | Cryptographic protocol securing communication over a network |
| **SSL** | Secure Sockets Layer | Predecessor to TLS; all versions are deprecated and insecure |
| **HSTS** | HTTP Strict Transport Security | HTTP header forcing browsers to use HTTPS only |
| **PFS** | Perfect Forward Secrecy | Property ensuring past sessions remain secure if long-term keys are compromised |
| **SNI** | Server Name Indication | TLS extension that sends the target hostname in cleartext during ClientHello |
| **ECH** | Encrypted Client Hello | TLS 1.3 extension that encrypts the SNI field |
| **ESNI** | Encrypted Server Name Indication | Predecessor to ECH; deprecated in favour of ECH |
| **OCSP** | Online Certificate Status Protocol | Signed certificate-status information, subject to freshness and client policy |
| **CRL** | Certificate Revocation List | Signed list of revoked certificates published by a CA |
| **CT** | Certificate Transparency | Public log infrastructure for auditing certificate issuance |
| **CA** | Certificate Authority | Entity that issues and signs digital certificates |
| **CSR** | Certificate Signing Request | Message sent to a CA to request a signed certificate |
| **SAN** | Subject Alternative Name | X.509 extension carrying presented service identifiers such as DNS names |
| **DV/OV/EV** | Domain/Organisation/Extended Validation | Certificate validation levels with increasing assurance |
| **ACME** | Automatic Certificate Management Environment | Protocol for automated certificate issuance (Let's Encrypt) |
| **HPKP** | HTTP Public Key Pinning | Deprecated browser pinning header; CT provides detection, not equivalent key pinning |
| **AEAD** | Authenticated Encryption with Associated Data | Cipher mode providing both encryption and integrity (e.g. AES-GCM) |
| **PSK** | Pre-Shared Key | Symmetric key known to both parties before the handshake |
| **ALPN** | Application-Layer Protocol Negotiation | TLS extension for negotiating the application protocol (HTTP/2, h3) |
| **PKI** | Public Key Infrastructure | Framework of policies, hardware, software, and procedures for managing certificates |
| **mTLS** | Mutual TLS | TLS where both client and server present certificates |

## Version History

| Version | Year | Status | Notes |
|---------|------|--------|-------|
| SSL 2.0 | 1995 | Broken | No handshake authentication; vulnerable to truncation attacks |
| SSL 3.0 | 1996 | Broken | POODLE attack (2014); must be disabled everywhere |
| TLS 1.0 | 1999 | Deprecated | BEAST attack; removed by major browsers in 2020 |
| TLS 1.1 | 2006 | Deprecated | Fixed BEAST IV issue but lacks modern cipher suites; removed by browsers in 2020 |
| TLS 1.2 | 2008 | Supported | Configurable cipher suites; supports AEAD; still widely deployed |
| TLS 1.3 | 2018 | Current | AEAD suites; typical full 1-RTT handshake; ephemeral DH or PSK modes; no RSA key transport |

**What TLS 1.3 removed:**

- RSA key exchange (no PFS)
- CBC mode ciphers (padding oracle attacks)
- RC4, 3DES, MD5, SHA-1 in cipher suites
- Compression (CRIME attack)
- Renegotiation
- Static DH/ECDH (non-ephemeral)
- Custom DHE groups (must use well-known groups)
- ChangeCipherSpec no longer changes TLS 1.3 cryptographic state; a dummy message can still appear for compatibility

## TLS Architecture

TLS commonly protects application traffic over TCP. QUIC instead integrates the TLS 1.3 handshake with QUIC packet protection; it does not carry ordinary TLS records over UDP. It consists of two sub-protocols:

**Record Protocol:**

- Fragments application data into records (max 16 KB)
- Protects application records with negotiated cryptography. Legacy TLS compression existed in TLS 1.2 but should be disabled; TLS 1.3 removed it
- Adds a 5-byte header: content type, protocol version, length

**Handshake Protocol:**

- Negotiates cipher suite, authenticates parties, establishes session keys
- Uses the selected key establishment and authentication mode to derive traffic keys; PSK-only modes do not perform a new ephemeral DH exchange
- All subsequent data encrypted with the negotiated symmetric cipher

## TLS Handshakes

The traces below assume full server-certificate handshakes, with ephemeral ECDH in the TLS 1.2 example. They omit client certificates, resumption, optional messages, and TLS 1.3 HelloRetryRequest paths.

Both peers exchange public key shares and derive matching secret material without transmitting the ephemeral private values. The certificate binds a signing key to presented identities; the client validates the chain and matches its expected service name. A signed handshake proof binds key establishment to that authenticated party: TLS 1.2 ECDHE signs its key-exchange parameters, while TLS 1.3 CertificateVerify signs a transcript-bound context. Finished messages confirm possession of derived keys and integrity of the handshake transcript. Application protection starts only under the negotiated mode's validated state; the message names alone are not the checks.

### TLS 1.2 Handshake (2-RTT)

```text
Client                                Server
  |--- ClientHello ------------------>|   (versions, cipher suites, random, SNI)
  |<-- ServerHello -------------------|   (selected cipher, random)
  |<-- Certificate -------------------|   (server certificate chain)
  |<-- ServerKeyExchange -------------|   (DH/ECDH parameters, signed)
  |<-- ServerHelloDone ---------------|
  |--- ClientKeyExchange ------------>|   (client DH/ECDH public value)
  |--- ChangeCipherSpec ------------->|   (switching to encrypted)
  |--- Finished --------------------->|   (encrypted, MAC of handshake)
  |<-- ChangeCipherSpec --------------|
  |<-- Finished ----------------------|
  |<========= Application Data ======>|
```

### TLS 1.3 Handshake (1-RTT)

```text
Client                                Server
  |--- ClientHello ------------------>|   (versions, cipher suites, key_shares, SNI)
  |<-- ServerHello -------------------|   (selected cipher, key_share)
  |<-- EncryptedExtensions -----------|   (encrypted from this point)
  |<-- Certificate -------------------|
  |<-- CertificateVerify -------------|   (signature proving key ownership)
  |<-- Finished ----------------------|
  |--- Finished --------------------->|
  |<========= Application Data ======>|
```

**Key differences:**

| Property | TLS 1.2 | TLS 1.3 |
|----------|---------|---------|
| Round trips | 2-RTT | 1-RTT (0-RTT with resumption) |
| Key establishment | Legacy RSA transport or DH/PSK modes by suite | Ephemeral DH, PSK-only, or PSK with ephemeral DH; properties differ |
| Certificate authentication | During handshake, certificate normally visible | During certificate-based handshake, encrypted after ServerHello |
| Cipher negotiation | Complex (key exchange + cipher + MAC) | Simplified (AEAD only) |
| Encrypted extensions | No | Yes (server extensions hidden from passive observers) |

### 0-RTT Resumption (TLS 1.3)

A client that holds suitable resumption state can send **early data** before the new handshake completes. This reduces waiting but does not establish that the request has not been processed before. Captured early data can be replayed under the protocol's constraints, especially when multiple accepting servers do not share adequate replay state. Early data also lacks forward secrecy with respect to the PSK.

```text
Client -- ticket T, early request /export --> Region A -> export work starts
Replay -- ticket T, same early request ----> Region B -> duplicate work possible
# Both servers can understand the protected early data; encryption is not uniqueness.
# This annotated trace omits packet encoding and is not a live replay tool.
```

HTTP method names alone do not establish application replay safety. PUT/DELETE are defined as idempotent, while a badly designed GET or an expensive read can still have unsafe repeated effects/cost. Decide per operation whether repeated processing is acceptable. Disable early data where it is not, or combine supported early-data rejection/retry with an application design that enforces uniqueness at the side-effect boundary. A per-region replay cache is insufficient when another region can accept the same work. An idempotency key helps only when the server binds it to the operation and coordinates deduplication, result retention, and expiry.

### Session Resumption

A session ticket identifies retained or protected resumption state. In TLS 1.3, **NewSessionTicket** supplies an identity and parameters used with a derived resumption PSK. The client supplies a PSK binder to prove possession in its next ClientHello; a copied opaque ticket identifier alone is not the complete proof.

| Mechanism | State and behavior | Lifecycle concern |
|---|---|---|
| TLS 1.2 session ID | Server looks up retained session state | Sharing/expiry under load balancing |
| TLS 1.2 session ticket | Server can recover protected state from ticket | Protect and rotate ticket-encryption keys; ticket lifetime is separate from certificate lifetime |
| TLS 1.3 resumption PSK and ticket | Client/server derive PSK state; client proves possession via binder | Enforce ticket expiry and policy; PSK-only and PSK-with-DHE have different forward-secrecy properties |

**Forward secrecy** means later compromise of a long-term authentication key does not reveal recorded past traffic when ephemeral secrets were properly protected and erased. Ordinary certificate-authenticated ephemeral-DH TLS can provide this. TLS 1.3 PSK-only use and early data require separate analysis; “TLS 1.3 always has PFS” is too broad. Compromise of an endpoint holding live traffic secrets is another threat.

!!! info "External Resources"
    - [TLS 1.3 PSK Modes, Tickets, and Early Data](https://www.rfc-editor.org/rfc/rfc8446.html#section-2.2) (IETF)
    - [HTTP Early Data and Retry](https://www.rfc-editor.org/rfc/rfc8470.html) (IETF)

## Cipher Suites

In TLS 1.2 a cipher-suite name combines key establishment/authentication and record-protection choices. TLS 1.3 suite names specify AEAD and the key-derivation hash, while authentication and key exchange are negotiated separately.

### TLS 1.2 Cipher Suite Notation

```text
TLS_ECDHE_RSA_WITH_AES_256_GCM_SHA384
 |     |     |        |    |     |
 |     |     |        |    |     +-- PRF hash (used in key derivation)
 |     |     |        |    +-------- AEAD mode (GCM = authenticated encryption)
 |     |     |        +------------- Bulk cipher and key size
 |     |     +---------------------- Authentication algorithm (certificate type)
 |     +---------------------------- Key exchange algorithm
 +---------------------------------- Protocol
```

### TLS 1.3 Cipher Suite Notation (Simplified)

TLS 1.3 decouples key exchange from cipher suite. Only AEAD ciphers remain:

```text
TLS_AES_256_GCM_SHA384        (AES-256 in GCM mode, SHA-384 for HKDF)
TLS_AES_128_GCM_SHA256        (AES-128 in GCM mode, SHA-256 for HKDF)
TLS_CHACHA20_POLY1305_SHA256  (ChaCha20-Poly1305, SHA-256 for HKDF)
```

Key exchange is negotiated separately via `supported_groups` and `key_share` extensions.

### Recommended Cipher Suites

**TLS 1.3 (all are acceptable):**

- `TLS_AES_256_GCM_SHA384`
- `TLS_CHACHA20_POLY1305_SHA256`
- `TLS_AES_128_GCM_SHA256`

**TLS 1.2 examples for an ECDHE + AEAD policy:**

- `TLS_ECDHE_ECDSA_WITH_AES_256_GCM_SHA384`
- `TLS_ECDHE_RSA_WITH_AES_256_GCM_SHA384`
- `TLS_ECDHE_ECDSA_WITH_CHACHA20_POLY1305_SHA256`
- `TLS_ECDHE_RSA_WITH_CHACHA20_POLY1305_SHA256`
- `TLS_ECDHE_ECDSA_WITH_AES_128_GCM_SHA256`
- `TLS_ECDHE_RSA_WITH_AES_128_GCM_SHA256`

**Must disable:**

- Static RSA key transport (no forward secrecy); `ECDHE_RSA` uses RSA authentication and is a different construction
- Anything with `CBC` mode (padding oracle attacks)
- `RC4`, `3DES`, `DES`, `NULL`, `EXPORT` ciphers
- `MD5` or `SHA-1` for authentication
- Static `DH` or `ECDH` (non-ephemeral)

### Key Exchange Groups

| Group | Type | Security | Notes |
|-------|------|----------|-------|
| **X25519** | ECDH (Curve25519) | ~128-bit | Common implementation choice; platform/policy support varies |
| **P-256 (secp256r1)** | ECDH (NIST curve) | ~128-bit | FIPS-approved; widely supported |
| **P-384 (secp384r1)** | ECDH (NIST curve) | ~192-bit | Higher security margin; slower |
| **X448** | ECDH (Curve448) | ~224-bit | Higher security; less common |
| **ffdhe2048** | Finite-field DH | ~112-bit | Minimum acceptable DH; prefer ECDH |
| **X25519MLKEM768** | Hybrid ECDHE + ML-KEM | Combines classical and post-quantum shared-secret establishment | TLS 1.3 group defined in RFC 10024 (August 2026); verify implementation support |

ML-KEM is a key-encapsulation mechanism used to establish shared secret material. A hybrid group combines it with a classical exchange; it does not by itself change certificate authentication into a post-quantum signature scheme. Pre-standard Kyber Draft00 groups are not interchangeable with the standardized group.

!!! info "External Resources"
    - [Hybrid ECDHE–ML-KEM Groups — RFC 10024, August 2026](https://www.rfc-editor.org/rfc/rfc10024.html) (IETF)

For cipher algorithm details (AES, ChaCha20, RSA, ECC), see [Cryptographic Fundamentals](../crypto-identity/fundamentals.md).

## Public Key Infrastructure (PKI)

PKI is the framework of trust that makes TLS authentication work. It binds public keys to identities through digital certificates.

### Certificate Chain of Trust

```text
Root CA (self-signed, in browser/OS trust store)
  └── Intermediate CA (signed by Root CA)
        └── End-entity certificate (signed by Intermediate CA, your server's cert)
```

- **Trust anchors** come from the verifier's configured trust store; browser, OS, private PKI, and application stores can differ
- **Intermediate CAs** issue certificates subject to their constraints; compromise requires rejecting affected chains and investigating/reissuing affected credentials
- Servers normally send the leaf and needed intermediates; the verifier independently establishes trust in its configured anchor, not in an arbitrary transmitted root
- Client validates each signature up the chain to a trusted root

### Certificate Types

| Type | Validation | What CA verifies | Visual indicator |
|------|-----------|-----------------|-----------------|
| **DV** (Domain Validation) | Commonly automated | Domain control under CA policy | UI varies; does not prove a site is benign |
| **OV** (Organisation Validation) | CA validation policy | Domain control and organization identity | Details depend on client UI |
| **EV** (Extended Validation) | Additional CA requirements | Domain and organization verification under EV rules | Does not authorize the site's business actions |
| **Wildcard** | Profile-dependent | A permitted `*.example.com` pattern normally covers one label, not the apex | Does not imply unrestricted naming or EV issuance |
| **SAN / Multi-domain** | Any level | Multiple distinct domains in one certificate | Depends on DV/OV/EV |

**Self-signed certificates:**

- A self-signature does not create trust. A client can explicitly provision/pin an appropriate certificate or private CA for a controlled deployment.
- A private CA-signed leaf is different from a self-signed leaf; both need explicit trust configuration.
- General public browser clients normally need a chain accepted by their public trust policy; disabling validation is not an internal-PKI solution.

### X.509 Certificate Fields

| Field | Purpose | Example |
|-------|---------|---------|
| **Subject** | Entity the certificate identifies | `CN=example.com, O=Example Inc` |
| **Issuer** | CA that signed the certificate | The actual issuing intermediate distinguished name |
| **Serial Number** | Unique identifier from the CA | `0x0A01...` |
| **Not Before / Not After** | Validity period | Actual signed timestamps; lifetime depends on CA/profile |
| **Public Key** | Subject's public key | RSA 2048-bit or ECDSA P-256 |
| **Subject Alternative Name (SAN)** | Presented DNS/IP/service identifiers under the profile | `DNS:example.com, DNS:www.example.com` |
| **Key Usage** | Permitted operations | `Digital Signature, Key Encipherment` |
| **Extended Key Usage** | Specific purposes | `TLS Web Server Authentication` |
| **Authority Information Access** | Issuer access information, possibly OCSP | Not every current certificate contains an OCSP URL |
| **CRL Distribution Points** | URL of CRL | `http://crl.example.com/ca.crl` |
| **SCT** | Signed Certificate Timestamp (CT proof) | Embedded or via TLS extension |

### Service Identity Validation

A trusted chain proves a CA-certified key binding under constraints; it does not mean the certificate matches the service requested. The client obtains its **reference identity** from trusted connection configuration, such as `api.example`, then checks the appropriate SAN identity. A certificate valid for `attacker.example` must fail that check even if its CA is trusted and its signature verifies. Common Name fallback is not the current RFC 9525 service-identity rule.

The verifier also checks validity times, certificate usage/constraints, configured trust and revocation policy. The handshake verifies possession of the private key through the protocol's proof. These are different checks: displaying a certificate is not possession of its private key, and accepting an arbitrary root sent by a peer does not establish trust.

```bash
# OpenSSL 3.x diagnostic example, only against an authorized lab endpoint.
# Connect by address, send SNI for virtual hosting, and validate reference identity.
openssl s_client -connect 192.0.2.80:443 -servername api.example \
  -verify_hostname api.example -verify_return_error -CAfile lab-ca.pem </dev/null
# A chain trusted by lab-ca.pem for attacker.example must still fail hostname checking.
# SNI selects a server configuration; -servername alone is not hostname validation.
```

!!! info "External Resources"
    - [TLS Service Identity and SAN Matching](https://www.rfc-editor.org/rfc/rfc9525.html) (IETF)
    - [OpenSSL s_client Verification Options](https://docs.openssl.org/3.5/man1/openssl-s_client/) (OpenSSL)

### Certificate Lifecycle

1. **Generate key pair** on the server (never transmit private key)
2. **Create CSR** containing public key and subject information
3. **Submit to CA** for validation (DV: automated; OV/EV: manual review)
4. **CA signs** and returns the certificate
5. **Deploy** certificate + intermediate chain on the server
6. **Monitor** expiry dates and automate renewal (ACME/certbot)
7. **Rotate** before expiry; overlap old and new certs during rollout
8. **Revoke** if private key is compromised, then reissue

## Certificate Revocation

Revocation marks a certificate unusable before its signed expiry. The issuer publishing revocation does not guarantee that every client immediately learns or enforces it. The client's mechanism, data age, supported extensions, trust policy, and failure behavior determine acceptance.

| Mechanism | Information and check | Dependency or limitation |
|---|---|---|
| CRL | Issuer-signed list of revoked certificate identifiers | Freshness, distribution, signature validation, and verifier policy |
| OCSP | Signed status for a certificate from an authorized responder | Response validity/freshness and handling of revoked, unknown, or unavailable status |
| OCSP stapling | Server supplies a cached signed OCSP response in the handshake | Must be supported by CA/certificate/server/client; old responses cannot be reused indefinitely |
| Must-Staple | Certificate extension requests required status behavior | Enforcement/support vary; not a universal strongest control |
| Browser-distributed revocation data, including CRLite | Client obtains maintained revocation information outside each connection | Browser-specific coverage, update cadence, and failure policy |

**Soft-fail** can permit a connection when status cannot be checked; **hard-fail** denies without required evidence. Those are deployment choices, not fixed properties of all CRL or OCSP clients. A cached good response can remain acceptable for its defined interval after an incident. Certificate expiry and revocation are separate clocks.

```text
10:00  Client caches an authorized good-status response, valid until 10:30
10:05  Certificate revoked after key compromise
10:06  Client without newer rejection information may still use cached evidence
# Illustrative timing, not a universal OCSP lifetime or browser guarantee.
```

**Current operational example:** Let's Encrypt ended its OCSP service on August 6, 2025 and publishes revocation via CRLs. An OCSP-stapling requirement cannot be applied unconditionally to every certificate from every issuer. CRLite is deployed in Firefox desktop rather than merely an experiment; use the current browser's documented behavior instead of assuming a common revocation implementation across clients.

On compromise, stop unauthorized key use, replace affected key/certificates, publish revocation, and verify consumer enforcement. Shorter certificate validity limits some stale acceptance but does not replace an incident response path or terminate existing application sessions.

!!! info "External Resources"
    - [Certificate Revocation Lists and Validation](https://www.rfc-editor.org/rfc/rfc5280.html) (IETF)
    - [Let's Encrypt OCSP End of Life, August 2025](https://letsencrypt.org/2025/08/06/ocsp-service-has-reached-end-of-life/) (ISRG)
    - [CRLite in Firefox, August 2025](https://hacks.mozilla.org/2025/08/crlite-fast-private-and-comprehensive-certificate-revocation-checking-in-firefox/) (Mozilla)

## Certificate Transparency (CT)

CT is a public, append-only log system for submitted certificates/precertificates; public trust programs define which issuance must be logged. It enables domain owners and the public to detect mis-issued or unauthorized certificates.

**How it works:**

1. CA submits a pre-certificate to one or more CT logs before issuance
2. CT log returns a **Signed Certificate Timestamp (SCT)** - a promise to include the certificate
3. SCT is embedded in the certificate, delivered via TLS extension, or via OCSP stapling
4. Clients with a CT policy enforce their required SCT/log conditions; private PKI and non-browser TLS clients can have different policies
5. **Monitors** watch CT logs for certificates issued for their domains
6. **Auditors** verify that logs are append-only and consistent

**Why it matters:**

- Monitoring can detect unexpected issuance; log inclusion does not block an attacker from using a certificate and does not replace revocation
- Domain owners can monitor for unauthorized certificate issuance using tools like crt.sh, Certspotter, or Facebook CT monitoring
- Chrome requires all publicly-trusted certificates to have CT since April 2018

**CT log search (results are issued names, not proof of current live services):**

```bash
# Search for certificates issued for a domain
curl -s "https://crt.sh/?q=example.com&output=json" | jq '.[].common_name'
```

!!! info "External Resources"
    - [Certificate Transparency - Google](https://certificate.transparency.dev/) (Google)
    - [crt.sh - Certificate Search](https://crt.sh/) (Sectigo)
    - [RFC 6962 - Certificate Transparency](https://datatracker.ietf.org/doc/html/rfc6962) (IETF)

## HSTS (HTTP Strict Transport Security)

HSTS is an HTTP response header that instructs browsers to only connect to the site over HTTPS for a specified duration. It prevents supported browsers from using HTTP for remembered hosts. It does not fix XSS, stolen sessions, untrusted certificates, or every protocol downgrade.

**Header syntax:**

```http
Strict-Transport-Security: max-age=31536000; includeSubDomains; preload
```

| Directive | Purpose |
|-----------|---------|
| `max-age` | Duration (in seconds) the browser remembers to use HTTPS only. 31536000 = 1 year. |
| `includeSubDomains` | Applies HSTS to all subdomains. Required for preload. |
| `preload` | Signals intent to be included in browser HSTS preload lists. |

**How HSTS protects:**

1. User types `http://example.com` or clicks an HTTP link
2. Without HSTS: browser sends HTTP request; server redirects to HTTPS (but the first request was cleartext - attackable)
3. With HSTS: browser internally upgrades to HTTPS before sending any request (no cleartext exposure)

**HSTS preload list:**

- Browsers ship with a hardcoded list of domains that must always use HTTPS
- Eliminates the trust-on-first-use (TOFU) problem: even the very first visit is HTTPS
- Submit at [hstspreload.org](https://hstspreload.org/)
- Removal is possible but delayed by list/browser-update propagation; plan it as a significant deployment commitment rather than an immediate rollback
- Requirements: valid certificate, redirect HTTP to HTTPS, serve HSTS header on the HTTPS root domain with `max-age >= 31536000`, `includeSubDomains`, and `preload`

**Deployment steps:**

1. Ensure all subdomains support HTTPS (including any internal ones)
2. Add `Strict-Transport-Security: max-age=300` (5 min) and test
3. Increase `max-age` gradually: 1 day, 1 week, 1 month, 1 year
4. Add `includeSubDomains` when all subdomains are ready
5. Add `preload` and submit to hstspreload.org

**Risks:**

- Premature deployment locks out HTTP-only subdomains
- Expired or misconfigured certificates become site-breaking (no fallback to HTTP)
- Preload removal is slow; commit only when certain

## SNI, ESNI, and ECH

### SNI (Server Name Indication)

- TLS extension that sends the requested hostname in the ClientHello (cleartext)
- Allows multiple HTTPS sites on a single IP address (virtual hosting)
- **Privacy problem**: passive observers (ISPs, firewalls) can see which site you're connecting to even though the traffic is encrypted
- Used by censorship systems to block specific domains

### ECH (Encrypted Client Hello)

ECH encrypts a ClientHelloInner containing the actual server name and sensitive parameters, while a ClientHelloOuter remains visible for routing and compatibility. It is not encryption of every observable connection field: destination IP, timing, sizes, and the outer handshake remain visible.

Clients obtain a suitable server ECH configuration through supported provisioning, commonly HTTPS/SVCB DNS records. Encrypted DNS can additionally hide the name on the client-to-resolver hop; it is a separate privacy control, not an intrinsic requirement that makes ECH itself possible. Plaintext DNS can reveal the intended name even when the subsequent inner hello is encrypted.

When ECH is rejected, the outer connection cannot carry application data; an authenticated retry can obtain updated configuration. Support, retry, and deployment behavior must be checked in the actual client/server. Test actual negotiation instead of relying on an old browser-version/feature-flag statement. The earlier ESNI proposal is not equivalent to ECH.

!!! info "External Resources"
    - [Encrypted Client Hello — RFC 9849, March 2026](https://www.rfc-editor.org/rfc/rfc9849.html) (IETF)
    - [ECH Configuration Discovery](https://www.rfc-editor.org/rfc/rfc9848.html) (IETF)

## TLS Attacks and Threats

### Protocol-Level Attacks

| Attack | Year | Target | How it works | Impact | Mitigation |
|--------|------|--------|-------------|--------|-----------|
| **POODLE** | 2014 | SSL 3.0 CBC padding | Exploits non-deterministic CBC padding in SSL 3.0 to decrypt one byte per 256 requests | Decrypt session cookies | Disable SSL 3.0 entirely |
| **BEAST** | 2011 | TLS 1.0 CBC IV | Predictable IV in CBC mode allows chosen-plaintext attack | Decrypt HTTPS cookies | Upgrade to TLS 1.2+; use AEAD ciphers |
| **CRIME** | 2012 | TLS compression | Chosen-plaintext attack exploiting compression ratio changes to infer secret data | Steal session tokens | Disable TLS-level compression |
| **BREACH** | 2013 | HTTP compression | Same principle as CRIME but targets HTTP body compression | Steal CSRF tokens, secrets in responses | Disable HTTP compression for sensitive pages; add per-request randomness; separate secrets from user-controlled content |
| **HEARTBLEED** | 2014 | OpenSSL heartbeat extension (CVE-2014-0160) | Buffer over-read leaks up to 64KB of server memory per request | Leak private keys, session data, passwords from server memory | Patch OpenSSL; revoke and reissue all certificates; rotate all secrets |
| **FREAK** | 2015 | Export-grade RSA (512-bit) | Man-in-the-middle downgrades connection to export-grade RSA, which can be factored | Decrypt traffic | Remove export cipher suites |
| **Logjam** | 2015 | Weak DHE (512-1024 bit) | Downgrade to 512-bit DH; precomputation allows real-time decryption | Decrypt traffic | Use 2048-bit+ DH groups; prefer ECDHE |
| **ROBOT** | 2017 | RSA PKCS#1 v1.5 key exchange | Bleichenbacher-style padding oracle on RSA key exchange | Decrypt traffic, sign with server key | Disable RSA key exchange; use ECDHE |
| **Raccoon** | 2020 | DH(E) key exchange timing | Timing side-channel in DH key processing | Recover pre-master secret (theoretical) | Use TLS 1.3; constant-time DH implementations |
| **ALPACA** | 2021 | Cross-protocol confusion | Redirect TLS traffic to a different application protocol on the same server (FTP, SMTP) | Steal cookies, execute XSS | Use ALPN; isolate services on different certificates |

### Implementation Attacks

| Attack | Target | How it works | Mitigation |
|--------|--------|-------------|-----------|
| **Certificate forgery** | CA trust model | Compromised or rogue CA issues certs for arbitrary domains (DigiNotar 2011) | Certificate Transparency; monitor CT logs; reduce trusted CAs |
| **Null byte injection** | Certificate parsing | Embed `\0` in CN (e.g., `evil.com\0.example.com`) to bypass validation | Modern libraries fixed; keep TLS libraries updated |
| **Triple Handshake** | TLS renegotiation | Synchronise two TLS sessions to relay authentication | Extended Master Secret (RFC 7627) for affected TLS 1.2 sessions; TLS 1.3 uses a different transcript-bound design |
| **Lucky Thirteen** | CBC timing | Timing differences in CBC MAC verification reveal plaintext | Constant-time implementations; use AEAD ciphers |
| **Sweet32** | 3DES/Blowfish (64-bit blocks) | Birthday attack after ~32GB of data in a single session | Disable 64-bit block ciphers; use AES |

### Operational Attacks

| Attack | How it works | Mitigation |
|--------|-------------|-----------|
| **SSL stripping** | Attacker intercepts HTTP-to-HTTPS redirect and keeps victim on HTTP | HSTS with preload; never serve sensitive content over HTTP |
| **TLS interception proxy** | Corporate/government proxy terminates TLS, inspects traffic, re-encrypts with its own CA | Pin certificates in sensitive apps; detect proxy CAs; use client certificate auth |
| **Downgrade attack** | Active attacker modifies ClientHello to remove strong cipher suites | Enforce protocol policy; supported downgrade signaling and authenticated transcript checks; do not disable checks on failure |
| **Key compromise** | Attacker obtains server private key | PFS (ephemeral keys) protects past sessions; revoke cert, reissue, rotate |
| **Replay attack (0-RTT)** | Attacker replays captured 0-RTT early data | Disable early data for replay-unsafe operations; coordinate application deduplication where needed |

## TLS Fingerprinting

TLS fingerprinting supplies a heuristic about client behavior based on the combination of parameters in the ClientHello message.

### JA3 / JA3S

- **JA3** fingerprints the client: hashes the TLS version, cipher suites, extensions, elliptic curves, and point formats from ClientHello
- **JA3S** fingerprints the server: hashes the ServerHello response
- Output is an MD5 hash (e.g., `e7d705a3286e19ea42f587b344ee6865`)
- Used to detect malware C2, identify bots, and profile clients without decrypting traffic

### JA4+ (Successor to JA3)

- More granular fingerprinting with human-readable components
- Includes JA4 (TLS client), JA4S (server), JA4H (HTTP), JA4X (X.509), JA4SSH (SSH)
- Better at distinguishing clients and more resistant to randomisation

### Defensive Uses

- Detect known-bad TLS fingerprints (malware, vulnerability scanners)
- Identify clients that claim to be browsers but have non-browser fingerprints
- Correlate observations with endpoint, DNS, and server context; shared libraries and deliberate mimicry prevent treating a fingerprint as unique identity

### Evasion

- Attackers randomise ClientHello parameters to avoid fingerprint-based detection
- TLS 1.3 reduces fingerprintable surface by encrypting more of the handshake
- Some C2 frameworks (Cobalt Strike, Sliver) support custom TLS profiles to mimic legitimate browsers

## TLS Inspection and Interception

Corporate environments often deploy TLS inspection proxies (Palo Alto, Zscaler, Blue Coat) that perform man-in-the-middle decryption.

**How it works:**

1. Proxy terminates the client's TLS connection using a corporate CA certificate installed on all managed devices
2. Proxy inspects the plaintext traffic (DLP, IDS, malware scanning)
3. Proxy establishes a new TLS connection to the destination server
4. Client sees a certificate signed by the corporate CA, not the real server certificate

**Security implications:**

- Breaks end-to-end encryption; introduces a single point where all traffic is visible
- Corporate CA signing-key compromise enables active impersonation to trusting clients; it does not by itself decrypt recorded ECDHE sessions. Compromise of the live proxy or traffic secrets can expose plaintext
- May break certificate pinning in applications
- Define inspection/exemption policy from application sensitivity, privacy, authentication compatibility, and organizational requirements
- Client cannot verify the real server's certificate (only the proxy can)

**Detection:**

- Compare certificate issuer (corporate CA vs expected public CA)
- Inspect the chain and trust context; absence of SCTs alone does not prove interception for private PKI
- Application-level certificate pinning will fail

## Mutual TLS (mTLS)

In standard TLS, only the server presents a certificate. In mTLS, the client also presents a certificate, providing mutual authentication.

For detailed coverage of mTLS including handshake flow, use cases, and operational considerations, see [mTLS](../authentication/mtls.md).

**Common use cases:**

- Service-to-service authentication in microservices (Istio, Linkerd)
- Zero trust network access (device certificates)
- Key-possession-based client authentication; token/API authorization remains a separate design choice
- B2B integrations

## Testing and Auditing

### Command-Line Tools

These OpenSSL 3.x examples are diagnostic templates for authorized endpoints. Supply the intended trust store; do not hide verification errors while claiming to validate a service.

```bash
# TLS 1.3 negotiation AND server identity validation.
openssl s_client -connect api.example:443 -servername api.example \
  -verify_hostname api.example -verify_return_error -tls1_3 </dev/null

# Repeat with -tls1_2 when testing supported compatibility policy.
# Display the presented chain; display alone is not a validation result.
openssl s_client -connect api.example:443 -servername api.example -showcerts </dev/null

# Inspect a saved fixture's validity, separately from live chain validation.
openssl x509 -in server.pem -noout -dates -ext subjectAltName

# Inspect stapling only when this certificate/issuer supports OCSP.
openssl s_client -connect api.example:443 -servername api.example \
  -verify_hostname api.example -verify_return_error -status </dev/null
```

`-cipher` controls pre-TLS-1.3 suites; OpenSSL uses `-ciphersuites` for TLS 1.3. A negotiation result proves only the offered path/client configuration. Repeat relevant positive and negative cases, including wrong hostname, untrusted CA, expired leaf, missing intermediate, and unsupported protocol.

!!! info "External Resources"
    - [OpenSSL TLS Diagnostic Client](https://docs.openssl.org/3.5/man1/openssl-s_client/) (OpenSSL)

### Online Tools

| Tool | URL | Purpose |
|------|-----|---------|
| **SSL Labs** | ssllabs.com/ssltest | Comprehensive TLS configuration grading (A+ to F) |
| **testssl.sh** | testssl.sh | Open-source CLI tool for TLS testing |
| **Mozilla Observatory** | observatory.mozilla.org | Tests HTTPS + security headers |
| **Hardenize** | hardenize.com | TLS, DNS, email security assessment |
| **crt.sh** | crt.sh | Certificate Transparency log search |

### What to Check

- [ ] TLS 1.3 supported; TLS 1.2 as fallback only
- [ ] No SSL 2.0, SSL 3.0, TLS 1.0, TLS 1.1
- [ ] Only AEAD cipher suites (GCM, ChaCha20-Poly1305)
- [ ] No static RSA key transport; key establishment and resumption match forward-secrecy policy
- [ ] Certificate chain complete and valid
- [ ] Certificate not expired; automated renewal in place
- [ ] For browser HTTPS, HSTS scope/lifetime fits a staged deployment and recovery plan
- [ ] Revocation mechanism matches issuer/client support; stapling only where applicable
- [ ] CT requirements satisfied where the client trust policy requires them
- [ ] No mixed content (HTTP resources on HTTPS pages)
- [ ] Redirect HTTP to HTTPS (301)
- [ ] Preload considered only after subdomain readiness and long-lived commitment are established

## Configuration Reference

### Deployment Policy and Nginx Example

Use a current maintained configuration generator for the actual server/library version and client compatibility requirements. Do not copy a legacy profile simply because an older matrix called it “Old.” RFC 9325 supplies protocol guidance; applicable compliance/platform policy may impose additional constraints.

```nginx
# Fragment for an existing HTTPS server, not a complete nginx.conf.
ssl_protocols TLSv1.2 TLSv1.3;
# ssl_ciphers governs TLS <= 1.2; TLS 1.3 suite control is library/version specific.
ssl_ciphers ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384;
ssl_certificate /etc/nginx/tls/fullchain.pem;
ssl_certificate_key /etc/nginx/tls/private-key.pem;
ssl_early_data off;  # Avoid accepting replayable requests without an explicit design.

# Start small; increase only after validating HTTPS and relevant subdomains.
add_header Strict-Transport-Security "max-age=300" always;
# OCSP stapling omitted: enable only if this issuer/certificate supports it.
# Configure backend certificate verification separately when proxying to HTTPS.
```

This fragment does not define session-ticket rotation, trust in upstreams, listeners, application authorization, or all compatibility settings. Changing a certificate does not necessarily end established TLS connections, resumption state, or application sessions; include each relevant lifecycle in a compromise/offboarding requirement.

!!! info "External Resources"
    - [Mozilla SSL Configuration Generator](https://ssl-config.mozilla.org/) (Mozilla)
    - [Mozilla Server Side TLS Guidelines](https://wiki.mozilla.org/Security/Server_Side_TLS) (Mozilla)
    - [Dutch NCSC - TLS Guidelines](https://english.ncsc.nl/publications/publications/2021/january/19/it-security-guidelines-for-transport-layer-security-2.1) (NCSC-NL)
    - [Qualys SSL Labs - SSL Server Test](https://www.ssllabs.com/ssltest/) (Qualys)
    - [RFC 8446 - TLS 1.3](https://datatracker.ietf.org/doc/html/rfc8446) (IETF)
    - [RFC 6797 - HSTS](https://datatracker.ietf.org/doc/html/rfc6797) (IETF)
    - [RFC 6962 - Certificate Transparency](https://datatracker.ietf.org/doc/html/rfc6962) (IETF)
    - [Bulletproof TLS and PKI](https://www.feistyduck.com/books/bulletproof-tls-and-pki/) (Ivan Ristic / Feisty Duck)

!!! info "External Resources"
    - [Secure TLS Deployment Recommendations](https://www.rfc-editor.org/rfc/rfc9325.html) (IETF)
    - [Extended Master Secret and Triple Handshake](https://www.rfc-editor.org/rfc/rfc7627.html) (IETF)
