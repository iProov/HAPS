# HAPS Conformance v0.4 (Draft)

**Status:** Experimental draft; not production-ready
**Applies to:** HAPS v0.4.x (including v0.4.0 bundle)

Conformance with this document requires satisfying every applicable requirement, including the supplied
vectors. Passing the supplied tests does not establish full protocol conformance or that a person saw
and approved the correct action. The partial reference implementation does not implement a complete
PP or RP. See [limitations](../../docs/limitations.md). Conformance is **not** evidence
of any PoHP level, and not evidence about the quality of a presence signal — see `haps-v0.4.0.md` §16,
which separates the two kinds of certification and forbids presenting one as the other.

This document defines conformance classes and testable requirements (MUST/SHOULD) for:
- Presence Providers (PP)
- Relying Parties (RP)
- Identity Binding adapters (initial: `entra_oidc`)

## Conformance classes

- **HAPS-PP-0.4**: Presence Provider core (PoHP + intent binding)
- **HAPS-RP-0.4**: Relying Party verifier core
- **HAPS-PP-ID-0.4**: Presence Provider identity binding support
- **HAPS-RP-ID-0.4**: Relying Party identity binding verification
- **HAPS-ADAPTER-entra_oidc-0.1-PP**: PP implementing Entra OIDC adapter
- **HAPS-ADAPTER-entra_oidc-0.1-RP**: RP verifying Entra OIDC adapter
- **HAPS-PROFILE-entra_claims_challenge-0.1-PP**: PP supporting Entra claims challenge transport
- **HAPS-PROFILE-entra_claims_challenge-0.1-RP**: RP / bridge consuming Entra claims challenge transport
- **HAPS-PROFILE-digital_credentials_portrait_liveness-0.1-PP**: PP supporting Digital Credentials / portrait/liveness identity binding
- **HAPS-PROFILE-digital_credentials_portrait_liveness-0.1-RP**: RP verifying Digital Credentials / portrait/liveness identity binding
- **HAPS-PP-FACTORS-0.1**: Presence Provider factor declaration
- **HAPS-RP-FACTORS-0.1**: Relying Party factor policy enforcement

## Smallest conformant implementation

A relying party claiming **HAPS-RP-0.4** must satisfy the RP core requirements below and VEC-01–03.
The core covers malformed-input and numeric-syntax rejection, signature and issuer trust, audience,
temporal claims, both hashes, minimum assurance and certification, conditional single use and envelope
constraints. Replay-state retention (RP-CORE-10) and conditional Challenge Mode requirements
(RP-CORE-09) also apply as stated below. Passing the vectors is only one part of a conformance claim.

Additional conformance classes and profiles cover identity binding (`HAPS-*-ID-0.4`), specific IdP
adapters, factor policy (`HAPS-RP-FACTORS-0.1`) and MCP transport. Challenge Mode applies when used or
required by RP policy. Core verification can run locally when keys, certifications, policy, a clock and replay state are
provisioned. This does not eliminate deployment needs such as key/certification updates or online
provider and identity-adapter interactions.

## PP core requirements (HAPS-PP-0.4)

- **PP-CORE-01 (MUST)** Validate AI-INTENT schema and compute `intent_hash` from AI-INTENT canonicalized with RFC 8785 (JCS).
- **PP-CORE-02 (MUST)** Derive Signing View and compute `presentation_hash`.
- **PP-CORE-02a (MUST)** Reject an AI-INTENT containing a JSON number with a fractional part or an exponent (§6.1) rather than canonicalize it.
- **PP-CORE-03 (MUST)** Display the Signing View (or equivalent deterministic rendering) to the human in the PP UI.
- **PP-CORE-04 (MUST)** Issue HAPS-CC containing `intent_hash`, `presentation_hash`, `aud`, `jti`, `iat`, `exp`, `assurance`.
- **PP-CORE-05 (MUST)** Do not include biometric samples in HAPS-CC.
- **PP-CORE-06 (MUST)** Include provider certification evidence (embedded or referenced) sufficient for RP to check PoHP level validity.
- **PP-CORE-07 (MUST)** If credential issuance is based on HAPS-CHAL input, include `claims.challengeId` and set it to the exact challenge used for the consent event.

## RP core requirements (HAPS-RP-0.4)

