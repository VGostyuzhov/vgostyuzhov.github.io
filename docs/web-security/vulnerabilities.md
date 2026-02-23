# Web Vulnerabilities

## Common Web Vulnerabilities

**Cross-Site Scripting (XSS)**

- *What happens* Attacker injects JavaScript into a page viewed by other users.
- *Impact* Session theft, DOM modification, phishing overlays, drive-by malware.
- *Typical root causes* Unsanitised user input echoed into HTML, attributes, or JS.
- *Mitigations* Context-aware output encoding (HTML vs JS vs URL), CSP `script-src` with nonces/hashes, HttpOnly cookies, strict input validation.

**SQL Injection (SQLi)**

- *What happens* User-supplied data alters back-end SQL queries.
- *Impact* Data exfiltration, authentication bypass, stored-procedure execution.
- *Typical root causes* String-concatenated queries, insufficient parameterisation.
- *Mitigations* Prepared statements / ORM placeholders, least-privilege DB user, input whitelists, WAF filtering.

**Server-Side Request Forgery (SSRF)**

- *What happens* Server forced to fetch internal resources on attacker's behalf.
- *Impact* Cloud metadata theft, port scanning, internal service exploitation.
- *Typical root causes* Unvalidated URLs in image fetchers, webhooks, PDF converters.
- *Mitigations* Allow-list outbound destinations, deny localhost/169.254.169.254, network egress filters, require signed URLs.

**Insecure Direct Object Reference (IDOR)**

- *What happens* User manipulates IDs in URLs or JSON to access others' data.
- *Impact* Horizontal/vertical privilege escalation, data leakage.
- *Typical root causes* Authorisation checks tied only to user-supplied identifiers.
- *Mitigations* Enforce access control on every request, use unpredictable IDs (UUID), avoid exposing internal keys.

**Cross-Site Request Forgery (CSRF)**

- *What happens* Logged-in victim's browser sends unwanted state-changing request.
- *Impact* Account takeover actions (change email, transfer funds).
- *Typical root causes* Session cookies auto-sent, no secondary validation.
- *Mitigations* `SameSite=Lax/Strict`, double-submit CSRF tokens, verify `Origin/Referer`, use REST patterns with access tokens instead of cookies.

**Deserialization of Untrusted Data**

- *What happens* Crafted payload triggers gadget chain when server deserialises.
- *Impact* Remote code execution, privilege escalation.
- *Typical root causes* Java/PHP/Python object deserialisation without allow-list.
- *Mitigations* Use safe formats (JSON), enforce class allow-list, upgrade libraries, isolate deserialisation in low-privilege context.

**Command / Path Injection**

- *What happens* User input inserted into shell command or file path.
- *Impact* Arbitrary command execution, file overwrite or read.
- *Typical root causes* `os.system`, back-ticks, unsanitised file names in `exec` or `open`.
- *Mitigations* Use parameter APIs (`execFile`), allow-list file names, escape/quote args, run under least-privilege account.

**Directory Traversal**

- *What happens* `../../../etc/passwd` accesses files outside web root.
- *Impact* Sensitive file disclosure, configuration theft.
- *Typical root causes* Concatenating user path fragments, no canonicalisation.
- *Mitigations* Resolve to absolute path then validate against base dir, strip `..`, serve files via mapped IDs.

**Broken Access Control**

- *What happens* Endpoints rely on client-side checks or omit authorisation layer.
- *Impact* Privilege escalation, data leaks, admin-only functions exposed.
- *Typical root causes* RBAC not enforced on back-end, hidden UI elements only.
- *Mitigations* Centralised authZ middleware, deny-by-default, penetration tests.

**Insecure File Upload**

- *What happens* Attacker uploads executable script or oversized file.
- *Impact* Remote code execution, DoS storage exhaustion.
- *Typical root causes* File type verified by extension only, no size limits.
- *Mitigations* Content-type/extension validation, virus scanning, rename + store outside web root, limit size and count.

**Prototype Pollution / Mass Assignment**

- *What happens* Attacker overwrites JavaScript object prototype keys or model attributes.
- *Impact* Add arbitrary properties, escalate privileges.
- *Typical root causes* Merging user JSON into objects (`Object.assign(req.body, obj)`).
- *Mitigations* `Object.create(null)`, deep-clone allow-list, framework-level param filtering.

**Weak TLS Configuration**

