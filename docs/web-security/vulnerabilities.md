# Web Vulnerabilities

A vulnerability connects attacker-controlled input or identity to an operation that violates a trust boundary. For each class, identify the affected interpreter or authorization decision, fix that boundary, and use additional controls to limit impact.

## Common Web Vulnerabilities

**Cross-Site Scripting (XSS)**

- *What happens* Attacker injects JavaScript into a page viewed by other users.
- *Impact* Session theft, DOM modification, phishing overlays, drive-by malware.
- *Typical root causes* Unsanitised user input echoed into HTML, attributes, or JS.
- *Mitigations* Use safe DOM/template sinks and context-appropriate output encoding; sanitize intentionally allowed HTML with a maintained library. CSP and HttpOnly cookies reduce particular impacts but do not fix the injection.

**SQL Injection (SQLi)**

- *What happens* User-supplied data alters back-end SQL queries.
- *Impact* Data exfiltration, authentication bypass, stored-procedure execution.
- *Typical root causes* String-concatenated queries, insufficient parameterisation.
- *Mitigations* Use prepared statements / ORM bind variables; allow-list identifiers that cannot be bound. Least-privilege DB access and WAF rules are additional layers.

**Server-Side Request Forgery (SSRF)**

- *What happens* Server forced to fetch internal resources on attacker's behalf.
- *Impact* Cloud metadata theft, port scanning, internal service exploitation.
- *Typical root causes* Unvalidated URLs in image fetchers, webhooks, PDF converters.
- *Mitigations* Constrain schemes and destinations, validate resolved addresses and redirects, and enforce egress policy. Blocking a single metadata IP or signing a user-chosen URL is not sufficient.

**Insecure Direct Object Reference (IDOR)**

- *What happens* User manipulates IDs in URLs or JSON to access others' data.
- *Impact* Horizontal/vertical privilege escalation, data leakage.
- *Typical root causes* Authorisation checks tied only to user-supplied identifiers.
- *Mitigations* Enforce object ownership, tenant, and action authorization on every access. Unpredictable IDs can reduce enumeration but do not replace access checks.

**Cross-Site Request Forgery (CSRF)**

- *What happens* Logged-in victim's browser sends unwanted state-changing request.
- *Impact* Account takeover actions (change email, transfer funds).
- *Typical root causes* Session cookies auto-sent, no secondary validation.
- *Mitigations* Use framework CSRF protection, such as synchronizer tokens or signed session-bound double-submit tokens, and appropriate Origin/Referer checks. SameSite adds protection; REST alone does not. Risk depends on credential delivery.

**Deserialization of Untrusted Data**

- *What happens* Crafted payload triggers gadget chain when server deserialises.
- *Impact* Remote code execution, privilege escalation.
- *Typical root causes* Java/PHP/Python object deserialisation without allow-list.
- *Mitigations* Avoid deserializing untrusted native objects; use data-only parsing with schema/type validation. If native formats are unavoidable, apply strict type restrictions and isolate processing; dependency patches alone are not sufficient.

**Command / Path Injection**

- *What happens* User input inserted into shell command or file path.
- *Impact* Arbitrary command execution, file overwrite or read.
- *Typical root causes* `os.system`, back-ticks, unsanitised file names in `exec` or `open`.
- *Mitigations* Avoid a shell; use a fixed executable and argument array with validated values. Prevent option injection as well as shell metacharacters. Run with least privilege.

**Directory Traversal**

- *What happens* `../../../etc/passwd` accesses files outside web root.
- *Impact* Sensitive file disclosure, configuration theft.
- *Typical root causes* Concatenating user path fragments, no canonicalisation.
- *Mitigations* Prefer mapped file IDs. Otherwise decode/normalize consistently, resolve the path, and enforce path-component containment in a trusted base. Account for symlinks and filesystem races; stripping `..` is not sufficient.

**Broken Access Control**

- *What happens* Endpoints rely on client-side checks or omit authorisation layer.
- *Impact* Privilege escalation, data leaks, admin-only functions exposed.
- *Typical root causes* RBAC not enforced on back-end, hidden UI elements only.
- *Mitigations* Centralised authZ middleware, deny-by-default, penetration tests.