- **RP-CORE-01 (MUST)** Verify HAPS-CC signature and issuer trust.
- **RP-CORE-02 (MUST)** Verify `aud` matches RP context.
- **RP-CORE-03 (MUST)** Enforce expiry/max-age with bounded clock skew, including `exp`, `iat`, and `nbf` (if present).
- **RP-CORE-04 (MUST)** Recompute and match `intent_hash` using RFC 8785 (JCS) canonicalization of AI-INTENT.
- **RP-CORE-04a (MUST)** Reject an AI-INTENT containing a JSON number with a fractional part or an exponent before canonicalization (§6.1), with the other malformed-input checks.
- **RP-CORE-05 (MUST)** Recompute and match `presentation_hash` using RFC 8785 (JCS) canonicalization of Signing View.
- **RP-CORE-06 (MUST)** Enforce minimum PoHP level and validate provider certification.
- **RP-CORE-07 (MUST)** Enforce one-time use when required (`jti` replay prevention) with atomic consume semantics.
- **RP-CORE-08 (MUST)** Enforce envelope constraints when present.
- **RP-CORE-09 (MUST when challenge mode is used/policy-required)** Verify `claims.challengeId` exists, matches an outstanding challenge, is unexpired, and has not been consumed.
- **RP-CORE-10 (SHOULD)** Retain replay-cache state for consumed `jti`/`challengeId` entries until at least `exp + skew`.

## PP identity binding requirements (HAPS-PP-ID-0.4)

- **PP-IDB-01 (MUST)** Support `requirements.identity` input.
- **PP-IDB-02 (MUST)** If `mode="required"` and identity cannot be satisfied, PP MUST NOT issue HAPS-CC.
- **PP-IDB-03 (MUST)** If identity performed, include `claims.identityBinding` with `mode`, `scheme`, `subject`.
- **PP-IDB-04 (MUST)** Bind identity evidence to the same approval session as PoHP (single consent event).
- **PP-IDB-05 (MUST)** If `requireEmbeddedEvidence=true`, embed evidence and mark it embedded.
- **PP-IDB-06 (SHOULD)** Minimize PII; prefer stable subject identifiers over display names/emails.

## RP identity binding requirements (HAPS-RP-ID-0.4)

- **RP-IDB-01 (MUST)** If identity required, reject HAPS-CC missing identityBinding.
- **RP-IDB-02 (MUST)** Enforce scheme allow-list and required identity mode.
- **RP-IDB-03 (MUST)** Verify identityBinding subject matches RP account/session identity.
- **RP-IDB-04 (MUST when required)** If embedded evidence required, verify embedded evidence per scheme rules.
- **RP-IDB-05 (SHOULD)** Enforce identity freshness when `auth_time` is present.
- **RP-IDB-06 (SHOULD when requiring HAPS-PoHP-3 or above)** Require embedded evidence and re-verify it, rather than relying on the provider's assertion. A provider that can forge presence otherwise decides outcomes the RP intended to decide (`haps-v0.4.0.md` §14).

## Entra adapter requirements (PP) — HAPS-ADAPTER-entra_oidc-0.1-PP

- **ENTRA-PP-01 (MUST)** Use OIDC Authorization Code + PKCE (session-bound).
- **ENTRA-PP-02 (MUST)** Validate ID token signature and core claims: `iss`, `aud`, `exp`, `iat/nbf`, with bounded clock skew.
- **ENTRA-PP-03 (MUST)** Validate nonce binding and output `nonceHash`.
- **ENTRA-PP-04 (MUST)** Normalize subject as `tid+oid` and output `subject.type="entra_oid_tid"`.
- **ENTRA-PP-05 (MUST when provided)** Enforce `allowedTenants`.
- **ENTRA-PP-06 (MUST when required)** Enforce MFA/auth context requirements when requested by RP policy.
- **ENTRA-PP-07 (MUST)** Keep OIDC nonce/session freshness state outside AI-INTENT payload semantics.

## Entra adapter requirements (RP) — HAPS-ADAPTER-entra_oidc-0.1-RP

- **ENTRA-RP-01 (MUST)** Verify `tid+oid` match the RP’s authenticated Entra user (or mapping).
- **ENTRA-RP-02 (MUST when required)** If embedded evidence is present and policy requires, verify signature and claims with bounded clock skew.
- **ENTRA-RP-03 (MUST when nonce evidence is present)** Verify nonce binding (`nonce` and/or `nonceHash`) for embedded Entra evidence.


## Entra claims challenge profile (PP) — HAPS-PROFILE-entra_claims_challenge-0.1-PP

- **ENTRA-CC-PP-01 (MUST)** Accept an explicit claims request via `requirements.identity.policy.entraClaimsChallenge` when present. Legacy MCP `schemeParams` input is usable only if explicitly supported; it is not allowed by the HAPS-CHAL v0.4 schema.
- **ENTRA-CC-PP-02 (SHOULD)** Derive a minimal Entra claims request from `requireMfa` and/or `requiredAuthContexts` when explicit challenge data is absent.
- **ENTRA-CC-PP-03 (MUST)** Send the effective Entra `claims` parameter on the Authorization Code + PKCE authorization request.
- **ENTRA-CC-PP-04 (MUST)** Keep `state`, `nonce`, PKCE, and the HAPS session bound together for the entire enterprise step-up flow.
- **ENTRA-CC-PP-05 (MUST)** Continue to issue a normal HAPS Consent Credential after the Entra step-up completes; the refreshed token alone is not sufficient.

