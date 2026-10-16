# Tricky Network Security Interview Questions

Fourteen scenarios combine seven cloud/Kubernetes networking questions with seven core questions on DNS, TLS, state, and availability. Read [core networking](index.md), then the linked teaching sections. Explain the mechanism before expanding the answer. Each worked example is a separate expander with commented commands, configuration, code, or protocol traces.

Examples use documentation addresses and synthetic data. Commands are templates for an explicitly authorized lab, not reports of live scans. Pseudocode and conceptual configurations are labeled; no firewall changes or attack traffic are executed by this page.

## 1. Why Can an AWS Security Group Allow a Connection While a NACL Breaks It?

**Study first:** [Cloud security groups and subnet ACLs](segmentation.md#cloud-security-groups-and-subnet-acls).

**Scenario:** An EC2 database security group allows an application to connect on TCP/5432. The database subnet NACL allows incoming TCP/5432 but denies outgoing client ephemeral ports. Why does the connection still fail?

??? note "Answer guidance and common trap"
    Security groups and NACLs are independent enforcement points. The stateful security group permits matching replies to an allowed request; the stateless NACL still evaluates the reply as traffic leaving the subnet. Its rule must cover the actual reply destination port, not just the database's listening port. Also inspect routes, the client's subnet boundary, other security groups, and the host/application.

    **Trap:** “Stateful security group” is assumed to make every filter on the path stateful.

    **Test:** Trace the SYN and SYN/ACK tuples and inspect both subnet-boundary decisions. Permit only the necessary return ranges under the actual client configuration, then retest the intended application path.

??? example "Worked example"
    Illustrative AWS rule decisions, not an importable rule file. Assume routes, source-side controls, and the database listener otherwise permit the flow.

    ```text
    App 10.0.1.8:51000 -> DB 10.0.2.9:5432/TCP SYN
      DB security group: inbound 5432 from approved app -> allow
      DB-subnet NACL:    inbound destination 5432      -> allow
    DB 10.0.2.9:5432 -> App 10.0.1.8:51000/TCP SYN,ACK
      DB security group: matching reply               -> allow by state
      DB-subnet NACL:    outbound destination 51000    -> deny if not allowed
    # A NACL outbound rule for destination 5432 does not match this reply.
    # Use actual client ephemeral ranges; do not assume one universal range.
    ```

    Security-group allow rules aggregate; a narrower extra group cannot cancel a broader group's allow. NACLs instead evaluate ordered allow/deny rules at the subnet boundary.

**Follow-up:** Would a private subnet or a cloud private endpoint authorize every database/service operation? No; routing and private reachability are separate from IAM and application permissions.

## 2. Does IMDSv2 Make an EKS Pod Unable to Obtain the Node Role?

**Study first:** [Instance metadata and workload identity](segmentation.md#instance-metadata-and-workload-identity).

**Scenario:** An EKS node requires IMDSv2, and a pod has its own IRSA role. The compromised pod can still reach node metadata and perform arbitrary HTTP requests. Is the node role automatically isolated from it?

??? note "Answer guidance and common trap"
    No. IMDSv2 requires a metadata session-token exchange; it is not application-specific IAM authorization. A process able to perform the required PUT and subsequent requests can still access available metadata through a permitted path. Configuring IRSA does not itself block other credential sources, and hostNetwork changes the path assumptions.

    Use tested metadata restrictions, least-privilege node roles, and supported workload identities. Security groups/NACLs do not filter EC2 IMDS traffic. The PUT response hop limit affects token delivery across hops and container compatibility, not the caller's IAM identity.

    **Trap:** IMDSv2 or an assigned workload role is treated as proof that no other role is reachable.

    **Test:** In an authorized test environment, check configuration and harmless metadata reachability from each relevant pod mode without reading or logging live role credentials.

??? example "Worked example"
    Synthetic configuration and request trace; no live metadata token or IAM credentials are shown.

    ```json
    {
      "MetadataOptions": {
        "HttpEndpoint": "enabled", "HttpTokens": "required",
        "HttpPutResponseHopLimit": 1
      }
    }
    ```

    ```text
    Tokenless GET -> metadata denied when tokens are required
    Permitted PUT /latest/api/token -> metadata session token
    Permitted GET + that token     -> metadata can be returned
    # The metadata session token is not the pod's IRSA credential or an IAM scope.
    # Hop limit 1 can affect an extra routed pod hop; hostNetwork may avoid that hop.
    # Treat the node's available role as another capability to isolate and minimize.
    ```

    This scenario assumes the token-response path actually works; requiring IMDSv2 alone does not demonstrate that it cannot.

**Follow-up:** What if IPv4 metadata is blocked but the IPv6 endpoint is enabled? Validate both configured paths and the actual SDK credential selection.

## 3. Why Did a Deny Rule Leave an Existing Connection Alive?

**Study first:** [State and rule changes](segmentation.md#stateful-and-stateless-filtering) and [rollout verification](segmentation.md#change-outage-and-verification).

**Scenario:** A database connection is already established. An operator adds a deny for new application-to-database connections, but queries on the old connection continue. Did the firewall ignore the change?

??? note "Answer guidance and common trap"
    Not necessarily. An earlier existing-flow accept or session-based policy can allow the old connection while rejecting new ones. Define whether the requirement covers new connections or all access, then handle affected flow state and application sessions accordingly. Distribution to every enforcing path and failover device also matters.

    **Trap:** Updating a new-flow policy is assumed to terminate every retained session.

    **Test:** Compare a fresh connection with a pre-existing one after the change; measure the required boundary across all paths.

??? example "Worked example"
    Ordered-chain excerpt, not a complete deployable ruleset:

    ```nft
    ct state established,related accept  # Existing matching packets accepted here.
    ip saddr 10.0.20.0/24 tcp dport 5432 drop
    # New connections reach the drop; old established packets matched above it.
    ```

    ```text
    Before change: connection C1 opens and executes SELECT
    After change:  connection C2 cannot open, but C1 still executes SELECT
    # A stop-all-access contract needs coordinated targeted termination/rejection,
    # not an indiscriminate flush of unrelated state.
    ```

**Follow-up:** Can termination undo a database transaction already committed? No; state the execution and business-policy boundary.

## 4. Why Do Old DNS Answers Survive a TTL Reduction?

**Study first:** [DNS cache lifetimes](dns.md#cache-lifetimes-and-changes).

**Scenario:** An operator changes the authoritative TTL from 300 to 30 seconds and changes the service address. Some clients still connect to the old address two minutes later. Is DNS broken?

??? note "Answer guidance and common trap"
    The earlier cached answer can retain its original remaining lifetime. Reducing TTL at the authority does not rewrite every resolver's existing cache. Negative caching, application caches, connection pools, and configured serve-stale behavior also affect observations. Plan overlap and lower TTL in advance rather than promising a global immediate cutover.

    **Trap:** TTL is treated as one global clock reset by the operator's edit.

    **Test:** Compare authoritative and recursive answers, remaining TTL, and whether the client reused an already open connection.

??? example "Worked example"
    Synthetic dig-style answers showing separate snapshots:

    ```text
    # Recursive cache filled at 12:00 with TTL 300.
    api.example. 300 IN A 192.0.2.80
    # Authority changed at 12:01 to .81 with TTL 30.
    # The same recursive cache at 12:02 can still answer:
    api.example. 180 IN A 192.0.2.80
    # Authority at 12:02:
    api.example.  30 IN A 192.0.2.81
    ```

    ```bash
    # Authorized lab resolver/authority only; results depend on their actual state.
    dig @192.0.2.53 api.example A +noall +answer
    # TCP helps diagnose transport; it does not force a recursive cache refresh.
    dig @192.0.2.53 api.example A +tcp +noall +answer
    ```

**Follow-up:** What if the record did not previously exist? Explain negative caching and its separate lifetime.

## 5. Do DoH and DNSSEC Prove a Destination Is Safe?

**Study first:** [DNSSEC versus encrypted DNS](dns.md#dnssec-dot-and-doh).

**Scenario:** A client uses DoH and receives a DNSSEC-valid address for a malicious domain controlled by its attacker. Should either mechanism have blocked it?

??? note "Answer guidance and common trap"
    Neither mechanism judges whether a domain is benign. DoH protects/authenticates the resolver connection; DNSSEC validates covered DNS data under its trust chain. A malicious operator can legitimately sign their own records. Service authentication, allowed-destination policy, and application authorization remain separate.

    **Trap:** Encryption and signature validity are equated with trustworthy content or business authority.

    **Test:** Distinguish altered signed records that fail validation from correctly signed maliciously administered records.

??? example "Worked example"
    Conceptual validation outcomes, not live resolver output:

    ```text
    DoH endpoint TLS:        valid for configured resolver
    Zone signature chain:    valid for attacker.example
    Answer:                  attacker.example -> 192.0.2.66
    Content safety:          not established by either check

    Signed zone altered by an on-path forger -> bogus under required validation
    Unsigned delegation                     -> insecure, not automatically bogus
    # dig +dnssec asks for DNSSEC records; it is not local trust-chain validation.
    ```

    A resolver's reported validation flag is useful only over a resolver relationship/path the client trusts.

**Follow-up:** Does DoH hide the query from the resolver itself? No; identify which hop is encrypted and who terminates it.

## 6. Why Can DNS Exfiltration Use the Only Approved Resolver?

**Study first:** [DNS exfiltration and resolver policy](dns.md#data-exfiltration-and-resolver-policy).

**Scenario:** Endpoints cannot send DNS directly to the Internet, but may query the corporate resolver. A compromised host encodes data under an attacker-controlled delegated domain. Can the data still reach the attacker?

??? note "Answer guidance and common trap"
    Yes, if the corporate resolver follows the delegation and its policy permits it. The host uses the resolver as its normal path to the external authority. A client egress allow-list alone does not constrain all query content or authority destinations. Combine resolver policy, anomaly/context monitoring, endpoint containment, and control of alternate resolver paths.

    **Trap:** DNS transport centralization is mistaken for exfiltration prevention.

    **Test:** In a controlled lab, trace a synthetic name from endpoint through resolver to the authorized test authority, without transferring real secrets.

??? example "Worked example"
    Synthetic query carrying harmless test text:

    ```text
    Endpoint -> corporate resolver: QNAME=hello.chunk01.collector.example, QTYPE=A
    Resolver -> test authority:     QNAME=hello.chunk01.collector.example, QTYPE=A
    # The authority can observe QNAME even though the endpoint never contacted it.
    # TXT is not required for data carried in the query name.
    ```

    ```python
    # Detection pseudocode: one feature alone is not a verdict.
    if unusual_labels(query) and new_destination(query) and suspicious_endpoint(host):
        investigate(query, host)  # Legitimate CDNs/telemetry can also use long labels.
    ```

    Blocking the test domain at the recursive policy layer should stop that permitted-resolution path; it does not prove that all other exfiltration channels are closed.

**Follow-up:** What changes when a browser uses an unauthorized DoH resolver? Reassess endpoint policy, egress paths, and available telemetry.

## 7. Why Did a Kubernetes Default-Deny Policy Change Nothing?

**Study first:** [NetworkPolicy enforcement and directions](segmentation.md#kubernetes-networkpolicy-enforcement).

**Scenario:** The API server accepts a default-deny NetworkPolicy in namespace payments, but new ordinary pod-to-pod connections still succeed. Does successful YAML creation prove isolation?

??? note "Answer guidance and common trap"
    No. The networking implementation must support and enforce NetworkPolicy. Confirm that capability first, then verify namespace, target selectors, directions, and other applicable policies. Standard policies combine permits, so a broad allow can reopen access even when a default-deny object exists.

    **Trap:** An API object is mistaken for evidence that the dataplane enforced it.

    **Test:** Use fresh allowed/denied connections from named test pods on same-node and cross-node paths. Check both source egress and destination ingress where isolated; inspect hostNetwork and existing-flow cases separately.

??? example "Worked example"
    A complete baseline policy for a lab namespace that already exists:

    ```yaml
    apiVersion: networking.k8s.io/v1
    kind: NetworkPolicy
    metadata:
      name: default-deny
      namespace: payments
    spec:
      podSelector: {}  # Select every pod in payments, not every pod in the cluster.
      policyTypes: [Ingress, Egress]
      ingress: []
      egress: []
    # Effective only with a supporting network implementation.
    # Other matching policies can still add allowed traffic.
    ```

    ```bash
    # Read-only inspection templates for an authorized lab context.
    kubectl -n payments get networkpolicy -o yaml
    kubectl -n payments get pods -o wide --show-labels
    # Stored YAML and labels help diagnose intent; packet tests demonstrate enforcement.
    ```

**Follow-up:** If only Ingress is isolated, does that deny new outgoing pod connections? No; Egress isolation is independent, and the destination may have its own ingress policy.

## 8. Why Reject a Certificate with a Valid Trusted Chain?

**Study first:** [TLS reference identity checks](tls.md#service-identity-validation).

**Scenario:** A certificate chains to a trusted root and is within its validity period, but its SAN is `attacker.example`. The client requested `api.example`. Why should the connection fail?

??? note "Answer guidance and common trap"
    Chain validation and service-identity matching are independent checks. The CA's signature does not authorize the key for every hostname. Match the intended reference identity against the appropriate SAN, enforce usage/constraints/time/trust policy, and verify the handshake proof. SNI is routing input, not a substitute for hostname verification.

    **Trap:** A successful handshake or displayed certificate is assumed to prove the requested identity.

    **Test:** Keep the trusted CA fixed; the correct-name leaf succeeds and the wrong-name leaf fails.

??? example "Worked example"
    OpenSSL 3.x template for an authorized lab endpoint and CA fixture:

    ```bash
    openssl s_client -connect 192.0.2.80:443 -servername api.example \
      -verify_hostname api.example -verify_return_error -CAfile lab-ca.pem </dev/null
    # -servername selects virtual hosting; -verify_hostname checks the intended name.
    # -verify_return_error prevents treating verification failure as a successful test.
    ```

    ```text
    Trusted chain + SAN=api.example       -> name check can pass
    Trusted chain + SAN=attacker.example  -> reject hostname mismatch
    # Other checks can still reject the correct-name certificate independently.
    ```

**Follow-up:** If the server sends its own root certificate, should that root become trusted? No; trust anchors come from configured verifier policy.

## 9. Is a GET Request Automatically Safe in TLS 0-RTT?

**Study first:** [Early data and replay](tls.md#0-rtt-resumption-tls-13).

**Scenario:** A service accepts an early-data GET that starts an expensive export. The same protected request is processed in two regions. Did encryption fail?

??? note "Answer guidance and common trap"
    No. Early data can be validly protected yet replayed; encryption does not establish unique processing. Method names alone do not make the operation replay-safe. Disable early data for unsafe operations or implement supported rejection/retry and coordinated application deduplication at the actual side-effect boundary. A local cache in only one region cannot guarantee global uniqueness.

    **Trap:** “GET” or “encrypted” is treated as a proof of harmless duplicate processing.

    **Test:** Replay a harmless fixture across the intended accepting regions and verify the specified one-operation or rejection behavior.

??? example "Worked example"
    Application pseudocode; transport supplies authenticated early-data metadata:

    ```python
    if transport.is_early_data and not operation.is_replay_safe:
        reject_early_data_for_supported_retry()  # Do not execute the export here.
    else:
        execute_under_normal_authorization(operation)
    # For operations requiring once-only effects, coordinate an idempotency record
    # with execution across regions; a caller-provided key alone is not enforcement.
    ```

    ```text
    Ticket T + early request -> region A -> possible export
    Same early request      -> region B -> possible duplicate export
    # Demonstration trace; no live replay or production side effects are performed.
    ```

**Follow-up:** Does forward secrecy automatically cover early data? No; analyze PSK and ephemeral-DH modes separately.

## 10. Did One YAML Dash Open Access from the Wrong Kubernetes Namespace?

**Study first:** [NetworkPolicy selectors and additive rules](segmentation.md#kubernetes-networkpolicy-enforcement).

**Scenario:** An engineer intends to allow payments only from app=checkout pods in namespace frontend. They place namespaceSelector and podSelector in separate from-list entries. What did they allow instead?

??? note "Answer guidance and common trap"
    Separate peer entries are alternatives. One entry permits the selected frontend namespace; the standalone podSelector entry selects matching pods in the policy's own namespace, payments. Put both selectors in one peer to require their intersection. Also audit other applicable policies: a narrower policy cannot override a separate broad allow.

    **Trap:** Similar indentation is assumed to express the same boolean condition.

    **Test:** A frontend checkout pod is permitted, while a frontend non-checkout pod and a payments checkout pod are not permitted by this intended ingress rule. Assume other directional/transport prerequisites are satisfied when testing the selector distinction.

??? example "Worked example"
    Complete ingress policy; assumes a supporting implementation, trustworthy labels, and no other broad allow:

    ```yaml
    apiVersion: networking.k8s.io/v1
    kind: NetworkPolicy
    metadata:
      name: checkout-to-payments
      namespace: payments
    spec:
      podSelector:
        matchLabels: {app: payments}
      policyTypes: [Ingress]
      ingress:
        - from:
            - namespaceSelector:
                matchLabels: {kubernetes.io/metadata.name: frontend}
              podSelector:  # Same peer: namespace AND pod selector must match.
                matchLabels: {app: checkout}
          ports:
            - protocol: TCP
              port: 8080
    ```

    ```yaml
    # Incorrect replacement for the from field above (fragment, not a whole policy):
    from:
      - namespaceSelector:
          matchLabels: {kubernetes.io/metadata.name: frontend}
      - podSelector:  # Second peer: OR, scoped to payments for this standalone selector.
          matchLabels: {app: checkout}
    ```

    Protect permissions to change workload labels, create matching pods, and edit policies; a label match is not cryptographic service-account identity.

**Follow-up:** Would adding default-deny after the broad allow fix it? No; standard NetworkPolicy permits are additive rather than ordered deny-overrides.

## 11. Why Did Host DDoS Filtering Not Restore Availability?

**Study first:** [Match DDoS controls to resources](attacks.md#match-the-defense-to-the-exhausted-resource).

**Scenario:** A host drops attack packets correctly, but the upstream link is saturated and legitimate users cannot connect. Should increasing the host rule rate fix it?

??? note "Answer guidance and common trap"
    Not if the exhausted resource is the upstream link. Packets consumed scarce capacity before reaching the host. Obtain appropriately placed upstream mitigation/capacity; use host controls for resources they can actually protect. SYN state, application CPU, database work, and link bandwidth need different controls. Restrict direct origin bypass when relying on an edge service.

    **Trap:** A working local drop counter is treated as end-to-end availability protection.

    **Test:** Measure link utilization/loss and host/application state to locate the exhausted resource before tuning.

??? example "Worked example"
    Synthetic capacity trace:

    ```text
    Attack offered load: 10 Gbit/s
    Victim access link:   1 Gbit/s
    Host firewall:        drops observed attack packets
    User result:          timeout due to upstream saturation
    # Host filtering cannot recover capacity already consumed on that access link.
    ```

    ```text
    Client -> CDN/scrubber -> origin  # Intended protected path
    Client -------------> origin    # Direct bypass if still reachable/permitted
    # Test origin reachability from an authorized external vantage, not only CDN DNS.
    ```

**Follow-up:** Why can per-IP limits harm legitimate users? Shared NAT aggregates users, while distributed attackers can spread requests across sources.

## 12. Why Did Default-Deny Egress Break Kubernetes Service Discovery?

**Study first:** [DNS under egress isolation](segmentation.md#dns-under-egress-isolation) and [DNS transport](dns.md#basics-and-core-concepts).

**Scenario:** An application has default-deny egress plus an allow for its database port. It can reach a test database address directly but cannot resolve the database Service name. What path is missing?

??? note "Answer guidance and common trap"
    The application needs its actual resolver path as well as database traffic. For a conventional CoreDNS pod path, allow the right destination and both UDP/53 and TCP/53. A Service address may be translated before or after policy evaluation depending on implementation. NodeLocal DNSCache can use another pod-facing address/path, so copying a CoreDNS selector is not sufficient for every cluster.

    **Trap:** Allowing the final service port is assumed to permit its prerequisite DNS query.

    **Test:** Inspect the pod's resolver configuration and destination labels/path, then test resolution and the application connection separately. A DNS permit neither prevents DNS exfiltration nor authorizes all resolved destinations.

??? example "Worked example"
    Additional DNS permit for an egress-isolated application. Assumes ordinary CoreDNS pods in kube-system with label k8s-app=kube-dns and a plugin/path where this selector matches; verify these facts in the lab.

    ```yaml
    apiVersion: networking.k8s.io/v1
    kind: NetworkPolicy
    metadata:
      name: app-dns-egress
      namespace: payments
    spec:
      podSelector:
        matchLabels: {app: payments}
      policyTypes: [Egress]
      egress:
        - to:
            - namespaceSelector:
                matchLabels: {kubernetes.io/metadata.name: kube-system}
              podSelector:
                matchLabels: {k8s-app: kube-dns}
          ports:
            - {protocol: UDP, port: 53}
            - {protocol: TCP, port: 53}  # Larger/truncated answers may need TCP.
    # Existing database egress permit is separate; this rule does not grant it.
    ```

    With NodeLocal DNSCache, inspect the resolver address and enforced path instead of assuming this manifest covers it. Avoid broad all-destination port-53 access merely to make name resolution work.

**Follow-up:** Can standard NetworkPolicy permit HTTPS only to api.vendor.example by hostname? Not directly; choose supported FQDN-aware enforcement or an explicitly constrained proxy, and account for DNS and address changes.

## 13. Does Strict Mesh mTLS Stop Every Other Service from Calling Payments?

**Study first:** [Mesh identity versus authorization](segmentation.md#mesh-identity-and-traffic-entry-points).

**Scenario:** An Istio sidecar-mode payments workload requires STRICT mTLS. Another enrolled workload has a valid mesh certificate and calls it successfully. Did mTLS fail?

??? note "Answer guidance and common trap"
    Not necessarily. STRICT requires authenticated protected transport on covered paths; it does not by itself restrict callers to the intended workload identity or operation. Attach an applicable AuthorizationPolicy and verify the actual sidecar interception/policy path. Where no ALLOW policy applies, Istio does not impose a caller allow-list merely because STRICT is enabled. Where an ALLOW policy applies, a request must match an applicable allow after other policy evaluation.

    **Trap:** Every authenticated mesh principal is treated as an authorized caller.

    **Test:** An approved workload/method/path succeeds, a different valid mesh principal is denied, and plaintext fails on the covered path. Application tenant/object checks remain separate.

??? example "Worked example"
    Istio sidecar-mode HTTP example. Assumes destination labels app=payments, HTTP port 8080, the cluster.local trust domain, and no other broad ALLOW policy. Ambient/waypoint configuration requires separate treatment.

    ```yaml
    apiVersion: security.istio.io/v1
    kind: PeerAuthentication
    metadata:
      name: payments-mtls
      namespace: payments
    spec:
      selector:
        matchLabels: {app: payments}
      mtls:
        mode: STRICT  # Require mTLS; not an application caller allow-list.
    ---
    apiVersion: security.istio.io/v1
    kind: AuthorizationPolicy
    metadata:
      name: checkout-charge
      namespace: payments
    spec:
      selector:
        matchLabels: {app: payments}
      action: ALLOW
      rules:
        - from:
            - source:
                principals: [cluster.local/ns/frontend/sa/checkout]
          to:
            - operation:
                ports: ["8080"]
                methods: [POST]
                paths: [/charge]
    # Principal comes from authenticated peer identity, not a caller-supplied header.
    ```

    A different certificate-valid mesh service fails this allow rule. An approved checkout service still cannot charge an arbitrary user's account without the application's own authorization.

**Follow-up:** What if a workload or excluded port bypasses interception? Verify coverage and complementary network/backend controls; a policy YAML object alone does not prove enforcement on every path.

## 14. Can an Old LoadBalancer Service Bypass a New Kubernetes Gateway?

**Study first:** [Mesh and traffic entry points](segmentation.md#mesh-identity-and-traffic-entry-points) and [direct-origin paths](attacks.md#match-the-defense-to-the-exhausted-resource).

**Scenario:** A new Ingress/Gateway controller enforces login and WAF checks. An old externally reachable LoadBalancer Service still forwards directly to the same application. Its DNS name was removed. Is the gateway now the only entry point?

??? note "Answer guidance and common trap"
    No. The old listener/address can remain reachable without its DNS name. The gateway's controls apply to requests traversing that gateway, not to all possible backend paths. Inventory Services, cloud load balancers, relevant NodePorts, node reachability, and backend policy. Remove or restrict unintended paths while retaining independent application authentication/authorization.

    **Trap:** Routing new users through a gateway is assumed to force all callers through it.

    **Test:** The intended gateway path works with its required credentials, while an authorized external test against the old address/port cannot bypass the security contract. NodePort existence alone does not prove public reachability; confirm the attacker's vantage and cloud/node rules.

??? example "Worked example"
    Read-only inventory templates; these do not remove or change resources:

    ```bash
    kubectl get services -A -o wide
    kubectl get ingress -A
    # Gateway API resources exist only where the CRDs/controller are installed.
    kubectl get gateways.gateway.networking.k8s.io -A
    ```

    ```text
    Intended: external client -> gateway:443 -> ClusterIP Service -> app
    Bypass:   external client -> old public LoadBalancer:443 -----> app
    # This fixture assumes old-LB reachability was confirmed from that vantage.
    # Removing a DNS record does not close the listener or a remembered address.
    # Also inspect provider-specific target/node ports and any remaining direct path.
    ```

    Some load-balancer implementations allocate NodePorts; supported implementations can avoid that. Check actual allocation, reachability, and forwarding rather than assuming one universal topology.

**Follow-up:** Does preserving the client IP make X-Forwarded-For trustworthy? No; validate the proxy chain and strip/untrust caller-supplied identity headers according to the enforced path.

## Self-Assessment

A supported answer identifies the decision, traces the relevant state/message, rejects the incorrect assumption, proposes a control under explicit constraints, and names a positive/negative test. Change one assumption after answering: add IPv6, reuse an old connection, make a validator unavailable, move the observer, or introduce a second region.

These exercises cover this networking section. They do not establish completeness for the site's broader cloud, detection, or general interview collection.
