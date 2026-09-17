# HAPS v0.4.0 — Human Approval and Presence Specification (Draft)

**Status:** Experimental draft; not production-ready
**Version:** 0.4.0  
**Track:** Standards input / MCP Profile

This repository and its partial reference implementation are experimental. Passing the supplied tests
does not establish full protocol conformance or that a person saw and approved the correct action.
MUST/SHOULD statements describe intended implementation obligations, not completed features or proven
security outcomes. See [limitations](../../docs/limitations.md).

## 1. Abstract

HAPS (Human Approval and Presence Specification) proposes a protocol intended to support explicit human approval of a machine-readable Action Intent. It specifies a signed Consent Credential binding a provider's assertions of presence and approval, and optional identity evidence, to action data for relying-party validation and audit.

HAPS is intended to carry evidence of **participation**, subject to faithful display, approval capture, factor verification and deployment controls. Credential validation alone does not establish actual human presence, correct display or approval, or that the human was **authorized** to perform the action. Authorization remains the relying party's decision, made against its own account model or a mandate layer above HAPS. A Consent Credential is an input to that decision, never the decision itself.

HAPS is designed to be:
- **Vendor-neutral** (multiple certified Presence Providers can implement it),
- **Compatible with existing integration patterns** (an MCP tool profile + URL-mode consent UI; these are design targets, not a production-readiness claim),
- **Policy-driven** (Relying Parties decide whether PoHP alone is sufficient, or identity binding is also required),
- **Deepfake-aware** (PoHP assurance levels and certification evidence).

## 2. Motivation

Agents are moving from “assist” to “act” (payments, permissions, data deletion, account recovery). Traditional “consent prompts” are typically:
- UI-only and non-portable,
- difficult to audit,
- vulnerable to replay,
- vulnerable to UI/intent mismatch.

The draft specifies:
1) a deterministic **Action Intent** DSL,
2) a deterministic **Signing View** and **presentation_hash** intended to support WYSIWYS (what you show is what you sign), conditional on faithful derivation and rendering,
3) a portable **Consent Credential** (HAPS-CC),
4) optional **Identity Binding** (pluggable adapters),
5) a **Relying Party Challenge** pattern to enforce policy at the API boundary.

### 2.1 Relationship to existing mechanisms

HAPS is intended as a binding layer that composes with existing mechanisms. Its proposed contribution
is a common credential format linking asserted approval evidence to a structured action. The scope and
assurance of the underlying mechanisms depend on their protocols and deployments.

| Mechanism | Evidence it can supply | Relationship to HAPS |
| --- | --- | --- |
| WebAuthn / passkeys | An authenticator assertion bound to an RP and challenge, with presence/verification flags subject to authenticator behavior | A deployment must bind that assertion to the action and evaluate the actual factor and display path; a UV flag alone does not establish approval or biometric use. |
| OIDC step-up (`acr`, `amr`, claims challenge) | How and how recently the user authenticated | Authentication freshness, not approval of a particular action. The token says how you logged in, not what you agreed to. |
| Agent-signed request | Use of the agent's signing key | Does not by itself establish human participation or approval. |
| Transaction authentication / dynamic linking | Scheme-specific evidence intended to bind confirmation to transaction details | May overlap with HAPS goals; interoperability and assurance depend on the particular scheme. |
| Verifiable Credentials | An issuer asserted attributes about a subject | HAPS-CC can use a VC representation. The draft specifies action/view hashes, audience, freshness and use constraints; enforcing them remains an implementation obligation. |
| Audit logs and consent records | Recorded events, with integrity depending on the logging system | Some logs are signed or independently verifiable; HAPS proposes a common action/evidence binding for such records. |

An implementer already operating any of the first three keeps it. HAPS defines the credential that makes
its result checkable by the party bearing the risk.

## 3. Goals and non-goals

### 3.1 Goals
HAPS v0.4.0 MUST:
- Support **Proof of Human Presence** (PoHP) with an assurance level.
- Bind the approval to the **exact intent** via `intent_hash`.
- Bind the approval to **what the human was shown** via `presentation_hash`.
- Support **optional identity binding** via adapter schemes, controlled by RP policy.
- Be implementable as an **MCP tool** with URL-mode consent UI.

