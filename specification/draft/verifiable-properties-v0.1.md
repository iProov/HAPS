# HAPS Verifiable Properties v0.1 (Draft)

**Status:** Experimental draft property catalogue; not a verification report
**Applies to:** HAPS v0.4.x

## 1. Purpose

The conformance document states requirements an implementation must satisfy. This document restates the
core of them as **properties over a verifier's inputs and state**, precise enough to be mechanized.

These are intended properties and verification targets, not completed proofs. The partial reference
implementation has not established this property set and is not production-ready. Its finite tests and
bounded harnesses do not establish full protocol conformance or that a person saw and approved the
correct action. See [limitations](../../docs/limitations.md) and [implementation status](../../docs/reference-implementation-plan.md).

The [core specification](haps-v0.4.0.md) and [conformance requirements](conformance-v0.4.md) determine
MUST/SHOULD obligations. This catalogue includes strengthened research targets (notably unconditional
state retention, totality and policy monotonicity); it does not silently promote SHOULD requirements to
MUSTs or add conformance obligations. §5 maps related requirements, not proof completion.

## 2. Notation

- `V` — a parsed JSON value. `canon(V)` — its RFC 8785 canonical UTF-8 encoding.
- `H(b) = "sha256:" ++ base64url_unpadded(SHA-256(b))`.
- `view(I)` — the Signing View derived from AI-INTENT `I` under its profile.
- `cc` — Consent Credential claims. `π` — RP policy. `now` — the verifier's clock reading.
- `S` — verifier state (consumed `jti`, outstanding challenges). `verify(S, I, cc, π, now)` returns
  `Accept(S')` or `Reject(e)` for a typed error `e`.

## 3. Properties

### 3.1 Canonicalization and hashing

- **P1 — Canonical form is a function.** For every AI-INTENT `I` admissible under §6.1, `canon(I)` is
  uniquely determined by the parsed value of `I`. No admissible value has two canonical encodings.
  *RFC 8785 already specifies a deterministic serialisation for every JSON number, so P1 does not
  depend on §6.1. Excluding fractional and exponent numbers simplifies implementation and verification targets —
  an encoder never needs the shortest-round-trip float algorithm — but it is a simplification, not the
  source of determinism.*
- **P2 — Determinism.** Equal parsed values have equal hashes: `I₁ = I₂ ⟹ H(canon(I₁)) = H(canon(I₂))`.
- **P3 — Parse rejection.** A document with duplicate member names, or with a number carrying a
  fractional part or exponent, is rejected before canonicalization. `canon` is never applied to it.
- **P4 — Signing View derivation is pure.** `view(I)` is a total function of `I` and its profile alone:
  no clock, no randomness, no ambient configuration. Two implementations agreeing on `I` and the profile
  produce identical views.

### 3.2 Binding

- **P5 — Acceptance implies binding.**
  `verify(S, I, cc, π, now) = Accept(_) ⟹ cc.intent_hash = H(canon(I)) ∧ cc.presentation_hash = H(canon(view(I)))`.
  Intended condition: acceptance requires matching data hashes. This says nothing about actual rendering
  or human approval and is not a statement about the current code.
- **P6 — Acceptance implies audience.** `Accept ⟹ cc.aud = π.expected_aud`.
- **P7 — Acceptance implies freshness.** `Accept ⟹ now ≤ cc.exp + π.skew`, and if `nbf` is present,
  `cc.nbf − π.skew ≤ now`. Required `iat`/`exp` are well-typed; `iat ≤ now + π.skew`, and the
  credential meets the RP's maximum-age policy. These are clock-relative conditions, not a proof of
  real-world freshness.
- **P8 — Acceptance implies challenge binding when required.** If `π` requires Challenge Mode, then
  `Accept ⟹ cc.challengeId` is present, is in `S`'s outstanding set, is unexpired, and was unconsumed.

### 3.3 State

- **P9 — Single use when required.** When one-time use is required, for any `jti` at most one `Accept`
  is permitted during its acceptance window: `verify(S, …) = Accept(S') ⟹ verify(S', …) = Reject(Replay)` for the same `cc`.
- **P10 — State monotonicity.** Consumed identifiers are never removed while they could still be
  presented: for all transitions `S → S'`, `consumed(S) ⊆ consumed(S')` until `exp + skew` has passed.
- **P11 — Atomicity when single use is required.** No interleaving of two `verify` calls on the same `jti` yields two `Accept`
  results. Consumption and acceptance are a single atomic step.

