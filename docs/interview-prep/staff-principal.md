# Staff / Principal Practice: Cloud IAM and Multi-Account Design

These editorial practice scenarios exercise the mechanisms and worked architecture introduced in the first cloud-design batch. They are not employer-confirmed questions or a complete staff/principal curriculum. They focus on reasoning across account boundaries, implementation constraints, failure handling, and evidence that a design works.

Read the linked lessons before opening guidance. For each scenario, state assumptions, trace the permission or credential path, identify a failure, propose a scoped change, and name a test that distinguishes the corrected design from the broken one. A convincing answer explains who operates the control and how it behaves during rollout or an outage.

## 1. Design Security for 200 Accounts

**Study first:** [requirements and account boundaries](../infrastructure-cloud/multi-account-security.md#requirements-and-threat-model), [guarded access](../infrastructure-cloud/multi-account-security.md#guarded-human-and-workload-access), and [operating measures](../infrastructure-cloud/multi-account-security.md#validation-and-operating-measures).

**Scenario:** Forty teams deploy independently across 200 AWS accounts. One team wants a shared production deployment administrator, and another proposes putting central tooling in the management account. Design the account structure and access paths. Explain which compromise remains organization-wide.

??? note "Answer guidance and common trap"
    Separate workload environments and central responsibilities according to their access and recovery needs. Keep routine tooling outside the management account, whose identities are not restricted by SCPs. Use service-specific delegated administration, distinct log storage, and scoped target roles for deployments and response.

    Central automation remains a high-impact dependency; constrain which applications and environments each identity can deploy. A Security OU is a policy grouping, not proof that every role inside it is trustworthy. Explain account onboarding, evidence delivery, and policy rollback ownership as part of the design.

    **Trap:** Drawing account boxes without identifying the roles that cross them.

    **Test:** A compromised nonproduction job must fail production assumption, while the approved production job succeeds and cannot alter audit retention or responder trust.

??? example "Worked example"
    This synthetic access graph is narrower than a complete organization design:

    ```text
    # Approved path:
    payments-production-job -> PaymentsDeploy in production account P

    # Required rejections:
    payments-development-job -> PaymentsDeploy in P       DENY at trust/issuance
    PaymentsDeploy in P -> change Log Archive bucket      DENY at authorization
    PaymentsDeploy in P -> modify its own trust/boundary  DENY at authorization
    ```

    If every team uses one credential allowed to assume all production roles, separate accounts still leave that credential with an organization-wide deployment path. Split the caller identities and review their target permissions.

## 2. An Administrator Policy Still Gets Access Denied

**Study first:** [grants and permission ceilings](../infrastructure-cloud/iam.md#aws-policy-evaluation-grants-and-permission-ceilings) and [SCP inheritance](../infrastructure-cloud/iam.md#scp-inheritance-and-scope).

**Scenario:** A member-account role has an administrator identity policy. Its boundary permits only reads, and its OU has a deny for deleting audit objects. A developer proposes another allow policy and says the boundary also makes all bucket-policy grants safe. Diagnose both claims.

??? note "Answer guidance and common trap"
    Additional identity-policy allows cannot override a boundary restriction or an applicable SCP deny. Identify the actual principal, action, resource, conditions, and applicable policy path before changing anything. SCPs and boundaries do not grant permissions.

    Same-account resource grants directly to a role session have different implicit-deny behavior from grants to a role ARN. An implicit boundary restriction alone is not a universal prohibition. Applicable explicit denies still win; changing the grant route does not cancel the OU's explicit delete deny.

    **Trap:** Treating every policy type as one universal intersection, or treating `FullAWSAccess` as a grant of administrator privileges.

    **Test:** Distinguish a read allowed by the identity grant, a write blocked by the boundary, and audit deletion blocked by explicit deny. Review any separate resource grants.

??? example "Worked example"
    Assume the requests match the stated resources and there are no other restrictions:

    ```text
    Identity policy: Allow *
    Boundary:       Allow s3:GetObject
    OU SCP:         Deny s3:DeleteObject on audit objects

    # Ordinary identity-policy grant path:
    GetObject report.csv  -> allowed
    PutObject report.csv  -> implicit boundary deny
    DeleteObject audit/x  -> explicit SCP deny

    # A child-account SCP Allow s3:DeleteObject does not cancel the parent Deny.
    ```

    A separate same-account grant to the exact session can change the write analysis if no explicit deny applies. It cannot rescue the explicitly denied audit deletion.

## 3. Cross-Account Assumption Works but the Read Fails

**Study first:** [caller, trust, and target action](../infrastructure-cloud/iam.md#trace-caller-trust-and-target-action).

**Scenario:** Account A's build role successfully assumes account B's artifact role. Reading an object then fails. An engineer adds `s3:*` to A's build role. Explain why this may change nothing and how you would investigate without widening access.

??? note "Answer guidance and common trap"
    The first request used A's identity to obtain B's credentials; the second uses B's role session. Inspect the actual credential source, B's role policy, its limits, the requested object ARN, applicable resource policy, and encryption-key permissions if relevant. A's S3 allow does not become a permission of B's role session.

    Also distinguish reading an object from listing a bucket. A role can have the former without the latter. For direct cross-account object access without assumption, A remains the caller and both account-side permissions matter.

    **Trap:** Treating a successful `AssumeRole` as proof that the target resource action is authorized.

    **Test:** The accepted role reads the approved prefix, fails on another prefix, and fails writes; a different caller fails at assumption.

??? example "Worked example"
    Use the synthetic policies from the lesson and an unencrypted-by-customer-key test fixture:

    ```text
    A:BuildPublisher -> AssumeRole(B:ArtifactReader) -> succeeds
    B:ArtifactReader -> GetObject releases/v1.zip    -> succeeds
    B:ArtifactReader -> GetObject private/v1.zip     -> denied: prefix not granted
    B:ArtifactReader -> ListBucket                  -> denied: action not granted
    # Adding s3:* to A does not change these B-session permission decisions.
    ```

    If the intended `releases/` read also fails, inspect restrictions and resource/key policies rather than concluding that the prefix allow is sufficient in every deployment.

## 4. A Vendor Claims External IDs Prevent Compromise

**Study first:** [confused deputy and external IDs](../infrastructure-cloud/iam.md#confused-deputy-and-external-id-limits).

**Scenario:** A vendor serves many customers and accepts a role ARN in each customer's configuration. It proposes a shared external ID and says the ID protects you even if its own role is stolen. Correct the design and identify the residual risk.

??? note "Answer guidance and common trap"
    Bind each customer account to a vendor-assigned unique ID in trusted vendor state. The vendor selects the ID from the authenticated customer association; it must not trust a request to choose another customer's role/ID pair. Your role checks the vendor principal and the expected ID, with no alternate unconditional allow.

    This addresses one customer confusing the vendor into accessing another customer's account. It does not stop an attacker who controls the vendor principal and can send the expected ID. Limit the role's resources/actions, audit its use, and maintain a vendor-access removal procedure.

    **Trap:** Treating the external ID as an independent secret or replacing the principal restriction with it.

    **Test:** Correct ID succeeds; missing/wrong ID fails; another principal with the correct ID still fails.

??? example "Worked example"
    Assume Blue's role trusts only `VendorCollector` and requires `blue-7f42`:

    ```text
    # Vendor selects ID from its authenticated customer's stored configuration.
    Red customer -> request Blue role, vendor sends red-1a20 -> DENY
    Blue customer -> Blue role, vendor sends blue-7f42        -> ALLOW
    Untrusted AWS principal -> Blue role + blue-7f42          -> DENY
    Stolen VendorCollector -> Blue role + blue-7f42           -> may succeed
    ```

    The final case is vendor compromise, which requires containment and narrower authority; changing the ID's secrecy assumptions does not solve it.

## 5. An EKS Pod Has IRSA but Reaches Node Credentials

**Study first:** [IRSA exchange](../infrastructure-cloud/iam.md#eks-irsa-exchange-a-service-account-token) and [IMDSv2 behavior](../infrastructure-cloud/iam.md#ec2-metadata-credentials-and-imdsv2).

**Scenario:** A pod has a narrowly scoped IRSA role. An SSRF bug reaches metadata, and testing shows node-role credentials can be obtained. The team proposes increasing the IMDS token lifetime and says IRSA already isolates the pod. Explain the two credential paths and prioritize fixes.

??? note "Answer guidance and common trap"
    IRSA exchanges a projected service-account token with STS; EC2 metadata supplies the instance/node role through a different path. Restrict metadata access where pods do not require it, require IMDSv2, minimize node-role authority, and fix SSRF. Validate the actual SDK credential source, service-account permissions, and host isolation.

    IMDSv2 requires token acquisition and subsequent token-bearing metadata requests. The token lifetime differs from the issued AWS credential lifetime; response hop limit is an instance metadata option, not a request header. A stronger SSRF primitive or local code execution may complete the exchange. A compromised node also breaks assumptions beyond IRSA trust.

    **Trap:** Calling all AWS workload credentials “metadata credentials,” or treating temporary credentials as harmless.

    **Test:** Wrong service-account subject fails IRSA exchange, intended IRSA access works, and an ordinary pod cannot obtain node credentials. Test privileged/host-network cases separately.

??? example "Worked example"
    This annotated trace describes the credential sources; it is not an attack script:

    ```text
    projected token (payments/report-reader) -> STS -> ReportReader role
    local metadata token exchange           -> IMDS -> NodeRole credentials
    # Restricting ReportReader permissions does not restrict NodeRole.
    # Changing IMDS token TTL does not revoke AWS credentials already returned.
    ```

    A successful metadata test means a second authority path remains. Repair that path and confirm the application still uses its intended workload identity.

## 6. Disable the IdP Account: Is Access Gone?

**Study first:** [federation issuance and outages](../infrastructure-cloud/iam.md#federation-claims-issuance-and-outages), [role chaining](../infrastructure-cloud/iam.md#role-chaining-does-not-preserve-the-original-expiry), and [incident containment](../infrastructure-cloud/multi-account-security.md#emergency-access-and-containment).

**Scenario:** A compromised employee or deployment identity already exchanged an assertion for role A credentials, then assumed role B. You disable the IdP identity. State what can still work and how you would prove containment.

??? note "Answer guidance and common trap"
    Disabling the identity blocks new authentication but may leave valid assertions exchangeable. Block the relevant cloud issuance/trust path as needed, deny outstanding A and B sessions through supported controls, and inspect other reachable roles. Verify denial after propagation using controlled old credentials and look for new successful activity.

    Each issued session has its own expiry. Revoking the source role does not recursively revoke the downstream session. Existing credentials can also continue working during an IdP outage; that is availability behavior, not evidence of successful revocation.

    **Trap:** Reporting the IdP disable time as the time all cloud access ended.

    **Test:** Old A credentials fail, old B credentials fail, and a retained valid assertion cannot obtain a new session through the blocked path.

??? example "Worked example"
    Synthetic times; B's one-hour chained session is allowed:

    ```text
    09:00  Assertion issued, expires 09:05; A session obtained, expires 10:00.
    09:02  IdP identity disabled.
    09:03  Still-valid assertion may obtain another session if STS trust permits.
    09:59  Already-issued A assumes B; B expires 10:59 if not otherwise denied.
    # Neither 09:05 nor 10:00 is a universal end-of-access time.
    ```

    A defensible containment record includes the affected issuance paths, downstream roles, verification times, and remaining evidence gaps.

## 7. CloudTrail Is Enabled but Alerts Have Gone Quiet

**Study first:** [audit collection](../infrastructure-cloud/multi-account-security.md#audit-collection-and-delivery-failure) and [detecting silence](../infrastructure-cloud/multi-account-security.md#detect-silence-as-well-as-malicious-events).

**Scenario:** All account dashboards show a trail, but security events stop appearing after a central policy deployment. Distinguish a quiet workload from broken delivery, disabled collection, and downstream ingestion failure. Explain what log-integrity validation does and does not prove.

??? note "Answer guidance and common trap"
    Compare expected account/region/event-class coverage, successful synthetic probes, trail status, raw archive arrival, and analysis ingestion. Check event selectors, bucket/KMS permissions, regions, membership, and recent control changes. A trail object can exist despite delivery problems.

    Page through a path independent of the failing ingestion system. Restore the reviewed configuration, verify new and delayed events, and document any unreconciled gap. Integrity checks verify delivered log/digest evidence; they do not prove all relevant operations were selected for collection.

    **Trap:** Counting enabled trails or treating lack of alerts as evidence of no attack.

    **Test:** Break delivery permissions in a controlled account and show that an independent check alerts and that restoration is verified end to end.

??? example "Worked example"
    Assume the probe operation is included by selectors and its successful execution is independently recorded:

    ```text
    Case A: probe succeeded; archive has event; search has no event
            -> investigate parsing/transport/indexing after archive delivery
    Case B: probe succeeded; archive missing; trail reports delivery error
            -> investigate collection destination and service permissions
    Case C: probe never ran
            -> repair probe; missing event alone does not locate the failure
    ```

    Record event time and arrival time separately so a backlog does not conceal a continuing current gap.

## 8. Roll Out an SCP Without Creating a Permanent Exception

**Study first:** [onboarding acceptance checks](../infrastructure-cloud/multi-account-security.md#account-onboarding-with-infrastructure-as-code), [staged enforcement and exceptions](../infrastructure-cloud/multi-account-security.md#staged-enforcement-and-expiring-exceptions), and [SCP inheritance](../infrastructure-cloud/iam.md#scp-inheritance-and-scope).

**Scenario:** A new organizational deny breaks one release pipeline. The team asks to attach an allow to that account and promises to fix the pipeline later. Plan the immediate recovery and a safe resumed rollout.

??? note "Answer guidance and common trap"
    An account allow cannot override an inherited explicit deny. Stop expansion, use the tested organization-policy recovery path, and restore the last reviewed working scope if the acceptance criteria fail. Investigate whether the deny is wrong or the workflow needs migration before widening it.

    If an exception is justified, scope the actual enforcement change, name an owner, set an expiry, and automate removal with verification. Test a representative nonproduction cohort, then production cohorts, including release, recovery, and audit behavior. Historical logs and policy linting inform the tests but cannot replace them.

    **Trap:** Calling a tracking ticket a time-bounded permission, or putting a whole account in a weaker OU without enumerating lost controls.

    **Test:** The exceptional legacy workflow works during approval; another principal remains denied; after expiry the exception is removed and effective denial is confirmed.

??? example "Worked example"
    Illustrative controller behavior:

    ```text
    11:55  Notify owner that EX-042 ends at 12:00; migration readiness checked.
    12:00  Controller removes the scoped policy exception.
    12:02  Negative test confirms the old request is denied.
    # If removal fails, page the owner and mark the exception still effective.
    # A changed ticket status alone must never produce "exception closed".
    ```

    A rollout rollback restores a previously reviewed state; it should not disable all SCPs or silently grant administrator access.

## 9. Recover While Workforce Federation Is Unavailable

**Study first:** [emergency access and containment](../infrastructure-cloud/multi-account-security.md#emergency-access-and-containment) and [management-account limits](../infrastructure-cloud/multi-account-security.md#account-and-ou-boundaries).

**Scenario:** The corporate IdP is unavailable and a bad organization policy prevents production repairs. Your documented emergency procedure starts by signing in to the same IdP to retrieve a recovery credential. Redesign and test it under explicit constraints.

??? note "Answer guidance and common trap"
    Identify the shared dependency and pre-provision a recovery path that survives the specified outage. Protect its authenticators and recovery process, limit custodians and scope, monitor use, and practice the real repair. An inherited SCP may still deny a member-account recovery role; organization-policy rollback requires a tested authorized path with the necessary organization authority.

    State residual dependencies such as reachable AWS endpoints and IAM/STS availability. Explain the standing-access cost of independent credentials and how their custody, testing, and post-use review reduce risk. Distinguish an operational two-person retrieval process from technical enforcement of two-person approval.

    **Trap:** Assuming an administrator role overrides an SCP, or describing emergency access without testing credential retrieval.

    **Test:** With normal federation unavailable, named custodians obtain scoped access, perform the rehearsed repair, and produce reviewable evidence without the failed dependency.

??? example "Worked example"
    A tabletop dependency trace can reveal the flaw before a controlled exercise:

    ```text
    Broken: IdP outage -> recovery portal requires IdP -> credential unavailable
    Revised: independent custody -> scoped recovery identity -> reviewed rollback
             -> verify production repair + audit delivery -> review emergency use
    # Independent custody still needs available people, authenticators, and endpoints.
    ```

    If the scope is a provider-wide control-plane outage, this revised identity path alone cannot restore unavailable services; state that limit rather than promising universal recovery.