### 3.2 Non-goals
HAPS does NOT:
- Define biometric algorithms.
- Require centralized biometric storage.
- Guarantee correct display, human presence, explicit approval, understanding, or freedom from coercion. The protocol requires display and binding behavior, but its data checks cannot establish these real-world events.
- Guarantee failure-free code, production readiness or prevention of all fraud.
- Replace transport/authN standards (OAuth/OIDC/WebAuthn); it composes with them.
- Establish that the human is **authorized** to perform the action, or convey standing authority, scope or delegation. HAPS carries assertions of participation and approval; authorization is the relying party's decision.

## 4. Terminology

- **Human Principal:** the natural person who approves an action. HAPS assumes, and never verifies, that this person holds authority over the agent's actions; establishing that is the relying party's responsibility.
- **Agent:** software acting on behalf of a principal.
- **Relying Party (RP):** service that executes an action and verifies proof.
- **Presence Provider (PP):** service that performs a presence flow and issues HAPS-CC; certification evidence is checked under the draft's requirements.
- **Action Intent (AI-INTENT):** canonical JSON describing the action to authorize.
- **Signing View (HAPS-SigningView):** deterministic object derived from AI-INTENT for display and hashing.
- **PoHP:** Proof of Human Presence with assurance level HAPS-PoHP-1..4.
- **HAPS Challenge (HAPS-CHAL):** RP-issued envelope requiring HAPS for an intent.
- **HAPS Consent Credential (HAPS-CC):** signed credential binding PoHP (+ optional identity) to intent_hash + presentation_hash + audience.
- **Identity Binding:** optional scheme that binds consent to an identity, via a pluggable adapter.

## 5. Data model overview

HAPS defines three primary objects:

1) **AI-INTENT** (Action Intent)  
2) **HAPS-CHAL** (Relying Party Challenge; optional but recommended for enforcement)  
3) **HAPS-CC** (Consent Credential)  

HAPS also defines:
- **HAPS-PCC** (Provider Certification Credential)

Object schemas are provided under [schemas](../../schemas). Schema validation alone does not enforce
all semantic checks, numeric syntax restrictions, signatures, freshness, replay or display requirements.

## 6. Action Intent (AI-INTENT) v0.4

### 6.1 Canonicalization and hashing

AI-INTENT MUST be canonicalized using **RFC 8785 (JCS)** prior to hashing.

`intent_hash = "sha256:" + base64url( SHA-256( UTF8( JCS(AI-INTENT) ) ) )`

The base64url encoding is unpadded.

#### Numeric restriction

AI-INTENT and any Signing View derived from it **MUST NOT** contain a JSON number with a fractional
part or an exponent. Integers in the range −(2^53 − 1) to 2^53 − 1 are permitted. A quantity that needs
decimal places — a monetary amount, a rate — MUST be carried as a string (for example `"250.00"`), or as
an integer in a documented minor unit.

A PP MUST reject an AI-INTENT that violates this restriction rather than canonicalize it. An RP MUST
reject one before canonicalization, alongside the other malformed-input checks in §12.

This restriction simplifies implementation and verification targets by excluding fractional and
exponent number serialization. RFC 8785 already defines deterministic serialization; the restriction
does not itself establish correctness. Integer boundaries, negative zero, Unicode and parsing still
need validation (see [property targets](verifiable-properties-v0.1.md)).

### 6.2 Profiles

AI-INTENT SHOULD include a `profile` string. Profiles define:
- required parameters,
- “must-display” fields,
- Signing View construction rules.

If profile is absent or unknown, PP MUST use Generic Profile rules.

Known profiles:
- `ai_ui_confirm_v1` — AI UI confirmation / WYSIWYS profile ([ai-intent-profiles/ai-ui-confirm-v1.md](ai-intent-profiles/ai-ui-confirm-v1.md))

### 6.3 Envelopes (bounded flexibility)

AI-INTENT MAY include `constraints.envelope` describing allowed parameter ranges or allow-lists (e.g., amount range, allowed payees). If present:
- PP MUST include envelope constraints in the Signing View,
- RP MUST enforce envelope constraints at execution time.

If the action falls outside envelope constraints, a new AI-INTENT and new consent are required.

### 6.4 Freshness and nonce placement

AI-INTENT is the semantic description of the business action. It MUST be stable across retries for the same decision.

AI-INTENT MUST NOT carry per-session freshness artifacts such as:
- OIDC `nonce`/`state`,
- one-time callback tokens,
- transport/session correlation secrets.

