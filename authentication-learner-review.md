# Authentication learner review

Reviewed 2026-10-14. Read AGENTS.md and all 12 current Markdown pages in full (1,229 lines): index, sessions, jwt, oauth2-oidc, saml, mtls, webauthn, directory-services, comparison, design-patterns, case-studies, interview-questions. Read git status before review; no repository files edited. Existing changes and untracked files preserved.

## Publication judgment

Ready to publish as an intermediate security-engineering reference and interview practice module. No P0/P1 issue or demonstrated technical blocker found. It is not yet fully self-contained for a foundational learner: some central mechanisms are named and prescribed before their operation is explained. The focused additions below would improve that considerably without requiring another broad rewrite. The most valuable additions are a short PKCE explanation and a concrete step-up approval example.

The material is substantially stronger at explaining WHY than a conventional checklist: accepted credential versus issuing credential, audience versus signature, subject versus actor, and login versus session lifetime recur consistently in genuinely different scenarios. Caveats generally qualify real deployment differences rather than obscure recommendations. Claims about sender constraints, PKCE, SAML profiles, certificate checks, recovery, and immediate revocation are appropriately bounded.

## Prioritized actionable findings

### P2 — Explain the PKCE verifier/challenge relationship, not only its required use

Location: docs/authentication/oauth2-oidc.md:27–36; reinforced by interview-questions.md:50–56.

The flow creates a verifier, derives an S256 challenge, and says the provider checks the PKCE proof, but never explains the actual comparison or why a visible challenge does not let a code thief redeem the code. A learner can repeat “binds the transaction” without answering “why does this work?”

Recommended fix: add 3–4 sentences or a tiny sequence: the client retains an unpredictable verifier; it sends BASE64URL(SHA256(verifier)) at authorization; the server binds that challenge to the code; redemption supplies the verifier and the server recomputes and compares. An intercepted code/challenge alone is insufficient to recover a suitably random verifier. Keep the existing boundary that PKCE is not confidential-client authentication. This needs no production implementation snippet.

Source inspected for this concern: https://www.rfc-editor.org/rfc/rfc7636.html (introduction and protocol structure).

### P2 — Give one concrete meaning of operation-bound approval

Location: docs/authentication/design-patterns.md:204–209, with related interview-questions.md:153 and :256–258.

The sensitive-operation pattern correctly says to bind approval to the operation and revalidate its parameters, but it never shows what such binding looks like. “Recent MFA” and “approval of this payout” are distinguished negatively; a learner is still left to invent the positive design.

Recommended fix: add a conceptual example where the server records a pending approval containing actor/account, action, target, destination/amount, expiry, and a single-use approval identifier; the step-up transaction references that pending operation; execution checks that exact operation and current policy and consumes approval atomically with execution as appropriate. Explicitly distinguish this server-side binding from a trusted display or cryptographic proof that the human saw and approved the exact parameters. Do not promise malware resistance from a browser confirmation screen. This is an explanation gap, not a finding that the current guidance is wrong.

### P2 — Define the terms used to make architecture choices

Location: docs/authentication/comparison.md:30–40 and :55–61; design-patterns.md:86–91; directory-services.md:14–15.

The comparison uses BFF, STS, opaque, delegation, introspection and sender constraints before all are explained. BFF is expanded only in a later design-pattern heading; STS is never expanded in these 12 pages. Directory material uses DN without expanding distinguished name. Existing prerequisites cover crypto and HTTP but not these architecture terms. This costs a beginner substantial context switching.

Recommended fix: expand BFF (Backend for Frontend) and STS (Security Token Service) at first use and explain their role in one clause; define opaque token as a credential whose contents the client/API cannot interpret under its contract and introspection as querying the authorization server about token state/metadata. Add local links from the comparison to the detailed pattern/protocol section. Expand DN locally. Prefer local definitions over a large glossary of every acronym.

### P3 — Put the learning route before the advanced entry points

Location: docs/authentication/index.md:5–10 and :52.

The page begins by sending the reader to the advanced comparison, patterns, cases and tricky questions. Only its final sentence explains the foundations-first order. This makes the first useful instruction easiest to miss, and comparison/design-patterns link to each other without an explicit core prerequisite list.

Recommended fix: move the study-order sentence above the Architecture and Interview Practice section and add a short progression: core pages → comparison → selected patterns → cases/questions. Label the architecture area as application/practice. Avoid adding more navigation tables.

### P3 — Explain why Kerberoasting permits offline guessing

Location: docs/authentication/directory-services.md:37–43 and :52–53.

The flow is sound, but the attack row compresses the key inference into “offline password guessing against service-ticket material.” A student may not understand why a legitimate requester receives material useful for guessing someone else's password.