- *What happens* Old ciphers, TLS 1.0, missing HSTS.
- *Impact* Downgrade attacks, session hijack.
- *Typical root causes* Legacy servers, mis-configured load balancers.
- *Mitigations* TLS 1.3 (fallback TLS 1.2), modern cipher suites, HSTS preload, certificate pinning for mobile apps.

!!! info "External Resources"
    - [OWASP Top Ten](https://owasp.org/www-project-top-ten/) (OWASP)
    - [OWASP Web Security Testing Guide](https://owasp.org/www-project-web-security-testing-guide/) (OWASP)
    - [CWE Top 25 Most Dangerous Software Weaknesses](https://cwe.mitre.org/top25/archive/2023/2023_top25_list.html) (MITRE)

## XSS Variants

| Variant             | Where payload is stored or reflected | How the script gets executed | Typical entry point examples | Distinctive challenges |
|---------------------|--------------------------------------|------------------------------|------------------------------|------------------------|
| **Reflected XSS**   | Lives only in the *request* (URL, form field, header); echoed in the *same* response. | Victim clicks attacker-crafted link or submits form; browser immediately runs script. | `https://site.com/search?q=<script>` | Payload must convince user to visit link; easy to spot in access logs. |
| **Stored (Persistent) XSS** | Written into server-side datastore (DB, comment, profile) and served to *all* future visitors. | Victim simply views the page that reads the stored data. | Forum post, support ticket, user bio field. | High impact, long-lived; "sleeping" payloads evade scanning. |
| **DOM-Based XSS**   | Injection never touches server; JavaScript on page reads attacker-controlled value (e.g., `location.hash`) and writes it to the DOM with `innerHTML`. | Script runs purely in client's browser as page logic executes. | `https://site.com/#<img src=x onerror=alert(1)>` | Server logs look clean; CSP with strict `script-src` helps most. |
| **Blind XSS**       | Like stored XSS but payload fires in a *different* application or admin interface the attacker never sees. | Attacker gets no direct feedback - must beacon outbound (e.g., webhook). | Contact-us form reflected in help-desk dashboard. | Hard to test manually; use out-of-band detection (`<script src=//burp.collab/>`). |
| **Self-XSS**        | Victim pastes payload into their own browser console thinking it's "hack tool" or "coupon generator". | Runs in victim's context; social-engineering angle. | "Paste this in console for free skins!" | Browsers now warn users; educate rather than patch server. |
| **Mutation / Universal XSS (UXSS)** | Browser re-parses a benign string into executable code (e.g., SVG, CSS escapes, innerHTML auto-close). | Exploits parser quirks; may bypass usual sanitizers. | `<math><mtext></math><img src onerror=alert>` | Requires context-aware sanitisation libs; keep dependencies patched. |

**Key constants**

* All variants exploit unsanitised content ending up in an *executable* context (HTML, attribute, URL, JS string, CSS).
* Output-encoding appropriate to the *sink* is the universal mitigation, backed by a tight `Content-Security-Policy`.
* `HttpOnly` cookies cut off one common goal (session theft) but not others (defacement, phishing, keylogging).

**Detection quick-tips**

* Reflected - grep logs for `<script>` in query strings.
* Stored / Blind - fuzz stateful inputs, monitor for callbacks.
* DOM - run static analysis (`eslint-plugin-xss`) and dynamic scanners (`DOM Invader`).

Patch the fundamentals - encode + CSP - then treat each variant's quirks during testing.

!!! info "External Resources"
    - [Cross-Site Scripting Prevention Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Cross-Site_Scripting_Prevention_Cheat_Sheet.html) (OWASP)
    - [DOM-Based XSS Prevention Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/DOM_based_XSS_Prevention_Cheat_Sheet.html) (OWASP)
    - [XSS Filter Evasion Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/XSS_Filter_Evasion_Cheat_Sheet.html) (OWASP)

## Injection Flaws

| Injection type        | Where attacker sends payload                | Typical goal & impact                    | Classic payload snippet                    | Primary defences                                                   |
|-----------------------|---------------------------------------------|------------------------------------------|--------------------------------------------|--------------------------------------------------------------------|
| **SQL Injection (SQLi)**     | Query parameters, form fields, HTTP headers | Dump/alter database, bypass auth, RCE via xp_cmdshell. | `' OR 1=1--`                                   | Prepared statements / ORM bind variables, least-priv DB user, input allow-lists, WAF. |
| **NoSQL / Mongo Injection**  | JSON body or query string consumed by NoSQL driver | Read/modify NoSQL docs, escalate role.   | `{ "$ne": null }`                              | Use driver-level parameter APIs, stringify user input, schema validation, limit operators. |
| **Command / Shell Injection**| Any value passed to OS shell (`exec`, `system`) | Execute arbitrary commands, elevate privileges. | `; rm -rf /`                                   | `execve()` with argument array, allow-list commands, escape/quote, run as non-root. |
| **LDAP Injection**           | Filters in directory search/bind      | Read/modify directory entries, bypass login. | `*)(uid=*)`                                    | Use parameterised LDAP APIs, escape RFC-4515 special chars, least-priv LDAP creds. |
| **Path / File Inclusion**    | File paths, `include()` arguments     | Read sensitive files, RCE via uploaded script. | `../../../../etc/passwd`                       | Resolve to canonical path then compare to base dir, store uploads outside web root, use allow-list extensions. |
| **XML / XXE**                | XML documents parsed by server        | Read files, SSRF via external entities.  | `<!ENTITY x SYSTEM "file:///etc/passwd">`      | Disable external entities (`XXE`), prefer JSON, use parsing libraries with `secureProcessing`. |
| **Expression / Template Injection (SSTI)** | Unescaped template delimiters (`{{ }}` `#{}`) | Execute code on the server, leak secrets. | `{{7*7}}`                                       | Configure template engine sandbox, escape user input, strict template compilation modes. |
| **Log Injection**            | Values logged without sanitisation    | Forge log entries, inject newlines - log poisoning. | `User=admin\n[ERROR] ...`                       | Replace control chars, structured logging (JSON), encode before log write. |

---

*Root causes*

- Concatenating untrusted data into **code**, **queries**, **paths**, **commands**, or **log lines**.
- Missing contextual escaping / validation.
- Over-privileged service accounts amplifying the blast radius.

*Universal mitigation playbook*

1. **Parameterise, don't concatenate** - use prepared statements or driver APIs.
2. **Constrain identities** - DB, LDAP, or OS users should have minimum privileges.
3. **Input allow-lists** - reject unexpected types, formats, ranges early.
4. **Contextual output encoding** - SQL quotes, shell escapes, LDAP escapes, etc.
5. **Security-focused code reviews + fuzzers** - automate detection of risky string ops.
6. **Runtime protections** - WAF/IDS signatures for common injection payloads.

Treat any spot where user data meets an interpreter as toxic until you've isolated or neutralised it.

!!! info "External Resources"
    - [SQL Injection Prevention Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/SQL_Injection_Prevention_Cheat_Sheet.html) (OWASP)
    - [OS Command Injection Defense Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/OS_Command_Injection_Defense_Cheat_Sheet.html) (OWASP)
    - [XML External Entity Prevention Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/XML_External_Entity_Prevention_Cheat_Sheet.html) (OWASP)

## SSRF & Deserialization

For SSRF details, see the Common Web Vulnerabilities section above. For deserialization, refer to the Deserialization of Untrusted Data entry above.

Key additional notes on SSRF:

- **Cloud metadata endpoints** (`169.254.169.254`) are the highest-value SSRF target in cloud environments. AWS IMDSv2 requires a PUT with a hop-limit header, which mitigates many SSRF vectors.
- **DNS rebinding** can bypass allow-lists that resolve hostnames at request time. Resolve once, validate, then use the resolved IP.
- **Redirect chains** can bypass URL validation - follow redirects cautiously or disable them entirely for user-supplied URLs.

Key additional notes on Deserialization:

- **Language-specific risks**: Java (`ObjectInputStream`), Python (`pickle`), PHP (`unserialize`), Ruby (`Marshal.load`) all have known gadget chains.
- **Detection**: Monitor for serialised object markers in input (e.g., `rO0` for Java, `O:` for PHP).
- **Alternative formats**: Prefer JSON, Protocol Buffers, or MessagePack over native serialisation.

!!! info "External Resources"
    - [Server-Side Request Forgery Prevention Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Server_Side_Request_Forgery_Prevention_Cheat_Sheet.html) (OWASP)
    - [Deserialization Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Deserialization_Cheat_Sheet.html) (OWASP)
    - [A New Era of SSRF](https://portswigger.net/research/top-10-web-hacking-techniques-of-2017#702702) (PortSwigger)