Freshness and replay resistance MUST be enforced outside AI-INTENT using:
- HAPS-CHAL `challengeId` + `expiresAt` (+ `rpProof` when present),
- HAPS-CC `jti`, `iat`, `exp` (and `nbf` when present),
- RP-side replay caches / single-use consumption.

For high-risk actions, RP SHOULD enforce HAPS Challenge Mode and MAY reject agent-initiated requests that lack a challenge.

## 7. Signing View and presentation_hash (WYSIWYS)

### 7.1 Signing View

PP MUST derive a Signing View from AI-INTENT:
- using the declared profile rules if recognized, otherwise
- using Generic Profile rules that include:
  - audience identity,
  - agent identity,
  - action.type,
  - action.parameters (deterministic, complete),
  - constraints (expiry, maxUses/oneTime, envelope).

### 7.2 Presentation hash

Signing View MUST be canonicalized using JCS and hashed:

`presentation_hash = "sha256:" + base64url( SHA-256( UTF8( JCS(SigningView) ) ) )`

PP MUST include `presentation_hash` in HAPS-CC.
RP MUST recompute `presentation_hash` from the Signing View derived from AI-INTENT and compare it,
as required by §12 and RP-CORE-05. The hash binds the view data; it does not attest to rendered pixels,
visibility, accessibility output or the human's perception and decision.

## 8. Proof of Human Presence (PoHP)

PoHP assurance levels are defined in `pohp-assurance-levels-v0.4.md` and referenced by string:
- `HAPS-PoHP-1` .. `HAPS-PoHP-4`

The factor that produced the result is named by `assurance.method` and registered in
`factors-v0.1.md`. Level and factor class are orthogonal: the level states how strong the evidence is,
the factor states what kind of evidence it is.

RP policy determines minimum required level by action type. Because a level alone does not say whether
an approval could have been produced by an agent rather than a human, RP policy SHOULD also constrain
accepted factor classes — at minimum, whether agent-accessible factors are permitted.

## 9. Identity Binding (pluggable adapters)

### 9.1 Policy knob

RP MAY require identity binding. Requirements are expressed in:
- HAPS-CHAL `requirements.identity`, and/or
- MCP tool args `requirements.identity`.

Fields:
- `mode`: `"none" | "preferred" | "required"`
- `schemes`: array of acceptable identity schemes (adapter identifiers)
- `policy`: scheme-agnostic and scheme-specific options

If `mode="required"` and identity binding cannot be satisfied, PP MUST NOT issue HAPS-CC.

### 9.2 Identity binding block

If identity binding is satisfied, PP MUST include:

`claims.identityBinding = { mode, scheme, subject, idp, assurance?, evidence? }`

Where:
- `scheme` identifies the adapter
- `subject` is the normalized subject identifier for that adapter
- `evidence` MAY embed or reference verifiable evidence (e.g., OIDC ID token)

### 9.3 Adapter requirements

Identity binding is adapter-based and identity-provider-agnostic. This document defines the binding
block and the policy knob; each adapter is specified separately under `adapters/`, and an adapter
specification is the single normative source for its own scheme.

An adapter specification MUST define:

- its scheme identifier, as it appears in `identityBinding.scheme`;
- the normalized subject it emits, and the subject `type` naming that normalization;
- the claims or attributes it validates, and the clock skew bound it applies;
- how it binds its own freshness value (nonce, session, or equivalent) to the HAPS approval session,
  without placing that value into AI-INTENT business semantics;
- what evidence, if any, it embeds so that an RP can re-verify the binding independently;
- the scheme-specific `policy` keys an RP may set.

Any OIDC-capable identity provider fits this shape; so do non-OIDC schemes that can produce a stable
normalized subject and bind it to the approval session. Adapters available in this draft are listed in
`adapters/`.

### 9.4 Authentication step-up is not approval

An identity adapter, and any authentication step-up it performs, establishes **who** — not **that a
human approved this action**. This holds however strong the authentication is, and regardless of
provider.

Therefore:

- A refreshed or step-up authentication token MUST NOT be treated as a substitute for a HAPS Consent
  Credential. The HAPS-CC remains the approval artifact; the token is identity evidence supporting it.
- An RP MUST NOT accept the presence of such a token as proof of human approval on its own.
- A PP MUST continue to issue a normal HAPS-CC after any enterprise or step-up identity flow completes.

Authentication step-up also carries no implication of property **H** (human actuation) from
`pohp-assurance-levels-v0.4.md`. Many enterprise step-up factors — push approval, delivered one-time
passcodes — are classified as agent-accessible in `factors-v0.1.md`. An RP that needs a human, rather
than an agent acting with the user's credentials, MUST express that as a factor requirement on
`assurance`, independently of identity policy.