Recommended fix: add two sentences: a requester can obtain a ticket for a service, and part of that ticket is encrypted under that service account's long-term key; an attacker can test password-derived candidate keys locally against the captured encrypted material, without repeated service logins. State that this depends on the account/key/encryption characteristics, preserving the existing managed-account and strong-random-credential advice. No attack commands needed.

## Per-page coverage verdict

| Page | Verdict | Learner assessment |
|---|---|---|
| index.md | Publishable; improve ordering | Excellent distinction table and combined login/session example; study route should come first. |
| sessions.md | Ready | Clear HTTP example, fixation causal example, and cookie limitations. CSRF mechanisms remain compact reference terms; link to a specific web-security explanation if available. |
| jwt.md | Ready | Good format/protocol separation and audience counterexample. Could name JWS/JWE in full, but no material blocker. |
| oauth2-oidc.md | Publishable; highest-value clarification | Roles and flow are strong; add PKCE mechanics. The qualified state/nonce guidance is preferable to a misleading universal rule. |
| saml.md | Ready | Flow, exact signed content, recipient, request correlation and local-session boundaries are clear. NameID can remain an advanced linked term. |
| mtls.md | Ready | Useful conceptual handshake and same-CA/different-permission example; avoids confusing certificates with key possession. |
| webauthn.md | Ready | Strong explanation of RP/origin, UP/UV, backup and recovery. It correctly avoids claiming hardware-only keys or universal session protection. |
| directory-services.md | Publishable; clarify one attack mechanism | Good LDAP flow and Kerberos sequence; expand DN and the offline guessing explanation. |
| comparison.md | Ready for intermediate readers | Useful decision table, concrete revocation-cache exercise and independent grant/proof/use dimensions. Define first-use architecture terms. |
| design-patterns.md | Ready; clarify operation binding | Seven scenarios justify different designs rather than merely prescribe protocols. Step-up section needs one positive concrete example. |
| case-studies.md | Ready | Reported facts versus interpretation is unusually clear; the three cases show distinct credential boundaries and are worth retaining. |
| interview-questions.md | Ready | All 15 scenarios are substantive. Hidden answers, traps, rejection tests and altered assumptions support learning better than trivia. |

## What to keep and what to cut

Keep all seven patterns and all 15 questions. There is purposeful repetition, but the layers have distinct jobs: comparison selects, patterns trace, cases test real assumptions, questions test recall and transfer. In particular, keep questions 9, 10, 14 and 15: object/property checks, durable approval and cache correctness prevent this becoming a narrowly cryptographic authentication guide.

Keep caveats separating client authentication from sender-constrained token use, password rotation from token revocation, and phishing resistance from session theft. Removing those for brevity would remove core teaching value.

Optional cuts: consolidate the repeated disclaimer about original/non-employer interview questions in comparison.md:113, design-patterns.md:3, interview-questions.md:3 and :301. A brief note on the interview page plus an unobtrusive context link is sufficient. Keep the incident facts/analysis disclaimer, which serves a different purpose. The repeated incident-report inline link and resource-block link on case-studies.md is also optional duplication, not a defect.

Do not expand every protocol page into a comprehensive implementation guide. The module benefits more from a few causal paragraphs and local links than from additional attack catalogs or configuration recipes.

## Scope and uncertainty

This was a full current-text learner review, not a second independent exhaustive standards or external-link audit. I inspected RFC 7636 for PKCE and RFC 10017 to verify that the cited August 2026 browser-app BCP is real/current; I did not individually reopen every incident source or verify every changed URL. Source validation already performed by the parent remains the evidence for those claims. No source-related defect was established in this review.

I did not rerun the build, inspect rendered/mobile tables, or independently check every internal anchor. The parent reports the current build was verified. Wide comparison tables may deserve a narrow-screen visual check, but I have no observed rendering defect to report. No repository edits were made; only this review report was written.

## Follow-up and Final Verification

The parent applied the recommended PKCE mechanism explanation, operation-bound approval example, local terminology definitions, study-route ordering, and Kerberoasting explanation, plus expanded JWS/JWE. Original review line references above refer to the text before those additions.

Astra then reread the targeted changes and signed off for the intended intermediate reader: the explanations resolve the substantive learning gaps without unnecessary expansion; no necessary corrections or publication blockers were found. This is a content-review judgment, not a guarantee for every reader or an exhaustive implementation/security audit.

Parent verification after the changes: MkDocs build passes; all 12 Authentication pages render with one H1; 46 internal links/fragments involving the section resolve; all 15 collapsible interview answers render; `git diff --check` passes. Three pre-existing missing anchors in out-of-scope `docs/notes.md` remain.
