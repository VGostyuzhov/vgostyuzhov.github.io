# Cryptographic Fundamentals

## Encryption vs Encoding vs Hashing vs Signing

| Primitive | Purpose | Reversible | Key Required | Example |
|-----------|---------|-----------|-------------|---------|
| **Encryption** | Confidentiality - protect data from unauthorized reading | Yes (with key) | Yes | AES-256-GCM, RSA-OAEP |
| **Encoding** | Data representation - convert format for transport or storage | Yes (no secret) | No | Base64, URL encoding, Hex |
| **Hashing** | Integrity - produce fixed-size fingerprint of data | No (one-way) | No | SHA-256, BLAKE3, bcrypt |
| **Signing** | Authentication + Integrity - prove origin and detect tampering | Verifiable (not reversible) | Yes (private key to sign, public to verify) | RSA-PSS, Ed25519, HMAC |
| **Obfuscation** | Obscure logic or data - not a security control | Varies | No | Code minification, XOR "encryption" |

**Critical distinction:** Encoding is not a security mechanism. Base64 is not encryption. Using the wrong primitive breaks the security property you need.

**Common misuse patterns:**

- Encoding passwords instead of hashing them (Base64 "encryption")
- Using MD5 or SHA-1 for password storage (use bcrypt/scrypt/Argon2)
- Encrypting data that only needs integrity checking (sign instead)
- Rolling custom cryptographic implementations instead of using vetted libraries