Optional transport profiles that carry a specific provider's step-up semantics through the identity
flow are specified under `adapters/`.

## 10. Relying Party Challenge Mode (HAPS-CHAL)

An RP MAY enforce HAPS by returning a HAPS-CHAL to the client/agent when a request lacks sufficient authorization.

HAPS-CHAL MUST include:
- `challengeId`
- `expiresAt`
- `requirements` (PoHP + optional identity binding)
- `actionIntent`
- optional `rpProof` (signature over the challenge body)

RP MUST ensure `challengeId` is unique within RP scope and single-use.
RP SHOULD use short challenge lifetimes (for example, 300 seconds for high-risk actions).

If rpProof is present:
- PP MUST verify it prior to issuing HAPS-CC
- PP SHOULD display “Verified RP” to the user

If PP issues HAPS-CC from HAPS-CHAL input, PP MUST copy `challengeId` into `claims.challengeId`.

## 11. Consent Credential (HAPS-CC)

HAPS-CC MUST:
- be signed by PP
- include:
  - `intent_hash`
  - `presentation_hash`
  - `aud` (RP identifier)
  - `jti`, `iat`, `exp`
  - `assurance` (PoHP)
  - provider certification evidence (embedded or referenced)
- optionally include:
  - `nbf`
  - `identityBinding` (if performed)
  - challenge reference metadata

When issued from HAPS-CHAL input, HAPS-CC MUST include `challengeId` and it MUST equal the challenge used for the consent event.

Credential formats:
- `jwt` (JWS)
- `vc+json` (VC Data Model with a proof)

## 12. Verification requirements (RP)

RP verifying HAPS-CC MUST:
1. Verify PP signature and trust (keys/registry).
2. Verify `aud` matches RP.
3. Validate time claims with bounded clock skew (`maxClockSkewSeconds`, RECOMMENDED <= 300):
   - reject if `now > exp + skew`
   - if `nbf` present, reject if `now + skew < nbf`
   - reject if `iat > now + skew`
   - enforce max credential age policy relative to `iat`.
4. Recompute `intent_hash` from the received AI-INTENT using RFC 8785 (JCS) and match exactly.
5. Recompute `presentation_hash` from Signing View derived from the same AI-INTENT using RFC 8785 (JCS) and match exactly.
6. Enforce PoHP minimum level.
7. Verify provider certification supports asserted PoHP level.
8. Enforce one-time use when required (`jti` replay) with atomic consume semantics.
9. If Challenge Mode is expected by RP policy:
   - require `claims.challengeId`
   - verify it matches an outstanding RP challenge for this transaction context
   - verify the challenge is unexpired and not previously consumed
   - consume it atomically with action execution.
10. Enforce envelope constraints (executed action within bounds).
11. If identity binding required:
    - enforce scheme allow-list
    - enforce subject match against the adapter's normalized subject
    - enforce any scheme policies (MFA, auth contexts, allowed tenants)
    - if embedded evidence required, verify it.

RPs MUST reject malformed JSON inputs before canonicalization (including duplicate member names).
RPs SHOULD retain consumed `jti`/`challengeId` entries at least until `exp + skew`.

## 13. MCP profile

HAPS can be implemented as an MCP tool:
- `haps.request`

URL-mode consent UI is used for sensitive interaction.
See `mcp-profile-v0.4.md`.

## 14. Security considerations

HAPS is intended to support the following mitigations, subject to implementation and deployment:
- detection of view-data mismatch through `presentation_hash`, with faithful rendering separately required;
- replay resistance through audience/time checks and persistent atomic consumption of `jti`/`challengeId`;
- detection of action-data substitution through `intent_hash` and enforcement of the checked action at execution;
- assessment of spoofing, deepfake and agent-substitution risks through evaluated factors, certification and RP policy. Level or method labels alone establish none of these outcomes.

The draft and partial implementation do not establish protection against:
- compromised user devices
- coercion outside the protocol
- malware after authorization

**The Presence Provider is a trusted witness, not an authorization authority.** It asserts that a human
was present and approved a specific action; an RP must assess the evidence and provider trust. It cannot attest that the human was entitled to. A relying
party that treats presence as authorization has delegated its authorization decision to its witness, and
a provider that can forge presence then decides outcomes it was never meant to decide. The following controls are intended to support
that separation, and an RP relying on HAPS for consequential actions SHOULD apply all four:

