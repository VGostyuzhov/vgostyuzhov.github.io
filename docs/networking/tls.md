# SSL/TLS

SSL (Secure Sockets Layer) and its successor TLS (Transport Layer Security) provide confidentiality, integrity, and authentication for network communication. SSL is deprecated; TLS 1.3 is the current standard.

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
| **OCSP** | Online Certificate Status Protocol | Real-time certificate revocation checking |
| **CRL** | Certificate Revocation List | Signed list of revoked certificates published by a CA |
| **CT** | Certificate Transparency | Public log infrastructure for auditing certificate issuance |
| **CA** | Certificate Authority | Entity that issues and signs digital certificates |
| **CSR** | Certificate Signing Request | Message sent to a CA to request a signed certificate |
| **SAN** | Subject Alternative Name | X.509 extension listing additional hostnames a certificate covers |
| **DV/OV/EV** | Domain/Organisation/Extended Validation | Certificate validation levels with increasing assurance |
| **ACME** | Automatic Certificate Management Environment | Protocol for automated certificate issuance (Let's Encrypt) |
| **HPKP** | HTTP Public Key Pinning | Deprecated header that pinned expected certificate keys (replaced by CT) |
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
| TLS 1.3 | 2018 | Current | Removed all legacy ciphers; 1-RTT handshake; mandatory PFS; no RSA key transport |

**What TLS 1.3 removed:**

- RSA key exchange (no PFS)
- CBC mode ciphers (padding oracle attacks)
- RC4, 3DES, MD5, SHA-1 in cipher suites
- Compression (CRIME attack)
- Renegotiation
- Static DH/ECDH (non-ephemeral)
- Custom DHE groups (must use well-known groups)
- ChangeCipherSpec message

## TLS Architecture

TLS operates between the transport layer (TCP) and the application layer. It consists of two sub-protocols:

**Record Protocol:**

- Fragments application data into records (max 16 KB)
- Compresses (TLS 1.2 only; removed in 1.3), encrypts, and authenticates each record
- Adds a 5-byte header: content type, protocol version, length

**Handshake Protocol:**

- Negotiates cipher suite, authenticates parties, establishes session keys
- Uses asymmetric cryptography to derive shared symmetric keys
- All subsequent data encrypted with the negotiated symmetric cipher

## TLS Handshakes

### TLS 1.2 Handshake (2-RTT)

```
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

```
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
| Key exchange | RSA or DHE/ECDHE | ECDHE or DHE only (PFS mandatory) |
| Server authentication | After handshake completes | During handshake (encrypted) |
| Cipher negotiation | Complex (key exchange + cipher + MAC) | Simplified (AEAD only) |
| Encrypted extensions | No | Yes (server extensions hidden from passive observers) |

### 0-RTT Resumption (TLS 1.3)

- Client sends early data alongside ClientHello using a PSK from a previous session
- Reduces latency to zero round trips for repeat connections
- **Replay risk**: 0-RTT data has no anti-replay protection at the TLS layer; the server may receive the same data twice
- Only safe for idempotent requests (GET); never for state-changing operations (POST, PUT, DELETE)
- Servers should implement application-level replay protection or disable 0-RTT

### Session Resumption

| Mechanism | How it works | Notes |
|-----------|-------------|-------|
| **Session IDs** (TLS 1.2) | Server stores session state; client sends ID to resume | Requires server-side storage; breaks with load balancers unless shared |
| **Session Tickets** (TLS 1.2) | Server encrypts session state into a ticket sent to client | Stateless for server; ticket encryption key must be rotated |
| **PSK** (TLS 1.3) | Server sends a PSK identifier after handshake; client uses it for resumption | Replaces both session IDs and tickets; supports 0-RTT |

## Cipher Suites

A cipher suite defines the algorithms used for key exchange, authentication, bulk encryption, and message integrity.

### TLS 1.2 Cipher Suite Notation

```
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

```
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

**TLS 1.2 (use only these):**

- `TLS_ECDHE_ECDSA_WITH_AES_256_GCM_SHA384`
- `TLS_ECDHE_RSA_WITH_AES_256_GCM_SHA384`
- `TLS_ECDHE_ECDSA_WITH_CHACHA20_POLY1305_SHA256`
- `TLS_ECDHE_RSA_WITH_CHACHA20_POLY1305_SHA256`
- `TLS_ECDHE_ECDSA_WITH_AES_128_GCM_SHA256`
- `TLS_ECDHE_RSA_WITH_AES_128_GCM_SHA256`

**Must disable:**

- Anything with `RSA` key exchange (no PFS)
- Anything with `CBC` mode (padding oracle attacks)
- `RC4`, `3DES`, `DES`, `NULL`, `EXPORT` ciphers
- `MD5` or `SHA-1` for authentication
- Static `DH` or `ECDH` (non-ephemeral)

### Key Exchange Groups

| Group | Type | Security | Notes |
|-------|------|----------|-------|
| **X25519** | ECDH (Curve25519) | ~128-bit | Fastest; default in most modern implementations |
| **P-256 (secp256r1)** | ECDH (NIST curve) | ~128-bit | FIPS-approved; widely supported |
| **P-384 (secp384r1)** | ECDH (NIST curve) | ~192-bit | Higher security margin; slower |
| **X448** | ECDH (Curve448) | ~224-bit | Higher security; less common |
| **ffdhe2048** | Finite-field DH | ~112-bit | Minimum acceptable DH; prefer ECDH |
| **X25519Kyber768** | Hybrid post-quantum | Quantum-resistant | TLS 1.3 draft; Chrome/Firefox experimental support |

For cipher algorithm details (AES, ChaCha20, RSA, ECC), see [Cryptographic Fundamentals](../crypto-identity/fundamentals.md).

## Public Key Infrastructure (PKI)

PKI is the framework of trust that makes TLS authentication work. It binds public keys to identities through digital certificates.

### Certificate Chain of Trust

```
Root CA (self-signed, in browser/OS trust store)
  └── Intermediate CA (signed by Root CA)
        └── End-entity certificate (signed by Intermediate CA, your server's cert)
```

- **Root CAs** are pre-installed in OS and browser trust stores (~150 trusted roots)
- **Intermediate CAs** issue end-entity certificates; if compromised, only the intermediate needs revocation
- Servers must send the full chain (end-entity + intermediates) but NOT the root
- Client validates each signature up the chain to a trusted root

### Certificate Types

| Type | Validation | What CA verifies | Visual indicator |
|------|-----------|-----------------|-----------------|
| **DV** (Domain Validation) | Automated | Domain control (DNS or HTTP challenge) | Padlock only |
| **OV** (Organisation Validation) | Manual | Domain control + legal existence of organisation | Padlock (org name in cert details) |
| **EV** (Extended Validation) | Extensive | Domain + org + legal jurisdiction + physical address | Padlock (org name in cert details; no longer a green bar) |
| **Wildcard** | Any level | Covers `*.example.com` (one level of subdomain) | Depends on DV/OV/EV |
| **SAN / Multi-domain** | Any level | Multiple distinct domains in one certificate | Depends on DV/OV/EV |

**Self-signed certificates:**

- Not signed by a trusted CA; browser shows warning
- Acceptable for internal development, testing, service mesh (with custom CA)
- Never acceptable for public-facing services

### X.509 Certificate Fields

| Field | Purpose | Example |
|-------|---------|---------|
| **Subject** | Entity the certificate identifies | `CN=example.com, O=Example Inc` |
| **Issuer** | CA that signed the certificate | `CN=Let's Encrypt Authority X3` |
| **Serial Number** | Unique identifier from the CA | `0x0A01...` |
| **Not Before / Not After** | Validity period | 90 days for Let's Encrypt |
| **Public Key** | Subject's public key | RSA 2048-bit or ECDSA P-256 |
| **Subject Alternative Name (SAN)** | Additional hostnames | `DNS:example.com, DNS:www.example.com` |
| **Key Usage** | Permitted operations | `Digital Signature, Key Encipherment` |
| **Extended Key Usage** | Specific purposes | `TLS Web Server Authentication` |
| **Authority Information Access** | OCSP responder URL, CA issuer URL | `http://ocsp.letsencrypt.org` |
| **CRL Distribution Points** | URL of CRL | `http://crl.example.com/ca.crl` |
| **SCT** | Signed Certificate Timestamp (CT proof) | Embedded or via TLS extension |

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

When a private key is compromised or a certificate is mis-issued, it must be revoked before expiry. Revocation is one of the hardest problems in PKI.

### CRL (Certificate Revocation List)

- CA publishes a signed list of revoked certificate serial numbers
- Clients download and cache the full CRL
- **Problems**: CRLs grow large (megabytes for major CAs); stale cache means delayed revocation; download failures mean soft-fail (accept unverified cert)
- CRL Distribution Point URL is embedded in the certificate

### OCSP (Online Certificate Status Protocol)

- Client sends the certificate serial number to the CA's OCSP responder
- Responder returns signed status: `good`, `revoked`, or `unknown`
- **Advantages over CRL**: smaller response, per-certificate check, more timely
- **Problems**: privacy leak (CA sees which sites you visit); availability dependency (if OCSP responder is down, browsers soft-fail); adds latency to every connection

### OCSP Stapling

- Server queries the OCSP responder periodically and caches the signed response
- Server includes ("staples") the OCSP response in the TLS handshake
- Client verifies the stapled response (signed by CA, so server cannot forge it)
- **Advantages**: no privacy leak, no client-side latency, no dependency on OCSP responder availability at connection time
- **OCSP Must-Staple**: X.509 extension (`1.3.6.1.5.5.7.1.24`) that tells clients to reject the certificate if no stapled response is provided; converts soft-fail to hard-fail

### CRLite (Emerging)

- Firefox experiment: compress all revocation data into a Bloom filter cascade
- Pushed to browsers via updates (like a CRL but much smaller)
- Eliminates both CRL download size and OCSP privacy/availability issues

### Revocation Comparison

| Property | CRL | OCSP | OCSP Stapling | CRLite |
|----------|-----|------|---------------|--------|
| **Timeliness** | Hours to days | Minutes | Minutes (cached by server) | Hours (push-based) |
| **Privacy** | No leak (local check) | CA sees all sites you visit | No leak | No leak |
| **Availability** | Cached locally | Depends on responder uptime | Server handles it | Browser update |
| **Bandwidth** | Large (full list) | Small (per-cert) | Small (stapled) | Small (compressed) |
| **Failure mode** | Soft-fail (most clients) | Soft-fail (most clients) | Hard-fail with Must-Staple | Hard-fail |

**Practical reality**: Most browsers soft-fail on revocation check failure, meaning a compromised cert may be accepted if the revocation infrastructure is unreachable. OCSP Must-Staple is the strongest currently deployable mitigation.

## Certificate Transparency (CT)

CT is a public, append-only log system that records all issued certificates. It enables domain owners and the public to detect mis-issued or unauthorized certificates.

**How it works:**

1. CA submits a pre-certificate to one or more CT logs before issuance
2. CT log returns a **Signed Certificate Timestamp (SCT)** - a promise to include the certificate
3. SCT is embedded in the certificate, delivered via TLS extension, or via OCSP stapling
4. Browsers (Chrome, Safari, others) require valid SCTs to trust certificates
5. **Monitors** watch CT logs for certificates issued for their domains
6. **Auditors** verify that logs are append-only and consistent

**Why it matters:**

- Detects rogue or compromised CAs issuing unauthorized certificates (DigiNotar 2011, Symantec issues)
- Domain owners can monitor for unauthorized certificate issuance using tools like crt.sh, Certspotter, or Facebook CT monitoring
- Chrome requires all publicly-trusted certificates to have CT since April 2018

**CT log search:**

```bash
# Search for certificates issued for a domain
curl -s "https://crt.sh/?q=example.com&output=json" | jq '.[].common_name'
```

!!! info "External Resources"
    - [Certificate Transparency - Google](https://certificate.transparency.dev/) (Google)
    - [crt.sh - Certificate Search](https://crt.sh/) (Sectigo)
    - [RFC 6962 - Certificate Transparency](https://datatracker.ietf.org/doc/html/rfc6962) (IETF)

## HSTS (HTTP Strict Transport Security)

HSTS is an HTTP response header that instructs browsers to only connect to the site over HTTPS for a specified duration. It prevents protocol downgrade attacks and cookie hijacking.

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
- **Irreversible in practice**: removing a domain from the preload list takes months and requires browser updates to propagate
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

- TLS 1.3 extension that encrypts the entire ClientHello (including SNI) using a public key published in DNS (HTTPS record)
- Replaces the earlier ESNI proposal (which only encrypted the SNI field)
- Requires DNS-over-HTTPS (DoH) or DNS-over-TLS (DoT) to prevent DNS-level snooping
- Supported in Firefox and Chrome (behind flags as of 2024); deployment depends on CDN/server support
- **Outer ClientHello** contains a dummy SNI (usually the CDN's name); **Inner ClientHello** contains the real SNI, encrypted

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
| **Triple Handshake** | TLS renegotiation | Synchronise two TLS sessions to relay authentication | Renegotiation Indication Extension (RFC 5746); use TLS 1.3 (no renegotiation) |
| **Lucky Thirteen** | CBC timing | Timing differences in CBC MAC verification reveal plaintext | Constant-time implementations; use AEAD ciphers |
| **Sweet32** | 3DES/Blowfish (64-bit blocks) | Birthday attack after ~32GB of data in a single session | Disable 64-bit block ciphers; use AES |

### Operational Attacks

| Attack | How it works | Mitigation |
|--------|-------------|-----------|
| **SSL stripping** | Attacker intercepts HTTP-to-HTTPS redirect and keeps victim on HTTP | HSTS with preload; never serve sensitive content over HTTP |
| **TLS interception proxy** | Corporate/government proxy terminates TLS, inspects traffic, re-encrypts with its own CA | Pin certificates in sensitive apps; detect proxy CAs; use client certificate auth |
| **Downgrade attack** | Active attacker modifies ClientHello to remove strong cipher suites | TLS_FALLBACK_SCSV (TLS 1.2); TLS 1.3 signs the handshake transcript including ClientHello |
| **Key compromise** | Attacker obtains server private key | PFS (ephemeral keys) protects past sessions; revoke cert, reissue, rotate |
| **Replay attack (0-RTT)** | Attacker replays captured 0-RTT early data | Limit 0-RTT to idempotent operations; implement application-level replay protection |

## TLS Fingerprinting

TLS fingerprinting identifies client software based on the unique combination of parameters in the ClientHello message.

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
- Track threat actors across infrastructure changes (C2 fingerprint remains stable)

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
- Corporate CA private key compromise allows decryption of all proxied traffic
- May break certificate pinning in applications
- Some applications (banking, healthcare) should be exempted from inspection
- Client cannot verify the real server's certificate (only the proxy can)

**Detection:**

- Compare certificate issuer (corporate CA vs expected public CA)
- Check for CT SCTs (proxy-issued certs won't have them)
- Application-level certificate pinning will fail

## Mutual TLS (mTLS)

In standard TLS, only the server presents a certificate. In mTLS, the client also presents a certificate, providing mutual authentication.

For detailed coverage of mTLS including handshake flow, use cases, and operational considerations, see [mTLS](../authentication/mtls.md).

**Common use cases:**

- Service-to-service authentication in microservices (Istio, Linkerd)
- Zero trust network access (device certificates)
- API authentication (stronger than API keys)
- B2B integrations

## Testing and Auditing

### Command-Line Tools

```bash
# Test TLS configuration and cipher suites
openssl s_client -connect example.com:443 -tls1_3
openssl s_client -connect example.com:443 -tls1_2

# Show certificate details
openssl s_client -connect example.com:443 </dev/null 2>/dev/null | openssl x509 -text -noout

# Check certificate expiry
openssl s_client -connect example.com:443 </dev/null 2>/dev/null | openssl x509 -noout -dates

# Check OCSP stapling
openssl s_client -connect example.com:443 -status </dev/null 2>/dev/null | grep -A 5 "OCSP Response"

# Test specific cipher suite
openssl s_client -connect example.com:443 -cipher ECDHE-RSA-AES256-GCM-SHA384

# Check certificate chain
openssl s_client -connect example.com:443 -showcerts </dev/null 2>/dev/null

# Enumerate supported ciphers
nmap --script ssl-enum-ciphers -p 443 example.com
```

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
- [ ] No RSA key exchange; ECDHE only
- [ ] Certificate chain complete and valid
- [ ] Certificate not expired; automated renewal in place
- [ ] HSTS header present with `max-age >= 31536000`
- [ ] OCSP stapling enabled
- [ ] Certificate Transparency SCTs present
- [ ] No mixed content (HTTP resources on HTTPS pages)
- [ ] Redirect HTTP to HTTPS (301)
- [ ] HSTS preload if public-facing

## Configuration Reference

### Mozilla Recommended Configurations

Mozilla maintains three configuration profiles:

| Profile | Target | Min TLS | Cipher Suites | Use Case |
|---------|--------|---------|---------------|----------|
| **Modern** | Recent clients only | TLS 1.3 | TLS 1.3 suites only | New services, APIs, internal |
| **Intermediate** | Broad compatibility | TLS 1.2 | ECDHE + AEAD only | Most public-facing services |
| **Old** | Legacy clients | TLS 1.0 | Includes CBC (no RC4/3DES) | Legacy systems only; avoid if possible |

Use the [Mozilla SSL Configuration Generator](https://ssl-config.mozilla.org/) to generate configs for Nginx, Apache, HAProxy, and others.

### Nginx Example (Intermediate)

```nginx
ssl_protocols TLSv1.2 TLSv1.3;
ssl_ciphers ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-CHACHA20-POLY1305:ECDHE-RSA-CHACHA20-POLY1305;
ssl_prefer_server_ciphers off;

ssl_certificate /etc/letsencrypt/live/example.com/fullchain.pem;
ssl_certificate_key /etc/letsencrypt/live/example.com/privkey.pem;

ssl_session_timeout 1d;
ssl_session_cache shared:SSL:10m;
ssl_session_tickets off;

ssl_stapling on;
ssl_stapling_verify on;
resolver 1.1.1.1 8.8.8.8 valid=300s;

add_header Strict-Transport-Security "max-age=31536000; includeSubDomains; preload" always;
```

!!! info "External Resources"
    - [Mozilla SSL Configuration Generator](https://ssl-config.mozilla.org/) (Mozilla)
    - [Mozilla Server Side TLS Guidelines](https://wiki.mozilla.org/Security/Server_Side_TLS) (Mozilla)
    - [Dutch NCSC - TLS Guidelines](https://english.ncsc.nl/publications/publications/2021/january/19/it-security-guidelines-for-transport-layer-security-2.1) (NCSC-NL)
    - [Qualys SSL Labs - SSL Server Test](https://www.ssllabs.com/ssltest/) (Qualys)
    - [RFC 8446 - TLS 1.3](https://datatracker.ietf.org/doc/html/rfc8446) (IETF)
    - [RFC 6797 - HSTS](https://datatracker.ietf.org/doc/html/rfc6797) (IETF)
    - [RFC 6962 - Certificate Transparency](https://datatracker.ietf.org/doc/html/rfc6962) (IETF)
    - [Bulletproof TLS and PKI](https://www.feistyduck.com/books/bulletproof-tls-and-pki/) (Ivan Ristic / Feisty Duck)
