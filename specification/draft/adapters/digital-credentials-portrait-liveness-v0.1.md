# HAPS Digital Credentials + Portrait/Liveness Profile v0.1

**Profile ID:** `haps.profile.digital-credentials-portrait-liveness/v0.1`
**Status:** Experimental draft / optional adapter profile; not production-ready.
**Applies to:** HAPS v0.4.x

HAPS has a partial reference implementation. Passing the supplied tests does not establish full
protocol conformance, wallet or biometric interoperability, or that a person saw and approved the
correct action. This profile specifies intended evidence handling under deployment assumptions.

## Abstract

This profile describes evidence that may support a HAPS step-up using a Digital Credentials / Apple
Wallet / mdoc presentment, optionally combined with an independent portrait/liveness provider, when
the implementation and deployment meet the RP's policy.

The profile supports three deployment modes:

1. **Plain Digital Credentials mode** — the Presence Provider verifies a Digital Credentials / Apple Wallet / mdoc presentation and records available wallet holder-authentication evidence. This alone does not establish human actuation or action approval.
2. **Independent liveness mode** — the Presence Provider or RP verifies a Digital Credential presentation, then obtains a separate liveness / portrait-match result from a biometric provider.
3. **Trusted liveness issuer mode** — the liveness provider is itself trusted to verify the portrait-bearing credential and issue a signed identity-presence attestation.

In all modes, the final HAPS Consent Credential carries only evidence references, hashes, assurance levels, timestamps, and issuer identifiers. Raw portraits, biometric samples, face embeddings, and biometric templates MUST NOT be embedded in the HAPS Consent Credential.

## 1. Motivation

A relying party may request evidence of holder authentication and, separately, human participation.
Potential evidence sources include:

- Apple Wallet / Digital ID presentment;
- W3C Digital Credentials API presentment;
- ISO mdoc / mobile driving licence / national ID presentment;
- EUDI PID or age-verification presentment;
- liveness and face matching against a portrait from a trusted credential.

This profile describes how to represent those events as an identity-binding result for an RP or
integration bridge. It does not establish that the evidence is sufficient for a particular action.

## 2. Terminology

- **Digital Credential Presentation:** A wallet or browser-mediated credential presentation, including DCA, OpenID4VP, or ISO mdoc-based responses.
- **Portrait Credential:** A trusted credential that contains or references a portrait/ID photo.
- **Liveness Provider:** A provider that performs anti-spoofing/liveness and, when configured, face match against a reference portrait.
- **Identity Presence Binding:** The profile-specific evidence object binding a credential presentation, optional liveness/portrait-match result, and HAPS approval session.

## 3. Required binding properties

A conforming Presence Provider or bridge MUST ensure that:

1. the Digital Credential presentation is bound to the same HAPS challenge or approval session;
2. the presentation includes an anti-replay nonce or equivalent session transcript;
3. any liveness/portrait-match evidence is bound to the same `journey_id` / `challenge_id` / `approval_session_id`;
4. any trusted liveness issuer attestation is signed and issuer-verifiable;
5. the final HAPS-CC references the evidence through hashes or opaque references rather than raw biometric material.

## 4. Modes

### 4.1 Plain Digital Credentials mode

The wallet presentation supplies credential and available holder-authentication evidence. This mode
may be used when that evidence meets RP policy; wallet authentication is not automatically an
agent-resistant factor or approval of the HAPS Action Intent.

Required evidence:

- credential source (`apple_wallet`, `digital_credentials_api`, `eudi_wallet`, `mdoc`, `openid4vp`, or `other`, per the evidence schema);
- credential protocol (`org-iso-mdoc`, `openid4vp`, `digital_credentials_api`, etc.);
- presentation hash/reference;
- issuer/trust-anchor reference;
- holder-authentication result where available;
- challenge/session binding.

### 4.2 Independent liveness mode

The RP or HAPS bridge verifies the credential presentation, extracts or references the trusted portrait, and then invokes a liveness/face-match provider.

Required evidence:

- all plain Digital Credentials mode fields;
- portrait reference/hash;
- liveness provider ID;
- liveness result;
- match result, if portrait match is required;
- proof that liveness and credential presentation are bound to the same session/challenge.