1. Match `identityBinding.subject` against the RP's own account records (RP-IDB-03). Never take the
   provider's word for *who may act* — only for *who was there*.
2. Prefer verifiable evidence over asserted evidence at higher tiers: `requireEmbeddedEvidence` lets the
   RP re-verify the underlying assertion itself rather than trusting the provider's summary of it.
   RECOMMENDED from `HAPS-PoHP-3` upward.
3. Make the set of accepted providers explicit policy, with certification checked (RP-CORE-06,
   FACTOR-RP-04), rather than an accident of integration.
4. Accept more than one provider. A single accepted provider is a chokepoint whatever the specification
   says.

### 14.1 Symbolic analysis

A [Tamarin model](../../formal/tamarin/README.md) represents selected Challenge Mode (§10) and RP
verification (§12) behavior against a symbolic network attacker. Its lemmas concern relationships among
`Accept` and `HumanApproved` events and the consumption of a challenge. Those events are abstractions;
`HumanApproved` is an assumed atomic event, not an observation of a person or a rendered interface.
`view` is an abstract function, not a proved implementation of Signing View derivation.

The model includes conditional action-binding and replay statements and a provider-compromise trace
in which acceptance occurs without an approval event. An uncompromised provider key is an explicit
condition of the base binding statements. Correct signatures do not establish the truth of a provider's
assertions in a deployment; dishonest issuance, UI compromise and factor failures need separate analysis.

An exploratory [holder co-signature variant](../../formal/tamarin/variants/cosigned-holder-key.spthy)
adds a second symbolic signature. It studies a different key-compromise boundary under its own rules.
It does not implement WebAuthn, a trusted display or holder authentication, and does not establish that
adding a passkey produces the same real-world result. The variant is not a protocol requirement or an
implemented feature of the reference code.

The model omits wall-clock freshness, identity binding, factor semantics, executable rendering,
crash/persistence behavior and concrete cryptography. Negative controls explore removing challenge
consumption or the presentation hash. See the model README for exact statements and reproduction
commands; no model result establishes full specification conformance or fraud prevention.

Two further failure modes deserve explicit attention:

- **Self-asserted assurance.** Without a verifiable provider certification, `assurance.level` is asserted
  by whoever holds the signing key. An RP that does not verify certification is trusting a claim, not
  checking one (`pohp-assurance-levels-v0.4.md` §4).
- **Agent-accessible factors.** OTP and push approval can record credential access or an acknowledgement without establishing human presence; an
  agent with mailbox, SMS or device access can satisfy them. They are appropriate for low-risk actions
  and MUST NOT be relied on as evidence of human presence for consequential ones. A PoHP level alone
  does not convey this — the factor does.

## 15. Privacy considerations

- No biometric samples in HAPS-CC.
- Identity binding should use stable identifiers and minimize PII.
- Pairwise identifiers and selective disclosure are recommended for future versions.

## 16. Two kinds of certification

HAPS uses the word *certification* for two unrelated things. They are tested differently, evidence
different claims, and neither implies the other. Conflating them is the most likely way for a
deployment to believe it has an assurance it does not have.

**Conformance certification — does an implementation follow this specification?**

Scope: canonicalization and hashing, binding, freshness, replay consumption and policy evaluation.
Conformance requires satisfying all applicable requirements in [conformance-v0.4.md](conformance-v0.4.md),
including the [vectors](../../test_vectors/v0.4). Passing the supplied tests alone is insufficient: the
current kit covers selected canonicalization/hash behavior and does not certify a complete PP or RP.
This draft defines no operational conformance-certification service. Core checks can be evaluated
locally with provisioned trust inputs; complete deployment evaluation needs evidence beyond the kit.

It says nothing whatsoever about the quality of any presence signal. A Presence Provider can be fully
conformant and still produce presence evidence that a photograph would defeat.

**Assurance certification — does a provider actually deliver the PoHP level it asserts?**

Scope: the attack resistance behind `assurance.level` and `assurance.method` — presentation-attack
resistance, injection resistance, hardware binding, and the operational controls around them. It is
carried on the wire by `providerCertification` (HAPS-PCC,
`provider-certification-credential.v0.4.schema.json`), whose `evaluation` block references the
evaluation that produced it. Establishing it requires evaluation by a party other than the provider;
`pohp-assurance-levels-v0.4.md` §1 property **C** is this and nothing else.

