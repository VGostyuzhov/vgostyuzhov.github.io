# Mutual TLS (mTLS)

Mutual TLS (mTLS) is a method for mutual authentication in which both the client and the server verify each other's identity through X.509 digital certificates.

## How it Works

In standard TLS, only the server provides a certificate to the client. In mTLS:

1.  **Client Hello**: Client initiates TLS handshake.
2.  **Server Hello + Request**: Server provides its certificate and requests the client's certificate.
3.  **Client Certificate**: Client provides its certificate signed by a trusted Certificate Authority (CA).
4.  **Verification**: Both parties verify each other's certificates against their trusted CA stores.
5.  **Secure Channel**: If both are valid, an encrypted and authenticated tunnel is established.

## Pros and Cons

| Pros | Cons |
|------|------|
| **Strong Identity**: Cryptographically proven identity for both sides. | **Complexity**: Managing, issuing, and rotating client certificates is difficult. |
| **Zero Trust**: Foundation of Zero Trust networking (never trust, always verify). | **Inflexible**: Hard to use with browsers/humans; best for machine-to-machine. |
| **Tamper-Proof**: Certificates are hard to forge without the private key. | **Revocation**: CRLs and OCSP add latency and complexity to the stack. |

## Threats and Mitigations

| Threat | Mitigation |
|--------|------------|
| **Private Key Theft** | Use Hardware Security Modules (HSM) or TPMs to store private keys; use short-lived certificates. |
| **Stale Certificates** | Implement automated certificate rotation (e.g., using SPIFFE/Spire or cert-manager). |
| **Weak CA Security** | Protect the Root CA and use Intermediate CAs for issuance. |
| **Cipher Suite Weakness** | Enforce TLS 1.3 and disable weak/legacy ciphers. |

## Use Cases

-   **Microservices**: Securing traffic between services in a service mesh (e.g., Istio, Linkerd).
-   **API Security**: Restricting high-value internal APIs to specific authorized clients.
-   **IoT**: Authenticating large fleets of devices to a central cloud.
-   **B2B Integrations**: Securing data exchange between two corporate networks.
