# OAuth 2.0 & OpenID Connect (OIDC)

OAuth 2.0 is an **authorization** framework that allows applications to obtain limited access to user accounts. OpenID Connect (OIDC) is an **identity** layer on top of OAuth 2.0.

## Key Differences

-   **OAuth 2.0**: "Can this app access my photos?" (Returns an **Access Token**).
-   **OIDC**: "Who is the user logged in?" (Returns an **ID Token**).

## Core Concepts

-   **Resource Owner**: The User.
-   **Client**: The application (Web, Mobile, etc.).
-   **Authorization Server**: The Identity Provider (IdP) (e.g., Okta, Google, Auth0).
-   **Resource Server**: The API providing the data.

## Common OAuth 2.0 Flows

### 1. Authorization Code Flow (with PKCE)
**Used for**: SPAs, Mobile Apps, and Server-side apps.
-   **PKCE (Proof Key for Code Exchange)** is now mandatory for almost all clients to prevent authorization code injection attacks.

### 2. Client Credentials Flow
**Used for**: Machine-to-machine (M2M) communication where no user is involved.

## OpenID Connect (OIDC) Features

OIDC adds specific features to OAuth 2.0:
-   **ID Token**: A JWT containing user profile information.
-   **Standard Scopes**: `openid`, `profile`, `email`.
-   **UserInfo Endpoint**: A protected resource to fetch more user details.

## Pros and Cons

| Pros | Cons |
|------|------|
| **Delegated Access**: Users don't share passwords with third-party apps. | **Complexity**: Significant configuration required (Redirect URIs, Scopes, Client Secrets). |
| **Centralized Identity**: Single Sign-On (SSO) across multiple apps. | **Central Point of Failure**: If the IdP is down, all apps are inaccessible. |
| **Standardized**: Interoperable across different vendors. | **Token Leakage Risk**: Tokens are passed via browser redirects or headers. |

## Threats and Mitigations

| Threat | Mitigation |
|--------|------------|
| **Auth Code Interception** | Use **PKCE** (`code_verifier` and `code_challenge`). |
| **Open Redirects** | Use strict allow-lists for `redirect_uri`. |
| **CSRF in Auth Flow** | Use the `state` parameter to link the request and response. |
| **Token Theft** | Use short-lived tokens and Rotate Refresh Tokens. |
| **Implicit Flow Risks** | **Avoid the Implicit Flow**; it is deprecated in favor of Auth Code + PKCE. |