!!! info "External Resources"
    - [Crypto 101](https://www.crypto101.io/) (Crypto 101)
    - [Practical Cryptography for Developers](https://cryptobook.nakov.com/) (Nakov)
    - [OWASP Cryptographic Failures](https://owasp.org/Top10/A02_2021-Cryptographic_Failures/) (OWASP)

## Symmetric Encryption (AES, ChaCha20)

Symmetric encryption uses the same key for encryption and decryption. Fast, efficient, used for bulk data protection.

**AES (Advanced Encryption Standard):**

- Block cipher with 128-bit block size
- Key sizes: 128, 192, or 256 bits
- NIST standardised (FIPS 197); hardware acceleration on modern CPUs (AES-NI)
- Dominant standard for data encryption at rest and in transit

**ChaCha20:**

- Stream cipher designed by Daniel Bernstein
- 256-bit key, 96-bit nonce
- Software-optimised; faster than AES on devices without AES-NI (mobile, IoT)
- Used in TLS 1.3 (ChaCha20-Poly1305) and WireGuard

**Comparison:**

| Property | AES-256-GCM | ChaCha20-Poly1305 |
|----------|-------------|-------------------|
| Type | Block cipher + AEAD | Stream cipher + AEAD |
| Key size | 256 bits | 256 bits |
| Hardware accel | AES-NI (very fast) | No (but fast in software) |
| Nonce size | 96 bits (12 bytes) | 96 bits (12 bytes) |
| Authentication | GCM (GHASH) | Poly1305 |
| Primary use | TLS, disk encryption, cloud | TLS (mobile), VPN, messaging |

**Key management is harder than encryption itself.** AES-256 is unbreakable with current technology, but a leaked key makes it worthless.

!!! info "External Resources"
    - [NIST FIPS 197 - AES](https://csrc.nist.gov/publications/detail/fips/197/final) (NIST)
    - [ChaCha20 and Poly1305 - RFC 8439](https://datatracker.ietf.org/doc/html/rfc8439) (IETF)
    - [NIST SP 800-38D - GCM Mode](https://csrc.nist.gov/publications/detail/sp/800-38d/final) (NIST)

## Asymmetric Encryption (RSA, ECC/Ed25519)

Asymmetric encryption uses a key pair: public key encrypts (or verifies), private key decrypts (or signs). Slower than symmetric; used for key exchange, digital signatures, and identity.

**RSA:**

- Based on integer factorization difficulty
- Key sizes: 2048 bits (minimum), 3072 or 4096 recommended
- Operations: encrypt/decrypt (OAEP padding), sign/verify (PSS padding)
- Slow for bulk encryption; typically used to encrypt a symmetric session key

**ECC (Elliptic Curve Cryptography):**

- Based on elliptic curve discrete logarithm problem
- Smaller keys for equivalent security: 256-bit ECC is roughly equivalent to 3072-bit RSA
- Faster than RSA for signing; more efficient on constrained devices

**Ed25519 (EdDSA):**

- Specific curve (Curve25519) and signature algorithm
- 256-bit keys, 512-bit signatures
- Deterministic signing (no random nonce needed - avoids nonce reuse bugs)
- Used in SSH keys, TLS, DNSSEC, cryptocurrency

**Comparison:**

| Property | RSA-3072 | ECDSA P-256 | Ed25519 |
|----------|----------|-------------|---------|
| Key size | 3072 bits | 256 bits | 256 bits |
| Signature size | 384 bytes | 64 bytes | 64 bytes |
| Sign speed | Slow | Fast | Fast |
| Verify speed | Fast | Fast | Fast |
| Standards | FIPS, PKCS#1 | FIPS, NIST curves | IETF RFC 8032 |
| Post-quantum | Vulnerable | Vulnerable | Vulnerable |

**Protocol usage:**

- Asymmetric encryption establishes trust and exchanges symmetric keys
- TLS handshake: server proves identity with certificate (asymmetric), then both sides derive a symmetric session key
- SSH: server/client key exchange uses ECDH or Curve25519, then switch to AES/ChaCha20
- Perfect forward secrecy: each session uses ephemeral key pair, so compromising long-term key does not decrypt past sessions

!!! info "External Resources"
    - [RFC 8032 - Ed25519 and Ed448](https://datatracker.ietf.org/doc/html/rfc8032) (IETF)
    - [Elliptic Curve Cryptography - Cloudflare](https://blog.cloudflare.com/a-relatively-easy-to-understand-primer-on-elliptic-curve-cryptography/) (Cloudflare)
    - [NIST SP 800-56A - Key Establishment](https://csrc.nist.gov/publications/detail/sp/800-56a/rev-3/final) (NIST)

## Block vs Stream Ciphers

| Property | Block Cipher | Stream Cipher |
|----------|-------------|---------------|
| **Input** | Fixed-size blocks (e.g., 128 bits) | Continuous bit/byte stream |
| **Operation** | Encrypts one block at a time | XORs plaintext with keystream |
| **Mode of operation** | Required (ECB, CBC, CTR, GCM) | Built-in |
| **Parallelisable** | Depends on mode (CTR/GCM yes, CBC no) | Yes |
| **Error propagation** | Depends on mode | Single bit error affects single bit |
| **Examples** | AES, 3DES, Blowfish | ChaCha20, RC4 (broken), Salsa20 |

**Block cipher modes of operation:**

| Mode | Description | Security Notes |
|------|------------|---------------|
| **ECB** | Each block encrypted independently | Insecure - identical plaintext blocks produce identical ciphertext. Never use. |
| **CBC** | Each block XORed with previous ciphertext | Requires IV; padding oracle attacks possible (POODLE) |
| **CTR** | Counter encrypted, then XORed with plaintext | Parallelisable; nonce must never repeat |
| **GCM** | CTR mode + GHASH authentication | AEAD - provides encryption + integrity. Standard choice for AES. |
| **CCM** | CTR + CBC-MAC | AEAD; used in wireless (WPA2) |

**AEAD (Authenticated Encryption with Associated Data):**

- Combines encryption and authentication in a single operation
- Prevents tampering - any modification is detected
- GCM and ChaCha20-Poly1305 are the dominant AEAD constructions
- Always prefer AEAD over separate encrypt-then-MAC constructions

!!! info "External Resources"
    - [Block Cipher Modes - Wikipedia](https://en.wikipedia.org/wiki/Block_cipher_mode_of_operation) (Wikipedia)
    - [NIST SP 800-38A - Block Cipher Modes](https://csrc.nist.gov/publications/detail/sp/800-38a/final) (NIST)
    - [AES-GCM - NIST SP 800-38D](https://csrc.nist.gov/publications/detail/sp/800-38d/final) (NIST)

## Hashing Functions (SHA-256, BLAKE, MD5)

A cryptographic hash function takes arbitrary input and produces a fixed-size output (digest). Properties: deterministic, fast, pre-image resistant, second pre-image resistant, collision resistant.

| Hash | Output Size | Status | Use Cases |
|------|------------|--------|-----------|
| **MD5** | 128 bits | Broken (collisions found) | Legacy checksums only; never for security |
| **SHA-1** | 160 bits | Broken (SHAttered collision) | Deprecated; still found in legacy Git, old certs |
| **SHA-256** | 256 bits | Secure | File integrity, certificate signing, blockchain, general purpose |
| **SHA-3 (Keccak)** | 224-512 bits | Secure | Alternative to SHA-2; different internal structure (sponge) |
| **BLAKE2** | Up to 512 bits | Secure | Faster than SHA-256 in software; file hashing, KDF |
| **BLAKE3** | 256 bits | Secure | Extremely fast; parallelisable; BLAKE2 successor |

**Password hashing (different from general hashing):**

- General hashes are designed to be fast; password hashes are designed to be slow
- Slow hashing makes brute-force attacks computationally expensive

| Algorithm | Design | Notes |
|-----------|--------|-------|
| **bcrypt** | Blowfish-based, configurable cost factor | Widely used; limited to 72-byte input |
| **scrypt** | Memory-hard | Resistant to GPU/ASIC attacks |
| **Argon2** | Memory-hard, configurable parallelism | Winner of Password Hashing Competition; recommended for new systems |

**Security applications of hashing:**

- File integrity verification (checksums, SBOM)
- Malware fingerprinting (sample identification)
- Digital signature input (hash-then-sign)
- Commitment schemes (prove knowledge without revealing)
- Deduplication (content-addressable storage)

!!! info "External Resources"
    - [NIST FIPS 180-4 - SHA Standard](https://csrc.nist.gov/publications/detail/fips/180/4/final) (NIST)
    - [Password Hashing Competition](https://www.password-hashing.net/) (PHC)
    - [BLAKE3 Specification](https://github.com/BLAKE3-team/BLAKE3-specs/blob/master/blake3.pdf) (BLAKE3 Team)

## Message Authentication Codes (MAC, HMAC)

A MAC verifies both the integrity and authenticity of a message using a shared secret key. Unlike a hash, a MAC proves the message was created by someone who knows the key.

**HMAC (Hash-based MAC):**

- Construction: `HMAC(K, M) = H((K' XOR opad) || H((K' XOR ipad) || M))`
- Uses any cryptographic hash function (SHA-256 most common)
- Secure against length extension attacks (unlike plain `H(key || message)`)
- Used in: JWT signing (HS256), API authentication, TLS record integrity

**Other MAC constructions:**

| MAC | Based On | Notes |
|-----|----------|-------|
| **HMAC-SHA256** | SHA-256 | Standard choice for API auth and token signing |
| **CMAC** | Block cipher (AES) | Used in IEEE 802.11 (Wi-Fi) |
| **Poly1305** | Polynomial evaluation | Used with ChaCha20 for AEAD |
| **GMAC** | GHASH (GCM without encryption) | Part of AES-GCM |

**MAC vs Digital Signature:**

| Property | MAC (HMAC) | Digital Signature |
|----------|-----------|-------------------|
| Key type | Shared secret (symmetric) | Key pair (asymmetric) |
| Who can verify | Anyone with the shared key | Anyone with the public key |
| Non-repudiation | No (both parties hold key) | Yes (only signer holds private key) |
| Speed | Fast | Slower |
| Use case | API auth, session tokens, internal integrity | Certificates, code signing, legal documents |

!!! info "External Resources"
    - [RFC 2104 - HMAC](https://datatracker.ietf.org/doc/html/rfc2104) (IETF)
    - [NIST FIPS 198-1 - HMAC Standard](https://csrc.nist.gov/publications/detail/fips/198/1/final) (NIST)
    - [Serious Cryptography - Chapter on MACs](https://nostarch.com/seriouscrypto) (No Starch Press)

## Digital Signatures

Digital signatures use asymmetric cryptography to provide authentication (proof of origin), integrity (detect tampering), and non-repudiation (signer cannot deny signing).

**How signing works:**

1. **Sign:** `signature = Sign(private_key, Hash(message))`
2. **Verify:** `valid = Verify(public_key, signature, Hash(message))`

**Common signature algorithms:**

| Algorithm | Based On | Key Size | Signature Size | Notes |
|-----------|----------|----------|---------------|-------|
| RSA-PSS | RSA | 2048-4096 bits | Key-size dependent | Probabilistic; preferred over PKCS#1 v1.5 |
| ECDSA | ECC (P-256, P-384) | 256-384 bits | 64-96 bytes | NIST standard; nonce reuse is catastrophic |
| Ed25519 | Curve25519 | 256 bits | 64 bytes | Deterministic; no nonce reuse risk |
| EdDSA | Edwards curves | Varies | Varies | General framework; Ed25519 is most common instance |

**Applications:**

- **TLS certificates** - CA signs server certificate; browser verifies
- **Code signing** - developer signs binaries; OS verifies before execution
- **Container image signing** - Cosign/Notary signs images; admission controller verifies
- **Git commit signing** - GPG or SSH key signs commits; GitHub shows verified badge
- **JWT** - RS256/ES256 sign tokens; API verifies with public key
- **Software updates** - vendor signs packages; package manager verifies

**ECDSA nonce reuse vulnerability:**

- ECDSA requires a unique random nonce per signature
- Reusing a nonce leaks the private key (PlayStation 3 hack, Bitcoin wallet theft)
- Ed25519 is deterministic (nonce derived from message + key), eliminating this class of bug

!!! info "External Resources"
    - [RFC 8032 - Edwards-Curve Digital Signature Algorithm](https://datatracker.ietf.org/doc/html/rfc8032) (IETF)
    - [NIST FIPS 186-5 - Digital Signature Standard](https://csrc.nist.gov/publications/detail/fips/186/5/final) (NIST)
    - [Code Signing Best Practices](https://cheatsheetseries.owasp.org/cheatsheets/Key_Management_Cheat_Sheet.html) (OWASP)

## Key Exchange & Perfect Forward Secrecy

Key exchange protocols allow two parties to derive a shared secret over an insecure channel without an eavesdropper learning the key.

**Diffie-Hellman (DH):**

1. Alice and Bob agree on public parameters (prime `p`, generator `g`)
2. Alice picks private `a`, sends `A = g^a mod p`
3. Bob picks private `b`, sends `B = g^b mod p`
4. Shared secret: `K = B^a mod p = A^b mod p`
5. Eavesdropper sees `A`, `B`, `p`, `g` but cannot compute `K` (discrete log problem)

**ECDH (Elliptic Curve Diffie-Hellman):**

- Same concept over elliptic curves; smaller keys, faster computation
- X25519 (Curve25519) is the most widely deployed ECDH variant
- Used in TLS 1.3, SSH, WireGuard, Signal Protocol

**Perfect Forward Secrecy (PFS):**

- Use ephemeral key pairs for each session (DHE or ECDHE)
- If long-term private key is later compromised, past sessions remain secure
- Without PFS: attacker records ciphertext, later obtains server private key, decrypts all recorded sessions
- TLS 1.3 mandates PFS (only ECDHE/DHE cipher suites allowed)

**Signal Protocol (Double Ratchet):**

- Combines DH ratchet with symmetric key ratchet
- Every message uses a new encryption key derived from the ratchet state
- Compromise of current key does not reveal past or future messages
- Used in Signal, WhatsApp, Matrix/Element

!!! info "External Resources"
    - [Diffie-Hellman Key Exchange - Cloudflare](https://www.cloudflare.com/learning/ssl/what-is-diffie-hellman/) (Cloudflare)
    - [RFC 7748 - Curve25519 and Curve448](https://datatracker.ietf.org/doc/html/rfc7748) (IETF)
    - [Signal Protocol Documentation](https://signal.org/docs/) (Signal)

## Entropy & Random Number Generation

Cryptographic security depends on unpredictable random numbers. Weak randomness has caused real-world key compromise.

**Entropy sources:**

| Source | Type | Examples |
|--------|------|---------|
| **Hardware RNG** | True random | CPU RDRAND/RDSEED, TPM, dedicated HSM |
| **OS entropy pool** | Mixed | `/dev/urandom` (Linux), `CryptGenRandom` (Windows), `getrandom()` syscall |
| **Environmental noise** | Physical | Disk timing, network jitter, mouse movement, interrupt timing |
| **PRNG** | Pseudo-random | CSPRNG seeded from entropy pool (ChaCha20-based in modern Linux) |

**PRNG (Pseudo-Random Number Generator):**

- Deterministic algorithm that expands a seed into a long stream of bits
- CSPRNG (Cryptographically Secure PRNG) - unpredictable output even if internal state is partially known
- Non-cryptographic PRNGs (Mersenne Twister, `Math.random()`) must never be used for security

**Entropy starvation:**

- VMs and containers at boot may lack entropy (no hardware noise, no user input)
- Blocked reads from `/dev/random` (Linux pre-5.6) caused application hangs
- Modern Linux (5.6+): `/dev/random` and `/dev/urandom` behave identically after initial seeding
- Solutions: `haveged`, `virtio-rng` for VM guests, RDRAND instruction

**Real-world entropy failures:**

| Incident | Cause | Impact |
|----------|-------|--------|
| Debian OpenSSL (2008) | PRNG seeded with only PID (15 bits of entropy) | All keys generated over 2 years predictable |
| Android SecureRandom (2013) | Insufficient seeding in Java PRNG | Bitcoin wallet key collisions |
| Dual_EC_DRBG | NSA-influenced NIST standard with suspected backdoor | Compromised implementations in RSA BSAFE, Juniper |

!!! info "External Resources"
    - [RFC 4086 - Randomness Requirements for Security](https://datatracker.ietf.org/doc/html/rfc4086) (IETF)
    - [NIST SP 800-90A - DRBG Mechanisms](https://csrc.nist.gov/publications/detail/sp/800-90a/rev-1/final) (NIST)
    - [Linux Random Number Generator](https://www.kernel.org/doc/html/latest/admin-guide/hw-random.html) (Linux Kernel)

## Common Cryptographic Attacks

| Attack | Target | How it works | Mitigation |
|--------|--------|-------------|-----------|
| **Brute force** | Key/password space | Try all possible keys | Use sufficient key length (AES-256), strong password hashing |
| **Birthday attack** | Hash collisions | Probability of collision is ~2^(n/2) for n-bit hash | Use 256-bit+ hashes |
| **Chosen-plaintext** | Encryption oracle | Attacker chooses plaintext, observes ciphertext | Use IND-CPA secure schemes (AEAD) |
| **Padding oracle** | CBC mode decryption | Error messages reveal padding validity | Use AEAD (GCM); constant-time comparison |
| **Length extension** | H(secret || message) | Append data to hash without knowing secret | Use HMAC instead of naive secret-prefix hash |
| **Nonce reuse** | GCM, ECDSA, ChaCha20 | Reusing nonce leaks key material | Use deterministic nonces (Ed25519) or robust PRNG |
| **Downgrade** | Protocol negotiation | Force older, weaker cipher/protocol | Enforce minimum versions (TLS 1.2+), TLS_FALLBACK_SCSV |
| **Side-channel** | Implementation timing, power, EM | Measure physical signals during crypto operations | Constant-time implementations, blinding, hardware isolation |
| **Replay** | Signed/encrypted messages | Resend valid message to repeat action | Nonces, timestamps, sequence numbers, token expiry |
| **Key compromise** | Key storage/distribution | Steal or guess the key | HSMs, key rotation, PFS, strict access control |

**General principles:**

- Never implement your own cryptography; use vetted libraries (libsodium, OpenSSL, BoringSSL)
- Keep cryptographic dependencies updated
- Use AEAD for all encryption (provides confidentiality + integrity)
- Enforce minimum protocol versions and cipher suites
- Monitor for advances in cryptanalysis (SHA-1 collision, quantum computing threats to RSA/ECC)

!!! info "External Resources"
    - [Attack Model - Wikipedia](https://en.wikipedia.org/wiki/Attack_model) (Wikipedia)
    - [Cryptographic Right Answers](https://latacora.micro.blog/2018/04/03/cryptographic-right-answers.html) (Latacora)
    - [NIST Post-Quantum Cryptography](https://csrc.nist.gov/Projects/post-quantum-cryptography) (NIST)
