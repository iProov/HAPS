# HAPS — Human Approval and Presence Specification

HAPS is a proposed protocol for binding a provider's assertion of human presence and approval to a
machine-readable action. It is intended to support relying-party checks of that evidence, subject to
correct implementation, trustworthy providers and factors, and deployment controls.

**Status: experimental draft specification and partial reference implementation; not production-ready.**
This repository contains the draft specification; the partial implementation is in
[iProov/HAPS-reference](https://github.com/iProov/HAPS-reference). Passing the supplied tests does not
establish full protocol conformance or that a person saw and approved the correct action. See
[limitations and implementation status](docs/limitations.md).

## What HAPS does

An agent proposes an action. HAPS specifies a signed **Consent Credential** containing a provider's
assertions and data bindings that a relying party is intended to validate:

- **`intent_hash`** — a hash of the canonical Action Intent covered by the assertion.
- **`presentation_hash`** — a hash of the Signing View the provider asserts it displayed. Matching
  hashes does not establish what appeared on screen or what a person perceived or approved.
- **`aud`**, **`challengeId`**, **`exp`**, single-use **`jti`** — fields intended to constrain audience, freshness and use.
  Replay protection depends on verifier policy and atomic, persistent consumption state.
- **`assurance`** — the asserted presence factor and assurance level, requiring policy and evidence checks.

HAPS does **not** replace OAuth, OIDC, WebAuthn, or any wallet or biometric system; it composes with
them. It does not confer standing authority or scope — that belongs to a mandate or capability layer
above it. A presence assertion does not establish authorization. The intended use is to assess evidence
of fresh approval of a specific action; the credential alone cannot establish real-world approval.

## Factors and agent resistance

HAPS is factor-agnostic. Any factor that produces a fresh, attributable human signal can satisfy a
step-up: physically actuated secure hardware, smart-card or human-held signing keys, biometric passkeys,
WebAuthn, and liveness. Lower-risk interactions may use agent-accessible factors such as OTP or push
approval.

That distinction is first-class, because it is the one that matters for a protocol about human approval:
an **agent-resistant** factor is intended to require a human act even when software has access to the
user's device, session and credentials, under an evaluated threat model. The method name alone does not
establish that resistance. An **agent-accessible** factor can be satisfied by software in that threat
model. A relying party can require evidence of agent resistance independently of the level claimed.

The factor registry, the `assurance.method` values that name each factor on the wire, and each factor's
agent-resistance posture: **[specification/draft/factors-v0.1.md](specification/draft/factors-v0.1.md)**.

## Documents

| Document | Role |
| --- | --- |
| [Limitations](docs/limitations.md) | Experimental status, trust assumptions and implementation gaps |
| [haps-v0.4.0.md](specification/draft/haps-v0.4.0.md) | Core specification |
| [factors-v0.1.md](specification/draft/factors-v0.1.md) | Human-presence factor registry and agent resistance |
| [pohp-assurance-levels-v0.4.md](specification/draft/pohp-assurance-levels-v0.4.md) | PoHP assurance levels, stated as evidence properties |
| [conformance-v0.4.md](specification/draft/conformance-v0.4.md) | Conformance classes and testable MUST/SHOULD requirements |
| [verifiable-properties-v0.1.md](specification/draft/verifiable-properties-v0.1.md) | The core requirements restated as machine-checkable properties, with the boundary of what they do not cover |
| [mcp-profile-v0.4.md](specification/draft/mcp-profile-v0.4.md) | MCP transport profile |
| [ai-intent-profiles/ai-ui-confirm-v1.md](specification/draft/ai-intent-profiles/ai-ui-confirm-v1.md) | Action Intent profile for UI confirmation |
| [adapters/](specification/draft/adapters) | Optional identity-binding adapters and transport profiles, each the normative source for its own scheme |
| [schemas/](schemas) | JSON Schemas for Action Intent, HAPS-CHAL, HAPS-CC, and provider certification |
| [test_vectors/v0.4/](test_vectors/v0.4) | Canonicalization/hash vectors and illustrative policy/evidence examples |
| [formal/tamarin/](formal/tamarin) | Tamarin symbolic model of Challenge Mode against a Dolev-Yao attacker, with negative controls |

## Test vectors

Canonicalization is RFC 8785 (JCS), hashed with SHA-256 and encoded base64url without padding. The
three input/expected-hash pairs are normative. The edge-case vector detects selected non-JCS
encoders, while the policy and identity objects are illustrative examples. See [test_vectors/v0.4/README.md](test_vectors/v0.4/README.md).

## Versioning

Core protocol documents — the specification, conformance, the MCP profile, the PoHP levels — share the
protocol release, currently **v0.4.0**. Schemas, wire `version` fields and vector directories use
**v0.4** / **`"0.4"`** for that release family. Component registries and optional adapter profiles carry their own versions and move
independently. Identifiers that appear inside hashed canonical JSON (profile ids, PoHP level strings)
are part of the wire: changing one changes the hashes of documents containing it. For example,
`ai_ui_confirm_v1` is the UI mapping annex's current identifier; its `v1` is independent of the
core release. The annex does not yet define a complete executable derivation.
The specification lives in **iProov/HAPS**; the partial reference implementation lives in
**iProov/HAPS-reference**.

## Implementations

Conformance is defined by [conformance-v0.4.md](specification/draft/conformance-v0.4.md) and the
[test vectors](test_vectors/v0.4) — not by any particular codebase. An implementation is conformant when
it reproduces the vectors and satisfies the requirements for the classes it claims.

The reference implementation lives at **[iProov/HAPS-reference](https://github.com/iProov/HAPS-reference)**
and currently implements canonicalization, hashing and a test adapter. Its checked-in **Lean 4**
proof source contains two theorems about a single extracted comparison step, conditional on the step
returning a successful final verdict. These do not prove whole-string ordering, termination or the
verifier. **Kani** contributes four bounded harnesses, not a crate-wide panic-freedom result. Lean
extraction/build is not run in CI and needs external toolchain dependencies. Cryptographic primitives
are outside that proof scope; credential signature verification is not implemented. See
[docs/reference-implementation-plan.md](docs/reference-implementation-plan.md) and the property targets in
[verifiable-properties-v0.1.md](specification/draft/verifiable-properties-v0.1.md).

The reference implementation has not established the full published property set or any complete PP/RP
conformance class. Conformance requires satisfying all applicable requirements as well as the vectors;
a passing test report alone is insufficient.

Independent implementations in any language are encouraged and are not second-class: conformance is
defined by the document and the vectors. Core verification can run locally with provisioned keys,
certifications, policy, a clock and replay state (see "Smallest conformant implementation" in the
conformance document). Deployment may still need network services for key updates, identity adapters
or provider interactions.

## Participation

- [CONTRIBUTING.md](CONTRIBUTING.md) — what is most useful to contribute, and what to do when a change
  moves a published hash.
- [SECURITY.md](SECURITY.md) — how to report a vulnerability privately, and what counts as one in a
  specification.
- [CODE_OF_CONDUCT.md](CODE_OF_CONDUCT.md) — Contributor Covenant v2.1.

Independent implementations and adversarial review are the reason this is published. If a requirement
admits two readings, that is a defect worth an issue.

## License and IPR

Licensed under the [Apache License, Version 2.0](LICENSE). The following notice is also provided in
[NOTICE](NOTICE).

Scope of this License:

This repository, comprising the Human Approval and Presence Specification (HAPS) specification, reference application, and accompanying materials (the “Work”) is licensed under the Apache License, Version 2.0. Use of the Work under the License grants no rights to any patent, copyright, trade secret or other intellectual property right in iProov’s proprietary liveness technology, models, or services.

For clarity, while iProov welcomes your use of the HAPS specification and procedure in accordance with the License, including as incorporated into third-party products and services, use of the iProov name or branding (including trade marks) beyond identifying the origin of this Work, including any suggestion of endorsement, certification, or partnership, is not licensed (see Apache License 2.0, Section 6).
