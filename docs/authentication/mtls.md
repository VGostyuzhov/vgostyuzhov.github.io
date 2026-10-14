# Mutual TLS (mTLS)

Mutual TLS uses certificate-based authentication for both peers in a TLS connection. It is commonly used for service-to-service communication, device identity, and partner APIs. A trusted certificate proves an authenticated identity only after validation and proof of private-key possession; it does not grant every operation.

## How it Works

This flow summarizes certificate authentication in TLS 1.3. TLS 1.2 ordering, resumption, and post-handshake authentication differ.

1. The client starts negotiation. The server sends its negotiated parameters and requests a client certificate when mTLS is required.
2. The server sends its certificate chain and a CertificateVerify signature proving possession of the corresponding private key, followed by its Finished message.
3. The client validates the server chain, expected service identity, validity, and applicable key usage/policy, and verifies the handshake proof.
4. The client supplies its certificate and CertificateVerify proof, followed by Finished. The server validates these against its client-identity trust policy.
5. The server maps the verified client identity to an authorization policy for the requested service and operation.

```mermaid
sequenceDiagram
    accTitle: TLS 1.3 mutual certificate authentication
    accDescr: The client validates the server certificate and handshake proof, the server validates the client certificate and proof. Application authorization follows successful TLS authentication. The diagram omits key derivation and record-layer details.
    participant C as Client
    participant S as Server
    C->>S: ClientHello
    S-->>C: ServerHello, EncryptedExtensions,<br/>CertificateRequest
    S-->>C: Certificate, CertificateVerify, Finished
    C->>C: Validate server identity,<br/>certificate chain and handshake proof
    C->>S: Certificate, CertificateVerify, Finished
    S->>S: Validate client identity,<br/>certificate chain and handshake proof
    C->>S: Protected application request
    S->>S: Authorize client for operation
```

A certificate file by itself is not sufficient to impersonate its owner: the peer must possess the private key. TLS encryption protects the connection, while application authorization still decides resource access.

!!! info "External Resources"
    - [TLS 1.3 Certificate Authentication](https://www.rfc-editor.org/rfc/rfc8446.html#section-4.4) (IETF)

## Identity and Authorization Boundaries

Trust only the issuing authorities and identities needed for the deployment. Check certificate validity, appropriate Extended Key Usage and policy, and the intended peer identity. The server identity may be a DNS name; a workload identity system may use a URI SAN and its own trust-domain rules.

**Example:** Both `orders` and `reporting` have valid certificates from the same organizational CA. An inventory API permits `orders` to update stock and `reporting` to read aggregates. Accepting the `reporting` certificate must not permit stock updates. Trusting a CA is not equivalent to authorizing every identity that CA issues.

If a proxy terminates TLS, downstream services authenticate that proxy connection, not automatically the original client. Strip untrusted identity headers and propagate verified client identity only through an authenticated, explicitly trusted mechanism. Document which component makes the authorization decision.

!!! info "External Resources"
    - [X.509 Certificate and Path Validation](https://www.rfc-editor.org/rfc/rfc5280.html) (IETF)
    - [SPIFFE X.509-SVID Identity and Validation](https://github.com/spiffe/spiffe/blob/main/standards/X509-SVID.md) (SPIFFE)

## Certificate Lifecycle and Failure Handling

| Risk | Primary control | Additional considerations |
|---|---|---|
| Private-key theft | Restrict key access and use protected key storage where supported | Short certificate lifetimes reduce exposure but do not end it immediately |
| Expired credentials | Automate issuance/renewal and monitor expiry | Test rotation overlap and failure recovery before outages |
| Overbroad trust | Restrict trust domains and authorized peer identities | Separate issuing CAs by purpose when appropriate |
| Compromised issuer | Protect CA keys and maintain an incident response process | Plan trust-bundle replacement and workload reissuance |
| Revocation uncertainty | Define how validation responds to revocation information | CRL/OCSP use and availability behavior vary by stack; do not assume every verifier checks them |
| Weak transport settings | Prefer TLS 1.3; support securely configured TLS 1.2 where required | Compatibility policy must be explicit rather than an unqualified version mandate |

mTLS reduces dependence on reusable shared API secrets, but issuance, rotation, authorization mapping, and trust distribution remain operational responsibilities. It is one component of a broader access-control architecture.

!!! info "External Resources"
    - [Transport Layer Security Guidance](https://cheatsheetseries.owasp.org/cheatsheets/Transport_Layer_Security_Cheat_Sheet.html) (OWASP)
    - [Certificate Revocation and Validation](https://www.rfc-editor.org/rfc/rfc5280.html#section-6) (IETF)