It says nothing about protocol correctness. A provider with impeccable biometrics can canonicalize
incorrectly and produce credentials no relying party can verify.

### 16.1 Requirements

- An implementation **MUST NOT** present conformance with this specification as evidence of any PoHP
  level, or as evidence of the quality of a presence signal.
- A relying party **MUST NOT** infer an assurance level from an implementation's conformance status, and
  **MUST NOT** infer protocol conformance from a provider's assurance certification.
- A HAPS-PCC **MUST** state the levels and factors it certifies (`certifiedLevels`, `certifiedFactors`)
  and the evaluation behind it. A certification that names neither an evaluation nor an evaluator is not
  evidence.
- Where both are claimed, they **SHOULD** be reported separately, so a reader can see which is present.

### 16.2 Governance is out of scope

This specification defines what a certification asserts and how it travels. It does not name a
certifying body, an accreditation scheme, or an evaluation methodology, and it confers no authority on
any organisation to operate one. Which certifiers a relying party trusts is RP policy
(`accepted_certifiers`), not a property of the protocol.

An accreditation scheme that only one vendor could satisfy would defeat the vendor-neutrality this
specification is built for, and is a reason to be careful about who defines one.

## 17. Digital Credentials + portrait/liveness profile (informative)

HAPS deployments MAY use `haps.profile.digital-credentials-portrait-liveness/v0.1` when a relying party requires a step-up based on a trusted digital identity credential, wallet holder authentication, and optionally an independent liveness / portrait-match result.

This profile supports both plain Digital Credentials / Apple Wallet presentment and a combined Digital Credentials + liveness provider flow. A liveness provider MAY be treated either as an independent biometric evidence service or as a trusted issuer of identity-presence attestations, depending on RP policy.

HAPS core does not define Apple Wallet, DCA, mdoc, OpenID4VP, or biometric algorithms. The profile defines how verified evidence from those systems is bound to the HAPS challenge/session and carried into `identityBinding` without embedding raw portrait or biometric material.

## Changes in v0.4.0

Breaking (identifier namespace):

- `haps` is the root identifier namespace: namespaced profile ids use `haps.profile.*` and the MCP tool is
  `haps.request`. Earlier drafts used an organisation prefix; no organisation's name appears in a HAPS
  namespaced identifier. The UI mapping annex retains `ai_ui_confirm_v1`; it has no defined
  namespaced alias. Profile identifiers in sample JSON must be preserved when reproducing hashes.
- PoHP level strings are `HAPS-PoHP-1` .. `HAPS-PoHP-4`.
- Schema `$id`s are URNs (`urn:haps:schema:<name>:<version>`) rather than HTTP URLs, so identifier
  stability does not depend on any party continuing to own a domain.
- Schemas and test vectors move to v0.4 accordingly. Because these strings are inside the canonical
  JSON, every published hash changed; the v0.4 test vectors are authoritative.

Additive:

- New `factors-v0.1.md`: factor registry, `assurance.method` values, and each factor's
  agent-resistance posture.
- `pohp-assurance-levels-v0.4.md` restates the four levels as evidence properties rather than as a
  liveness modality, so non-liveness factors are placeable. Level semantics for liveness are unchanged.
- Provider certification credential v0.4 adds `certifiedFactors`.
- New conformance classes for factor declaration and factor policy, and normative test-vector
  requirements.
- New §2.1 relating HAPS to WebAuthn, OIDC step-up, transaction authentication and Verifiable
  Credentials.
- New §14.1 records the Tamarin symbolic analysis of Challenge Mode against a Dolev-Yao
  attacker: action binding, injective agreement and no approval reuse, with the provider-compromise
  case exhibited rather than assumed away.
- New §16 separates conformance certification (does an implementation follow this specification)
  from assurance certification (does a provider deliver the level it asserts), with requirements that
  neither be presented as the other.
- §8 states that assurance level and factor class are orthogonal. §14 names self-asserted assurance
  and agent-accessible factors as explicit failure modes.

Editorial:

- The specification is vendor-neutral: no vendor is named as an example presence or liveness provider.
  Vendor specifics live in optional adapter profiles under `adapters/`.

## References
- RFC 8785 JSON Canonicalization Scheme: https://www.rfc-editor.org/rfc/rfc8785
- MCP Specification: https://modelcontextprotocol.io/specification/2025-11-25