### 3.4 Assurance policy

- **P12 — Level ordering.** `rank` is a total order on the four PoHP levels, and `Accept ⟹
  rank(cc.assurance.level) ≥ rank(π.min_level)`.
- **P13 — Factor policy.** `Accept ⟹ cc.assurance.method ∈ π.accepted_factors` when `π` constrains
  factors. An unrecognised method is classified agent-accessible.
- **P14 — Agent resistance is not implied by level.** If `π.require_agent_resistant` then
  `Accept ⟹ cc.assurance.method` is an agent-resistant class in `factors-v0.1.md`, **independently of**
  `cc.assurance.level`. No level assertion can satisfy this requirement on its own.
- **P15 — Certification coverage.** For RP core conformance (RP-CORE-06), `Accept ⟹` a certification
  is resolvable for the credential's issuer, is unexpired at `now`, and lists
  `cc.assurance.level` in `certifiedLevels`; when factor coverage is required, it also lists
  `cc.assurance.method` in `certifiedFactors`. A lower assurance classification cannot bypass required
  certification validation.

### 3.5 Policy monotonicity

- **P16 — Stricter policy never accepts more.** For policies ordered by strictness, if `π₁ ≼ π₂` (`π₂` at
  least as strict on every dimension: level, factors, agent resistance, certification, identity,
  challenge) then `verify(S, I, cc, π₂, now) = Accept ⟹ verify(S, I, cc, π₁, now) = Accept`.
  Equivalently: tightening a policy can only ever reject more. This is the property that rules out a
  whole class of bug in which an additional requirement accidentally opens a path.

### 3.6 Totality and privacy

- **P17 — Totality (research target, not established).** A future verifier argument would need a defined
  input/resource model and show termination with `Accept` or a typed `Reject` within that model, including
  rejection of malformed inputs. No such result is claimed for the current code. Allocation failure,
  stack exhaustion and other runtime failures are not ruled out by the supplied checks.
- **P18 — No raw biometrics.** An accepted credential contains no raw portrait, raw biometric sample,
  biometric template, or face embedding field, at any depth.

## 4. What these properties do not cover

Stated plainly, because a verification claim is only as useful as its boundary:

- **Cryptographic primitives.** SHA-256 and signature verification are assumptions of these statements.
  The reference code uses `sha2`; this project does not prove that primitive and has no credential
  signature verifier.
- **Signature verification and key trust.** Whether the right key was used, and whether that key should
  be trusted, is policy and PKI, outside these properties.
- **The JSON parser.** Properties begin at the parsed value. P3 is an obligation *on* the parser; a
  verifier using an unverified parser inherits that risk and MUST enforce P3 explicitly.
- **Rendering and human behavior.** A pure `view` function and matching hashes do not establish faithful
  rendering, actual human presence, understanding or explicit approval of the intended action. Factor
  names and assurance levels do not themselves establish resistance to real-world attacks.
- **Whether the specification is the right specification.** Proving an implementation matches this
  document says nothing about whether the document prevents real-world fraud.
- **Side channels, resource exhaustion, and clock correctness.** `now` is an input; a verifier fed a
  wrong clock may accept stale credentials even if it implements the clock-relative predicate.

## 5. Mapping to conformance

| Property | Conformance requirement |
| --- | --- |
| P1, P2 | PP-CORE-01, RP-CORE-04, RP-CORE-05, VEC-01, VEC-02 |
| P3 | PP-CORE-02a, RP-CORE-04a, VEC-03, §12 malformed-input rejection |
| P4 | PP-CORE-02; derivation purity alone does not establish PP-CORE-03 display |
| P5 | RP-CORE-04, RP-CORE-05 |
| P6 | RP-CORE-02 |
| P7 | RP-CORE-03 |
| P8 | RP-CORE-09 |
| P9, P11 | RP-CORE-07 |
| P10 | RP-CORE-10 (SHOULD; the property is a stronger target) |
| P12 | RP-CORE-06 |
| P13, P14 | FACTOR-RP-01, FACTOR-RP-02, FACTOR-RP-03 |
| P15 | RP-CORE-06, FACTOR-RP-04 |
| P16 | (emergent; no single requirement — it is the consistency of the policy dimensions) |
| P17 | (implementation quality; not currently a stated requirement) |
| P18 | PP-CORE-05, DCA-PP-04 (requires content validation beyond the current schemas and tests) |
