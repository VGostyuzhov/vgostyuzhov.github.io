# DNS: A Study Guide

The Domain Name System (DNS) publishes records under names. Resolving an IP address, authenticating the DNS answer, encrypting the resolver connection, and authenticating the destination service are separate checks. Study [packet paths](index.md#follow-one-https-request) before the attack scenarios below.

## Basics and Core Concepts

A host's **stub resolver** normally asks a configured **recursive resolver** to obtain an answer. On a cache miss, the recursive resolver follows **delegations**: root servers identify a top-level domain's servers, those servers identify the domain's **authoritative servers**, and the authority returns the relevant records. The recursive resolver makes iterative queries on the client's behalf; the stub does not normally contact every authority itself.

```text
Stub                 Recursive resolver          Authoritative hierarchy
 |-- api.example A -->|                           |
 |                   |-- root/TLD referrals ---->|
 |                   |-- api.example A --------->| example authority
 |                   |<-- A 192.0.2.80, TTL 300 --|
 |<-- cached answer --|                           |
# A subsequent query can use remaining cache lifetime instead of this full path.
# Addresses and outputs here are documentation fixtures, not live observations.
```

A resolver also matches a response to an outstanding query: transaction ID, name, type, class, expected server address/port, and its own destination address/port. Source-port/ID unpredictability reduces off-path forgery opportunities but is not cryptographic authentication; an on-path adversary has a different capability. DNSSEC validation adds a separate authenticated-data check.

DNS uses UDP and TCP on port 53. A **truncated response** (`TC=1`), not an oversized domain name, normally causes a UDP client to retry over TCP. DNS over TCP is also usable without initial UDP truncation. Blocking TCP/53 can break legitimate answers, including larger DNSSEC responses; TCP is not only for zone transfers. EDNS allows a client to advertise a larger UDP response capacity, but large fragmented responses can still be unreliable.

```bash
# Query only a resolver you are permitted to use; outputs vary by environment.
dig @192.0.2.53 api.example A
# Request TCP explicitly to distinguish transport policy from record existence.
dig @192.0.2.53 api.example A +tcp
# +dnssec requests DNSSEC records; it does not make dig validate a trust chain.
dig @192.0.2.53 api.example A +dnssec
```

!!! info "External Resources"
    - [DNS Concepts and Delegation](https://www.rfc-editor.org/rfc/rfc1034.html) (IETF)
    - [DNS Response Matching](https://datatracker.ietf.org/doc/html/rfc5452#section-9.1) (IETF)
    - [DNS Transport over TCP](https://www.rfc-editor.org/rfc/rfc7766.html) (IETF)

## Cache Lifetimes and Changes

A record's **time to live (TTL)** controls how long it may normally be reused in a cache. It is not a global propagation countdown. Different clients queried at different times, retain separate caches, and can have application connection pools that continue using an earlier address.

```text
12:00:00  Resolver caches api.example -> 192.0.2.80 with TTL 300
12:01:00  Operator changes authoritative answer to 192.0.2.81
12:02:00  Resolver can still return 192.0.2.80 with about 180s remaining
12:05:00  Normal cached record expires; a new lookup can fetch 192.0.2.81
# Lowering the authoritative TTL at 12:01 does not rewrite the old cached TTL.
# This trace assumes ordinary caching, no explicit invalidation or serve-stale.
```

**Negative caching** retains evidence that a name or requested record does not exist. Creating the record later may not immediately fix clients that cached the earlier negative answer. An authoritative negative response carries an SOA; the initial negative-cache TTL is the smaller of that SOA's TTL and SOA MINIMUM, then counts down subject to resolver policy. For example, SOA TTL 600 and MINIMUM 300 yield an initial 300-second negative lifetime, not 600 seconds. Some resolvers deliberately **serve stale** data during authority outages under configured limits; document that exception before treating TTL as a hard operational cutover guarantee.

For a migration, lower TTL far enough in advance for previous cache lifetimes to expire, retain both working destinations during the transition, and test through representative resolvers. DNS changes do not revoke sessions or guarantee that every client has switched. Very short TTLs increase resolver/authority load and outage dependence; choose them from required change speed and capacity rather than a universal number.

!!! info "External Resources"
    - [Negative Caching](https://www.rfc-editor.org/rfc/rfc2308.html) (IETF)
    - [Serving Stale DNS Data](https://www.rfc-editor.org/rfc/rfc8767.html) (IETF)

## DNSSEC, DoT, and DoH

**DNSSEC** adds signatures and a delegation-based chain of trust so a validating resolver can authenticate covered DNS data. The zone signs record sets; the parent's DS record links to the child's DNSKEY, starting from a configured trust anchor. A valid signature authenticates published data, not whether a destination is benign.

A validator distinguishes **secure** validated data, **insecure** data under an unsigned delegation, **bogus** data that fails required validation, and **indeterminate** status when it cannot establish the necessary security state. DNSSEC does not mean every unsigned domain must fail. Broken key rollover or DS configuration can make a signed domain fail resolution.

**DNS over TLS (DoT)** and **DNS over HTTPS (DoH)** encrypt the connection to a resolver and authenticate that endpoint through the client's TLS policy. The chosen resolver can still see the query. Encrypting the stub-to-resolver hop does not automatically protect resolver-to-authority traffic or validate zone signatures.

| Mechanism | Protects | Does not establish |
|---|---|---|
| DNSSEC validation | Integrity/authenticity of signed DNS data | Query confidentiality or application safety |
| DoT / DoH | Privacy and integrity on the encrypted resolver hop | Honesty of that resolver or DNSSEC validity by itself |
| HTTPS destination validation | Service identity and protected application channel | That the DNS answer was signed, or that the service is authorized for an operation |

**Example:** An on-path attacker changes `api.example` to their address. Correct HTTPS hostname/certificate validation still rejects an impostor lacking acceptable credentials for `api.example`. If the attacker controls the real zone and legitimately changes its signed address, DNSSEC can validate the maliciously administered answer. The relevant trust boundaries differ.

!!! info "External Resources"
    - [DNSSEC Security Services and Limitations](https://www.rfc-editor.org/rfc/rfc4033.html) (IETF)
    - [DNS over TLS](https://www.rfc-editor.org/rfc/rfc7858.html) (IETF)
    - [DNS over HTTPS](https://www.rfc-editor.org/rfc/rfc8484.html) (IETF)

## Data Exfiltration and Resolver Policy

A compromised endpoint can put data in query names such as `chunk01.collector.example`. A recursive resolver that follows that delegation can deliver the name to the attacker-controlled authority even if the endpoint cannot directly contact that authority. Restricting clients to one approved resolver alone does not prevent this path: the approved resolver also needs policy and monitoring.

```text
Endpoint -> approved resolver: QNAME=chunk01.collector.example, QTYPE=TXT
Resolver -> collector authority: same query name on a cache miss
# Data can be in QNAME regardless of whether QTYPE is TXT, A, or another type.
# Labels/total DNS name length limit each chunk; an example name is not a tunnel.
```

Monitor query volume, unusual label length/entropy, newly observed domains, response patterns, and endpoint context. Long names and TXT records also have legitimate uses; one feature alone is weak evidence. Enforce approved resolver paths, control unauthorized encrypted DNS where policy requires it, and correlate resolver logs with endpoint activity. HTTPS hides URL paths from passive flow monitoring; an authorized terminating HTTP proxy/application can have richer logs.

### DNS Sinkhole

A **sinkhole** deliberately overrides answers for selected names through a controlled DNS policy or changes to infrastructure under defensive control. A policy resolver might return a controlled address, NXDOMAIN, or another configured response. This is not equivalent to accepting forged third-party DNS data.

It can prevent or observe some command-and-control lookups, but direct IP connections, cached answers, alternative resolvers, and unrelated domains can bypass it. An infected host contacting a sinkholed name supplies an investigation lead, not automatic proof that every such query is malicious. Document policy behavior when it overrides DNSSEC-protected names rather than presenting a locally changed answer as authentic zone data.

!!! info "External Resources"
    - [Application-Layer Protocol: DNS](https://attack.mitre.org/techniques/T1071/004/) (MITRE ATT&CK)
    - [DNS-Based Policy and Filtering Guidance](https://www.rfc-editor.org/rfc/rfc7754.html) (IETF)

## DNS Record Types Cheat Sheet

| Record | Purpose | Security interpretation |
|---|---|---|
| A / AAAA | IPv4 / IPv6 destination address | Check both families; one permitted address does not authorize all returned addresses |
| CNAME | Alias to another name | Follow the chain; stale service references can create takeover risks |
| NS | Delegates authority | Protect registrar and authoritative configuration |
| MX | Mail destination | Transport and sender-authentication policies remain separate |
| PTR | Reverse lookup | A returned name alone is not authenticated client identity |
| SOA | Zone administration and timing information | Includes parameters used in negative-cache handling |
| TXT | Application-specific text, including SPF/DKIM/DMARC | Interpretation depends on the consuming protocol |
| DS / DNSKEY / RRSIG | DNSSEC delegation, keys, and signatures | Need a valid chain and correct rollover, not just a signature's presence |
| HTTPS / SVCB | Service connection parameters | Client behavior depends on supported record parameters and trust policy |

!!! info "External Resources"
    - [DNS Resource Records](https://www.rfc-editor.org/rfc/rfc1035.html#section-3) (IETF)
    - [SVCB and HTTPS Resource Records](https://www.rfc-editor.org/rfc/rfc9460.html) (IETF)
