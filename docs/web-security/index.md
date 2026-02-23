# Web Application Security

This section covers web application security fundamentals, common vulnerability classes, API security patterns, and the tools and technologies used to build and defend modern web applications.

## Core Web Security

- **[Web Security Fundamentals](fundamentals.md)** - Browser security model, Same-Origin Policy, CORS, Content Security Policy, secure cookies and storage
- **[Web Vulnerabilities](vulnerabilities.md)** - Common vulnerability classes (XSS, SQLi, SSRF, CSRF, IDOR), XSS variants, injection flaws, deserialization attacks
- **[Web Technologies & Tools](tools.md)** - API and microservice security, REST vs GraphQL concerns, authentication and authorization patterns, rate limiting and throttling

## Key Learning Objectives

After studying this section, you should understand:

- How browser security mechanisms (SOP, CORS, CSP) protect users
- Cookie security attributes and their role in session management
- The major web vulnerability classes and their mitigations
- XSS variant differences and detection approaches
- Injection flaw root causes and parameterisation defences
- REST and GraphQL security trade-offs
- Rate limiting algorithms and implementation patterns
- Authentication and authorization best practices for web APIs

!!! info "Prerequisites"
    Basic understanding of HTTP, HTML, and client-server architecture is assumed. For fundamentals, refer to [MDN Web Docs](https://developer.mozilla.org/en-US/docs/Web) and the [OWASP Web Security Testing Guide](https://owasp.org/www-project-web-security-testing-guide/).
