# WebAuthn & FIDO2

WebAuthn (Web Authentication) is a browser-based API that allows web applications to use built-in authenticators (like Windows Hello, FaceID, or YubiKeys) for secure, public-key-based authentication.

## How it Works

1.  **Registration**:
    -   User requests to register an authenticator.
    -   The authenticator generates a new **public/private key pair** unique to that origin (domain).
    -   The **public key** is sent to the server; the **private key** never leaves the device's secure hardware.
2.  **Authentication**:
    -   Server sends a challenge.
    -   User verifies identity via biometric or PIN.
    -   The authenticator signs the challenge with the private key.
    -   Server verifies the signature with the stored public key.

## Pros and Cons

| Pros | Cons |
|------|------|
| **Phishing Resistant**: Keys are scoped to a specific domain (origin-bound). | **Recovery**: If a user loses their hardware key and has no backup, account recovery is difficult. |
| **No Passwords**: Eliminates the risk of credential stuffing and password database leaks. | **Implementation**: Complex to build compared to simple password forms. |
| **High Security**: Uses secure hardware (TPM/Secure Enclave). | **Device Support**: Requires modern browsers and hardware. |

## Threats and Mitigations

| Threat | Mitigation |
|--------|------------|
| **Device Theft** | Hardware authenticators require a PIN or Biometric to unlock the private key. |
| **Origin Mismatch** | Browsers strictly enforce the `rpId` (Relying Party ID), preventing keys from being used on phishing sites. |
| **Replay Attack** | Every request includes a unique server-side **challenge**. |
| **Loss of Device** | Encourage users to register multiple authenticators (e.g., a YubiKey AND a Phone). |

## Key Concepts

-   **Authenticator**: The hardware (e.g., TouchID, USB Key).
-   **Relying Party (RP)**: The website using the API.
-   **Attestation**: Evidence that the authenticator is genuine and of a specific type.
-   **Assertion**: The cryptographically signed proof provided during login.
