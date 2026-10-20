# Interoperable Payment Authorization Evidence Profile

**Status:** Discussion proposal; not an adopted specification

**Prepared:** 20 October 2026

**Version:** 0.1

**Prepared by:** Dalibor Gogic (DLBR)

**Purpose:** Invite a bank, an SPC implementation team and an EUDI wallet team to conduct a small interoperability experiment.

## Abstract

We propose common payment authorization acceptance rules over version-pinned native proofs. An SPC adapter and a wallet adapter independently verify their respective artifacts, compare the verified evidence against an immutable bank-held transaction context, and report distinct verification, authentication-policy, authorization and execution outcomes.

The experiment preserves native signed artifacts. It introduces no replacement credential, cryptographic proof format or translation of signed content. It does not itself establish regulatory compliance, payment-scheme acceptance or settlement finality.

## 1. Problem and contribution

A valid cryptographic proof does not by itself show that a bank's authentication policy was satisfied, that the bank accepted an authorization, or that a payment executed. In three-party flows, the party collecting the proof may also differ from the bank making the decision.

Existing mechanisms address parts of this problem. The proposed contribution is a small acceptance profile and shared negative-test matrix across two implementations, with explicit evidence and responsibility boundaries. Before defining a new interface, participants should identify which existing scheme interfaces already meet these requirements.

The bank-issued challenge, strict beneficiary mapping, atomic authorization consumption and cross-mechanism execution-idempotency rules below are additional requirements of this experiment. They are not presented as requirements inherited uniformly from SPC, RFC008, TS12 or PaSO.

## 2. Initial scope and roles

The pilot covers one immediate payment, one currency, one beneficiary and a bank-issued unpredictable challenge. Recurring payments, mandates, offline payments, refunds and cross-border scheme governance are outside the first experiment.

- **Bank/authorizing party:** owns account association, transaction context, authentication policy, fraud decisioning, authorization acceptance and payment-instruction controls.
- **Merchant/intermediary:** supplies authenticated transaction context through an agreed channel, invokes authentication where applicable, and forwards original artifacts.
- **Wallet/browser:** performs the selected native confirmation ceremony and produces its native proof.
- **Verification service:** implements adapters, matching and evidence capture. A delegated verification result does not replace the bank's acceptance decision.

These are functional roles; a participant may perform several. Contracts and the selected payment scheme must allocate operational and legal responsibility separately.

## 3. Authoritative transaction context

Before customer authentication, the bank establishes an immutable context containing:

- Unique authorization ID, challenge, creation time and expiry.
- Exact amount and currency, using an agreed decimal/minor-unit representation without binary floating-point comparison.
- Beneficiary identifier, customer-facing payee identity and the authenticated mapping between them.
- Expected merchant/intermediary identity, permitted origins and native verifier/audience identity.
- Account or funding-source association, selected native profile and authentication-policy identifier/version.

Material changes require a new context and challenge. Context access is authenticated, authorized and tenant-scoped; an authorization ID is a correlation reference, not a bearer capability.

The bank binds each native request to this context. The selected profile must specify how this works when an intermediary constructs the presentation request, including challenge delivery, expected audience and authenticated request provenance.

## 4. Native-proof verification mapping

| Obligation | SPC adapter | Wallet adapter |
| --- | --- | --- |
| Customer/key association | Registered WebAuthn credential and applicable trusted device-binding evidence | Trusted issuer credential, holder binding and account association |
| Challenge | WebAuthn challenge in signed `clientDataJSON` | Presentation nonce in the native profile's signed structure, e.g. key-binding JWT |
| Transaction binding | Verified signed payment amount, currency and payee fields | Verify transaction binding using the selected native profile's exact hash inputs, encoding and signed claims; then validate the confirmed transaction against the bank-held context |
| Requesting-party binding | Expected RP binding, caller origin and top-level origin | Expected audience/client identity and request authentication required by the selected profile |
| Authentication policy | Available user-verification, authenticator and trusted device evidence | Profile-defined factor assertions and supporting assurance evidence |
| Freshness/single use | Bank challenge expiry and atomic context consumption | Same controls, plus native identifiers and timestamps where applicable |

The adapters must implement the full native verification rules, not only this table. They must not infer equivalent assurance across formats. Missing evidence yields `insufficient_evidence` when policy requires it.

The nonce need not be inside transaction_data: the native signed structure must bind the challenge and the confirmed transaction details together. For EWC RFC008, recompute hashes over the exact base64url-encoded transaction_data strings without parsing and reserializing them first. For other profiles, use their specified hash inputs, encodings and signed claims; do not assume RFC008 processing applies.