## Entra claims challenge profile (RP/bridge) — HAPS-PROFILE-entra_claims_challenge-0.1-RP

- **ENTRA-CC-RP-01 (MAY)** Request enterprise step-up by providing explicit Entra claims-challenge data in policy.
- **ENTRA-CC-RP-02 (SHOULD)** Require embedded evidence when direct verification of the refreshed enterprise identity token is necessary.
- **ENTRA-CC-RP-03 (MUST)** Continue to verify the resulting HAPS Consent Credential in full, including identity binding, rather than relying on the refreshed token alone.

## Digital Credentials + portrait/liveness profile — HAPS-PROFILE-digital_credentials_portrait_liveness-0.1

- **DCA-PP-01 (MUST)** Bind the Digital Credential presentation and any liveness/portrait-match result to the same HAPS challenge or approval session.
- **DCA-PP-02 (MUST)** Verify or rely on explicitly trusted verification of the Digital Credential presentation before including the evidence in `identityBinding`.
- **DCA-PP-03 (MUST)** When liveness is required, bind the liveness result to the same `challenge_id`, `journey_id`, or `approval_session_id`.
- **DCA-PP-04 (MUST)** Do not embed raw portraits, raw biometric samples, face embeddings, or biometric templates in HAPS-CC.
- **DCA-RP-01 (MUST)** Verify HAPS-CC core claims and enforce the expected `identityBinding.scheme`.
- **DCA-RP-02 (MUST)** Verify evidence reference/hash consistency and freshness when this profile is policy-required.
- **DCA-RP-03 (MUST)** Distinguish plain Digital Credentials mode from liveness-backed modes in policy and audit logs.

## Factor declaration (PP) — HAPS-PP-FACTORS-0.1

Applies to `factors-v0.1.md` and `pohp-assurance-levels-v0.4.md`.

- **FACTOR-PP-01 (MUST)** Set `assurance.method` to the factor class actually used, taken from the registry in `factors-v0.1.md` or from a documented extension value.
- **FACTOR-PP-02 (MUST)** Not assert a PoHP level whose required properties (`pohp-assurance-levels-v0.4.md` §1) the factor and its implementation do not satisfy.
- **FACTOR-PP-03 (MUST)** Not assert `HAPS-PoHP-2` or higher on the basis of an agent-accessible factor. Property **H** is required from PoHP-2 upward.
- **FACTOR-PP-04 (MUST)** Where several factors contribute, name the factor that established human actuation (**H**); if none did, the result is PoHP-1.
- **FACTOR-PP-05 (MUST when asserting PoHP-4)** Ensure the provider's HAPS-PCC lists both the asserted level in `certifiedLevels` and the asserted factor in `certifiedFactors`.

## Factor policy (RP) — HAPS-RP-FACTORS-0.1

- **FACTOR-RP-01 (MUST)** Support an accepted-factor-class policy and reject a HAPS-CC whose `assurance.method` falls outside it.
- **FACTOR-RP-02 (MUST)** Treat an unrecognised `assurance.method` as agent-accessible and unclassified unless policy states otherwise.
- **FACTOR-RP-03 (MUST when policy requires agent resistance)** Reject agent-accessible factors regardless of the level asserted. A level assertion alone is not evidence of property **H**.
- **FACTOR-RP-04 (MUST when requiring PoHP-4)** Verify the provider's HAPS-PCC covers both the asserted level and the asserted factor; treat an unverifiable certification as unattested. A lower classification MUST NOT bypass the required minimum level or RP-CORE-06 certification validation; reject if those requirements are unsatisfied.
- **FACTOR-RP-05 (SHOULD)** Record `assurance.method` alongside the PoHP level in audit logs, so a later review can tell what kind of proof was accepted.

## Test vectors

- **VEC-01 (MUST)** Reproduce the three input/expected-hash pairs listed in [the vector README](../../test_vectors/v0.4/README.md#hashing-vectors) using RFC 8785 (JCS) canonicalization. Other digest strings in illustrative evidence objects are not hash test oracles. These vectors are normative for canonicalization and hashing; disagreement with them is non-conformance, not a variant.
  Note that VEC-01 hashes a Signing View that the vector supplies already derived. Neither it nor any other vector in v0.4 tests the derivation from proposed action to displayed view (PP-CORE-02, PP-CORE-03), so passing the vectors is not evidence that a human saw a faithful view.
- **VEC-03 (MUST)** Reject the negative vector `invalid/action-intent.fractional-number.sample.json`. An implementation that canonicalizes and hashes it instead of rejecting it is non-conformant.
- **VEC-02 (MUST)** Pass the canonicalization edge-case vector (non-ASCII keys and values, key ordering by UTF-16 code unit, integer serialization, escaping and JSON literals; fractional syntax is rejected separately by VEC-03). This vector detects selected non-JCS encoders that pass the simpler cases; passing it does not establish JCS correctness for arbitrary input.