### 4.3 Trusted liveness issuer mode

The liveness provider is trusted to verify the credential presentation and issue an attestation. This mode is useful where a liveness provider acts as a delegated identity-presence verifier.

Required evidence:

- liveness issuer ID and key reference;
- signed attestation reference/hash;
- credential presentation hash/reference;
- portrait hash/reference where applicable;
- liveness and match results;
- challenge/session binding;
- expiry/freshness.

## 5. Identity Binding representation

When this profile is used, the HAPS-CC `claims.identityBinding` SHOULD contain the fields shown below.
The following is an illustrative identity-binding fragment, not an issued credential or verification
result. `verified` and `passed` are result labels and may be emitted only when the corresponding
checks have succeeded; their presence in JSON supplies no evidence by itself.

```json
{
  "mode": "verified",
  "scheme": "digital_credentials_portrait_liveness",
  "idp": {
    "issuer": "https://identity-issuer.example"
  },
  "subject": {
    "type": "credential_holder",
    "pairwise_id": "optional-pairwise-subject",
    "binding_key_thumbprint": "optional-key-thumbprint"
  },
  "assurance": {
    "digital_credential": "verified",
    "holder_auth": "verified",
    "portrait_match": "passed",
    "liveness": "passed"
  },
  "evidence": {
    "profile": "haps.profile.digital-credentials-portrait-liveness/v0.1",
    "mode": "independent_liveness",
    "evidence_hash": "sha256:...",
    "evidence_ref": "ipba-..."
  }
}
```

The separate [evidence schema](../../../schemas/haps-digital-credentials-portrait-liveness.v0.1.schema.json)
and [sample](../../../test_vectors/v0.4/haps-digital-credentials-portrait-liveness.sample.json) describe
the evidence object referenced here. Its `binding` requires `challenge_id`, `approval_session_id`,
`audience`, and `issued_at`; `journey_id` is optional. Session-only operation without `challenge_id`
is not representable by that schema in this draft. These evidence names use underscores; the core
HAPS-CC challenge field remains `claims.challengeId`.

## 6. Privacy requirements

A conforming implementation:

- MUST NOT embed raw portrait images in HAPS-CC;
- MUST NOT embed raw biometric samples in HAPS-CC;
- MUST NOT embed face embeddings or biometric templates in HAPS-CC;
- SHOULD minimize retained attributes to evidence references, hashes, issuer IDs, timestamps, result flags, and assurance levels;
- SHOULD support pairwise subject identifiers where the underlying credential protocol permits.

## 7. RP verification

An RP consuming this profile MUST verify:

- HAPS-CC signature and core HAPS claims;
- `identityBinding.scheme == digital_credentials_portrait_liveness` when policy requires this profile;
- evidence hash/reference consistency when embedded evidence is supplied;
- challenge/session binding;
- evidence freshness;
- issuer/trust-anchor policy for the credential and optional liveness issuer.

The RP's freshness policy needs a maximum evidence age, a bounded clock skew, and handling of missing
expiry. This draft does not define common values for these profile-specific settings; documenting
and enforcing them is part of an integration, alongside the core credential time checks.

If trusted liveness issuer mode is used, the RP MUST verify the issuer signature or rely on an explicitly trusted bridge that has verified it.

## 8. Non-goals and limitations

This profile does not define:

- Apple Wallet or DCA wire protocols;
- biometric algorithms;
- issuer certification programmes;
- raw image storage or retention policy beyond the privacy constraints above.

Credential presentation, holder authentication, portrait matching, liveness, and action approval are
different events. None alone establishes all the others. Liveness and matching can return incorrect
results; resistance to spoofing or injection depends on evaluated algorithms, capture paths, and
deployment controls. A trusted issuer or bridge may also make an incorrect assertion.

Hashes and signatures bind represented evidence under cryptographic and key-management assumptions.
They do not establish actual screen output, informed approval, authority to act, or absence of fraud.
The RP must also bind execution to the validated action and enforce its replay and authorization
policy. Schema validation checks data shape and does not validate credential signatures, biometrics,
or the truth of a `passed` result.