SPC browser-bound keys supplement the passkey assertion. Newly encountered device keys require the bank's enrollment or step-up policy; their presence alone is not trusted device possession.

## 5. Merchant and beneficiary binding

The pilot requires an authenticated, unambiguous mapping from the signed customer-facing payee fields and expected merchant/origin to the beneficiary actually credited. A display name alone is insufficient.

The proof need not expose the beneficiary's account identifier if the signed fields can be unambiguously resolved through the bank-held mapping. The mapping must be bound to the authorization ID, directly or through an integrity-protected reference. Retain the exact mapping used for the decision; later updates must not change an existing authorization.

Intermediary assertions must have authenticated provenance and transaction integrity. A hash alone does not authenticate their source. Missing, ambiguous or conflicting mappings cause rejection in this pilot, rather than a soft risk warning.

Forwarding does not change the native proof's audience. The bank verifies the original audience/caller binding and separately authenticates the intermediary's authority to submit evidence for this authorization context. It must not require itself to be the original audience where the selected native flow targets the merchant or another verifier.

The execution system must enforce the same beneficiary and amount accepted at authorization. An authorized intermediary must not be able to substitute a different instruction after verification.

## 6. Authentication policy and outcomes

Record the policy identifier/version, evidence observed, assurance/trust basis and evaluation result: `satisfied`, `not_satisfied` or `insufficient_evidence`. The profile does not prescribe a universal factor policy or treat a wallet assertion or browser flag as independent certification.

Report four separate dimensions:

| Dimension | Example outcomes | Responsible decision |
| --- | --- | --- |
| Proof verification | `verified`, `invalid`, `unsupported` | Native adapter |
| Authentication policy | `satisfied`, `not_satisfied`, `insufficient_evidence`, `not_evaluated` | Bank policy or authorized evaluator |
| Authorization | `accepted`, `rejected`, `pending`, `not_evaluated` | Bank |
| Execution | `not_started`, `pending`, `executed`, `failed`, `unknown` | Payment system |

These are proposed experiment labels, not an existing registry. Each result includes the authorization ID, decision time, profile identifier/version and relevant reason codes. Responses travel over authenticated channels with transaction integrity. `executed` must identify its reporting source and must not imply settlement finality unless the selected scheme explicitly establishes it.

## 7. Freshness, concurrency and retries

Check the bank-issued challenge, expiry, expected audience/origin and native freshness requirements. Participants agree clock-skew bounds explicitly.

Acceptance atomically consumes the authorization context and records the accepted proof identity and decision. An identical authenticated retry of an already accepted request returns the recorded decision even after the original context expires; it does not create a new acceptance or execution. A different proof against a consumed context returns replay_detected. Expiry prevents new acceptance of an unconsumed context and returns expired. Define proof identity from preserved artifacts and profile rules, not an unstable JSON re-encoding.

Single-use authorization does not guarantee single payment execution. Execution requires a durable idempotency key bound to the accepted payment instruction, plus recovery/reconciliation after timeouts or crashes. SPC and wallet fallback attempts for the same payment intent must share that intent's execution idempotency key, even if their native requests use separate authorization contexts and challenges. Competing attempts must not authorize conflicting instructions or create duplicate execution. An uncertain execution result is unknown or pending, not an instruction to initiate another payment.

## 8. Historical evidence and privacy

Under an agreed retention and access policy, preserve:

- Exact original requests, native proofs and encoded transaction data.
- Bank context, beneficiary mapping and authenticated intermediary artifacts.
- Acceptance-profile identifier/version, native specification revisions, adapter identifier/version and authentication-policy identifier/version.
- Public verification keys, certificates, metadata and contemporaneous trust/revocation/status evidence with provenance. Private signing keys, decryption keys and authentication secrets are excluded from the evidence archive.
- Receipt times, verification findings, authorization/execution decisions and integrity values.

Historical verification uses the artifacts and rules applicable at the time, rather than silently substituting current metadata or trust lists. Define timestamp assurance and archival renewal needs; hashes alone establish neither trustworthy time nor historical validity. Reconstructing a verification result does not by itself prove informed consent or determine liability.