**Insecure File Upload**

- *What happens* Attacker uploads executable script or oversized file.
- *Impact* Remote code execution, DoS storage exhaustion.
- *Typical root causes* File type verified by extension only, no size limits.
- *Mitigations* Allow-list supported formats, check content rather than trusting the supplied MIME header, use generated names, store outside executable paths, and restrict size/count. Scanning is an additional layer.

**Prototype Pollution**

- *What happens* Attacker-controlled property paths modify an object's prototype or shared inherited properties.
- *Impact* Alter application behavior, bypass checks, or reach code-execution paths in affected applications.
- *Typical root causes* Unsafe recursive merges or dynamic property writes accepting keys such as `__proto__` or `constructor.prototype`.
- *Mitigations* Use safe maps/null-prototype objects where appropriate, reject dangerous property paths, validate schemas, and maintain merge libraries. No single object-creation choice fixes every unsafe merge.

**Mass Assignment**

- *What happens* Input automatically populates model properties the caller is not allowed to change, such as `isAdmin` or `tenantId`.
- *Impact* Privilege escalation and unauthorized data modification without any prototype mutation.
- *Typical root causes* Binding an entire request object to a persistence model.
- *Mitigations* Explicitly allow-list writable fields and enforce field-level authorization; derive privileged properties on the server.

