# SAML 2.0 (Security Assertion Markup Language)

SAML 2.0 is an XML-based open standard for exchanging authentication and authorization data between an **Identity Provider (IdP)** and a **Service Provider (SP)**.

## How it Works

1.  **Access Attempt**: User tries to access an SP (e.g., Salesforce).
2.  **Redirect**: SP sends a SAML Request to the IdP (e.g., Okta).
3.  **AuthN**: User logs into the IdP.
4.  **Assertion**: IdP generates a signed XML document (Assertion) and sends it back to the SP via the browser.
5.  **Validation**: SP verifies the XML signature using the IdP's public key and grants access.

## Pros and Cons

| Pros | Cons |
|------|------|
| **Enterprise Standard**: Deeply integrated into legacy corporate environments (Active Directory/ADFS). | **Verbose**: XML payloads are large and complex. |
| **No Passwords on SP**: The Service Provider never sees the user's password. | **Complex Security**: XML security is notoriously difficult to implement correctly. |
| **User Experience**: Seamless Single Sign-On (SSO) experience for employees. | **Browser-Bound**: Primarily designed for web browsers; difficult for mobile/APIs. |

## Threats and Mitigations

| Threat | Mitigation |
|--------|------------|
| **XML Signature Wrapping (XSW)** | Ensure the signature covers the entire assertion and validate the position of elements. |
| **XML External Entity (XXE)** | Disable external entity resolution in the XML parser. |
| **Token Replay** | Use a unique `ID` for assertions and track them in a "used tokens" cache; validate timestamps (`NotBefore`, `NotOnOrAfter`). |
| **Insecure Redirects** | Use the **HTTP POST Binding** instead of HTTP Redirect where possible. |
| **Clock Skew** | Ensure IdP and SP clocks are synchronized via NTP. |

## Key Concepts

-   **Metadata Exchange**: The IdP and SP exchange XML metadata (containing public keys and endpoints) to establish trust.
-   **NameID**: A unique identifier for the user (e.g., email or employee ID).
-   **Bindings**: Methods of transport (HTTP POST, HTTP Redirect, Artifact).