Keep credentials and sensitive evidence within authorized boundaries. A merchant receives only the status and references necessary for its role. Independent bank verification must remain possible; privacy requirements may require direct delivery or issuer encryption instead of exposing the entire evidence package to an intermediary. For encrypted evidence, retain the original ciphertext and define controlled access and the decryption-key lifecycle separately from the evidence archive. Do not publish real customer artifacts in test results.

## 9. Relation to existing specifications

- **SPC:** browser-mediated payment confirmation using WebAuthn. Apply native assertion, origin, transaction and applicable browser-bound-key checks. This experiment adds bank-context matching and explicit acceptance outcomes.
- **EWC RFC007/RFC008:** Payment Wallet Attestation and payment-data confirmation using OpenID4VP and transaction-data hashes in a key-binding JWT. RFC008 already specifies matching payment details against intermediary-provided context in three-party flows.
- **TS12:** EUDI-wallet SCA processing, transaction types and additional proof claims. Select its exact version and associated deployment/rulebook requirements.
- **PaSO:** modular transaction evidence and authorization handoff. Evaluate reuse of its verification interface before introducing another one; pin its exact revision, rulebook and risk profile.

These are distinct mechanisms, not interchangeable claims of conformance. The first experiment selects SPC plus one named wallet profile. Additional wallet profiles follow only after the first pair passes.

## 10. Pilot success criteria

- [ ] One bank, browser/SPC team and wallet team agree roles, test access and profile revisions.
- [ ] A legitimate payment authorization is accepted through each adapter.
- [ ] Amount, currency, payee, beneficiary and requesting-origin substitutions are rejected.
- [ ] Mapping replacement after customer confirmation is rejected.
- [ ] Expired/wrong challenges and invalid audience or credential/account bindings are rejected.
- [ ] Newly registered or replaced device keys receive the required enrollment/step-up treatment.
- [ ] Missing required authentication evidence yields `insufficient_evidence`.
- [ ] Concurrent submissions and identical retries produce at most one accepted authorization per context; an identical authenticated retry after expiry returns the recorded acceptance.
- [ ] Execution retries/crash recovery and competing SPC/wallet fallback attempts for one payment intent cannot create duplicate or conflicting payment instructions.
- [ ] Cancellation and unsupported mechanisms cannot be reported as accepted authorization.
- [ ] Archived evidence can be reverified after controlled key, metadata and policy changes.
- [ ] Authentication success, authorization acceptance and execution remain distinguishable.

Use synthetic transactions first. Any subsequent funds-moving trial requires the participating bank's separately agreed scope and controls. No participant or passing result is claimed by this draft.

## 11. Questions and requested next step

1. Which existing scheme or PaSO interface should carry context and proof artifacts?
2. What minimum signed fields and authenticated mappings establish the credited beneficiary for each mechanism?
3. Which evidence is sufficient for the bank's factor and newly registered device policies?
4. What trust/status and time evidence is needed for historical verification?
5. Can one bank, one SPC implementation and one wallet team agree to run the same negative-test matrix?

The requested outcome is agreement on a small experiment and its participants, not immediate adoption of another payment standard.

## References

References checked 20 October 2026. Fixed revisions support review; pilot participants must separately agree supported versions.

- [SPC, Candidate Recommendation Draft, 13 October 2026](https://www.w3.org/TR/2026/CRD-secure-payment-confirmation-20261013/)
- [EWC RFC007 at reviewed revision](https://github.com/EWC-consortium/eudi-wallet-rfcs/blob/26028b7adc7345a660f3883d1ac70dbc11f031c9/payment-rfcs/ewc-rfc007-payment-wallet-attestation.md)
- [EWC RFC008 at reviewed revision](https://github.com/EWC-consortium/eudi-wallet-rfcs/blob/26028b7adc7345a660f3883d1ac70dbc11f031c9/payment-rfcs/ewc-rfc008-payment-data-confirmation.md)
- [TS12 at reviewed revision](https://github.com/eu-digital-identity-wallet/eudi-doc-standards-and-technical-specifications/blob/ee91a294c833af5188726fd8c302c641212192aa/docs/technical-specifications/ts12-electronic-payments-SCA-implementation-with-wallet.md)
- [PaSO at reviewed revision](https://github.com/APTITUDE-Consortium/payments-and-sca-for-openid/tree/cace6df294a5b11fff7be62e7d6bf9fa3e646589)
- [ARF Topic AA refinement discussion #660](https://github.com/eu-digital-identity-wallet/eudi-doc-architecture-and-reference-framework/discussions/660)
- [EUDI-4-Payments proposal repository](https://github.com/w3c-cg/eudi-4-payments)