!!! info "External Resources"
    - [Prototype Pollution Prevention](https://cheatsheetseries.owasp.org/cheatsheets/Prototype_Pollution_Prevention_Cheat_Sheet.html) (OWASP)
    - [Mass Assignment Prevention](https://cheatsheetseries.owasp.org/cheatsheets/Mass_Assignment_Cheat_Sheet.html) (OWASP)

**Weak TLS Configuration**

- *What happens* Old ciphers, TLS 1.0, missing HSTS.
- *Impact* Downgrade attacks, session hijack.
- *Typical root causes* Legacy servers, mis-configured load balancers.
- *Mitigations* Prefer TLS 1.3 and securely configured TLS 1.2 where required; validate certificates and deploy HSTS deliberately. Preloading and application pinning need separate compatibility and recovery planning.

!!! info "External Resources"
    - [OWASP Top Ten](https://owasp.org/www-project-top-ten/) (OWASP)
    - [OWASP Web Security Testing Guide](https://owasp.org/www-project-web-security-testing-guide/) (OWASP)
    - [CWE Top 25 — 2023 Edition (Historical)](https://cwe.mitre.org/top25/archive/2023/2023_top25_list.html) (MITRE)

## XSS Variants

Stored/reflected describe delivery and persistence; DOM-based describes client-side execution. These categories can overlap. Blind XSS describes limited feedback, rather than a separate interpreter boundary.

| Category | Mechanism | Teaching example or distinction |
|---|---|---|
| **Reflected XSS** | Untrusted input is reflected into an executable context | A search result inserts query text into HTML without appropriate encoding |
| **Stored XSS** | Persisted input later reaches an executable context | A saved comment or browser-stored value is rendered unsafely |
| **DOM-Based XSS** | Client code passes attacker-controlled data to an unsafe sink | `location.hash` reaches `innerHTML`; other sources may also pass through a server |
| **Blind XSS** | Execution occurs where the tester lacks direct feedback | An input later appears in an administrative interface; test with controlled lab instrumentation |
| **Self-XSS** | A user is induced to execute code in their own browser context | Console-paste social engineering; not proof of a server-side injection flaw |
| **Mutation XSS** | Parsing/reparsing changes sanitized markup into executable content | Sanitizer/browser parsing differences require maintained sanitizers and regression cases |
| **Universal XSS (UXSS)** | A browser or extension flaw breaks origin isolation | Distinct from an application's unsafe HTML insertion; browser/extension fixes are required |

**Key constants**

* Application XSS commonly arises when untrusted data reaches executable HTML, attribute, URL, or JavaScript contexts. UXSS instead involves a browser or related component violating isolation.
* Use context-appropriate encoding for plain data, safe DOM APIs for text, and maintained HTML sanitization for intentional markup. Avoid dangerous contexts; encoding is not universally sufficient. CSP is defense in depth.
* `HttpOnly` prevents direct cookie reads through JavaScript; injected script can still perform authenticated actions, read page data, or capture input.

**Detection and a Safe-Sink Example**

* Reflected - trace input into the response context and test safely; searching logs for `<script>` misses many encodings and non-script-element payloads.
* Stored / Blind - trace later rendering paths in an authorized test environment and use controlled observation; never send production secrets to callback services.
* DOM - run static analysis (`eslint-plugin-xss`) and dynamic scanners (`DOM Invader`).

For plain text, use a text sink instead of interpreting input as markup. Assume the document contains an element with ID `message`:

```javascript
// Display untrusted text without parsing it as HTML.
const message = new URLSearchParams(location.search).get("message") ?? "";
document.querySelector("#message").textContent = message;
```

With an input such as `<b>demo</b>`, the expected result is visible literal text, not a bold element. If rich HTML is required, this example does not implement it; use a maintained sanitizer and a deliberately constrained policy.

!!! info "External Resources"
    - [Cross-Site Scripting Prevention Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Cross_Site_Scripting_Prevention_Cheat_Sheet.html) (OWASP)
    - [DOM-Based XSS Prevention Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/DOM_based_XSS_Prevention_Cheat_Sheet.html) (OWASP)
    - [Browser Isolation and UXSS](https://www.chromium.org/Home/chromium-security/site-isolation/) (Chromium)

## Injection Flaws

Injection occurs when data changes the syntax or behavior of an interpreter. The table distinguishes the primary boundary control from measures that limit exposure or impact.

| Injection type        | Where attacker sends payload                | Typical goal & impact                    | Illustrative input                    | Primary defences                                                   |
|-----------------------|---------------------------------------------|------------------------------------------|--------------------------------------------|--------------------------------------------------------------------|
| **SQL Injection (SQLi)**     | Query parameters, form fields, HTTP headers | Dump/alter database, bypass auth, RCE via xp_cmdshell. | `' OR 1=1--`                                   | Bind data values; allow-list dynamic identifiers. Least-privilege DB access and WAF rules are additional layers. |
| **NoSQL / Mongo Injection**  | JSON body or query string consumed by NoSQL driver | Read/modify NoSQL docs, escalate role.   | `{ "$ne": null }`                              | Require expected scalar types and construct fixed query operators server-side; reject user-controlled operator objects. |
| **Command / Shell Injection**| Any value passed to OS shell (`exec`, `system`) | Execute arbitrary commands, elevate privileges. | `demo; printf injected`                                   | Avoid shell interpretation; use fixed executable/argument arrays and validate options and values. Least privilege limits impact. |
| **LDAP Injection**           | Filters in directory search/bind      | Read/modify directory entries, bypass login. | `*)(uid=*)`                                    | Use parameterised LDAP APIs, escape RFC-4515 special chars, least-priv LDAP creds. |
| **Path / File Inclusion**    | File paths, `include()` arguments     | Read sensitive files, RCE via uploaded script. | `../../../../etc/passwd`                       | Prefer mapped IDs; enforce canonical path-component containment and address symlink/race behavior. Never load executable templates from untrusted paths. |
| **XML / XXE**                | XML documents parsed by server        | Read files, SSRF via external entities.  | `<!ENTITY x SYSTEM "file:///etc/passwd">`      | Disable external entities (`XXE`), prefer JSON, use parsing libraries with `secureProcessing`. |
| **Expression / Template Injection (SSTI)** | Untrusted input incorporated into template source or evaluated as template code | Execute code on the server, leak secrets. | `{{7*7}}`                                       | Keep template source fixed and pass untrusted data only as values. HTML escaping protects output, not already parsed template code; sandboxing is an additional control. |
| **Log Injection**            | Values logged without sanitisation    | Forge log entries, inject newlines - log poisoning. | `User=admin\n[ERROR] ...`                       | Replace control chars, structured logging (JSON), encode before log write. |

---

*Root causes*

- Concatenating untrusted data into **code**, **queries**, **paths**, **commands**, or **log lines**.
- Missing contextual escaping / validation.
- Over-privileged service accounts amplifying the blast radius.

*Mitigation priorities*

1. **Parameterise, don't concatenate** - use prepared statements or driver APIs.
2. **Constrain identities** - DB, LDAP, or OS users should have minimum privileges.
3. **Input allow-lists** - reject unexpected types, formats, ranges early.
4. **Use context-specific controls** - safe query APIs, validated argument arrays, correct LDAP filter/DN escaping, and safe template/DOM sinks. HTML output encoding is not SQL parameterization or an SSTI defense.
5. **Security-focused code reviews + fuzzers** - automate detection of risky string ops.
6. **Runtime protections** - WAF/IDS signatures for common injection payloads.

**Parameterization example:** Python's SQLite API binds a string value separately from SQL syntax:

```python
# Assume an existing sqlite3 connection and an authorized account lookup.
row = connection.execute(
    "SELECT display_name FROM users WHERE user_id = ?",
    (supplied_user_id,),
).fetchone()
```

An input containing SQL punctuation remains a value and does not change the query structure. This does not authorize access to another user's record; ownership and tenant checks are separate. Placeholder syntax varies by database driver.

For a JSON-based NoSQL lookup, require the expected string field before building the query; reject an object such as `{ "$ne": null }` where a username string is expected. Driver APIs do not automatically make arbitrary attacker-provided query objects safe.

!!! info "External Resources"
    - [SQL Injection Prevention Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/SQL_Injection_Prevention_Cheat_Sheet.html) (OWASP)
    - [OS Command Injection Defense Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/OS_Command_Injection_Defense_Cheat_Sheet.html) (OWASP)
    - [NoSQL Security](https://cheatsheetseries.owasp.org/cheatsheets/NoSQL_Security_Cheat_Sheet.html) (OWASP)
    - [Server-Side Template Injection Prevention](https://cheatsheetseries.owasp.org/cheatsheets/Server_Side_Template_Injection_Prevention_Cheat_Sheet.html) (OWASP)
    - [SQLite Parameter Binding](https://docs.python.org/3/library/sqlite3.html#sqlite3-placeholders) (Python)

## SSRF & Deserialization

SSRF crosses a network-destination boundary; native-object deserialization crosses a data-to-code boundary. Both require controls at the point where untrusted input influences the privileged operation.

Key additional notes on SSRF:

- **Cloud metadata:** AWS IMDSv2 obtains a token through a PUT containing `X-aws-ec2-metadata-token-ttl-seconds`, then requires `X-aws-ec2-metadata-token` on metadata requests. Token lifetime is distinct from the token response's configured IP hop limit. Require IMDSv2 where appropriate; it limits some SSRF paths but is not a universal defense against an attacker able to issue arbitrary methods/headers and read responses.
- **DNS and address validation:** Validate all relevant resolved IPv4/IPv6 addresses and bind the connection to an approved result while preserving intended TLS server identity. Do not validate one resolution and let another uncontrolled resolution choose the destination. Constrain schemes, ports, and internal/link-local destinations according to policy.
- **Redirect chains:** Disable redirects for user-selected destinations unless needed; if allowed, revalidate every destination and address. Apply network egress restrictions independently of URL checks.

Key additional notes on Deserialization:

- **Language-specific risks**: Java (`ObjectInputStream`), Python (`pickle`), PHP (`unserialize`), Ruby (`Marshal.load`) all have known gadget chains.
- **Detection:** Serialized-object markers can support triage, but encodings and format variants bypass such signatures. Prevent unsafe parsing rather than relying on marker detection.
- **Alternative formats:** Prefer data-only formats with schema, type, size, and nesting limits. Avoid configurations or extensions that instantiate arbitrary classes; changing the format name alone does not guarantee safety.

**Failure example:** A URL initially resolves to an allowed public host but redirects to an internal admin endpoint. Validating only the original hostname does not enforce the destination policy. Reject the redirected destination before connecting; keep the egress rule as an independent control.

!!! info "External Resources"
    - [Server-Side Request Forgery Prevention Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Server_Side_Request_Forgery_Prevention_Cheat_Sheet.html) (OWASP)
    - [Deserialization Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Deserialization_Cheat_Sheet.html) (OWASP)
    - [AWS IMDSv2 Token and Hop-Limit Behavior](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/configuring-instance-metadata-service.html) (AWS)
