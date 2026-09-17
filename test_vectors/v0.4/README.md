# Test vectors (HAPS v0.4)

**Status:** Experimental draft test material for a partial reference implementation; not production-ready.
Passing the supplied tests does not establish full protocol conformance or that a person saw and
approved the correct action.

The three input/expected-hash pairs below are **normative for canonicalization and hashing**
(`conformance-v0.4.md`, VEC-01/VEC-02). The policy and identity evidence objects are illustrative; their
assertions and placeholder digests are not additional cryptographic test expectations.

## Algorithm

Both hashes are computed identically:

1. Canonicalize the JSON document with **RFC 8785 (JCS)** — object keys sorted by UTF-16 code unit,
   no insignificant whitespace, minimal string escaping, UTF-8 output.
2. `SHA-256` over the canonical UTF-8 bytes.
3. Encode as base64url **without padding** and prefix with `sha256:`.

## Hashing vectors

| Input | Expected |
| --- | --- |
| `action-intent.payment.transfer.sample.json` | `expected_intent_hash.txt` |
| `signing-view.payment.transfer.sample.json` | `expected_presentation_hash.txt` |
| `canonicalization-edge-cases.sample.json` | `expected_canonicalization_edge_cases_hash.txt` |

The `haps.profile.payment.transfer/v0.4` identifier is illustrative; this bundle does not define a
complete payment-profile derivation. Preserve the sample bytes for hash tests. Implementations that
do not recognize that profile must apply the core Generic Profile rules, whose executable derivation
also needs agreement between implementations.

The Signing View sample accompanies the Action Intent as a separate, already-derived JSON document.
The supplied hash checks do not invoke or validate the derivation; see "Signing View derivation" below.

### The edge-case vector

The first two vectors happen to hash identically under true JCS and under a naive "sorted keys, compact
separators" encoder, because their keys are all ASCII and they contain no numeric literals. That makes
them unable to detect a non-JCS canonicalizer.

`canonicalization-edge-cases.sample.json` is built to detect one. It contains:

- **Keys whose UTF-16 code-unit order differs from their UTF-8 byte order.** `"🔐"` (a
  surrogate pair) sorts *before* `"Ｚ"` under JCS and *after* it under byte ordering.
- **Non-ASCII string values** (`£`, `naïve`, `日本語`, an emoji), which an encoder using
  ASCII escaping (`\u00a3`) will hash differently.
- **Escapes** — quote, backslash, tab, newline.
- **Integer literals**, plus empty object, empty array, booleans and null.

The expected hash detects those particular ordering and escaping differences. A match on this finite
example does not establish that an encoder implements JCS correctly on every input.

The vector exercises a few integer literals, not full numeric boundary or negative-zero handling.
§6.1 excludes fractional and exponent syntax from AI-INTENT; rejection and safe-integer range checks
still need separate validation. The negative vector below exercises one fractional-number rejection.

## Policy vectors

These are unsigned `claims` objects (the payload of a HAPS-CC, without the JWS envelope). They exist so
an RP implementation can be tested against `factors-v0.1.md` and the FACTOR-RP-\* requirements. Both
bind the same Action Intent as the hashing vectors above.

| Vector | `assurance` | Expected RP outcome |
| --- | --- | --- |
| `consent-credential.webauthn-uv.sample.json` | `HAPS-PoHP-2` / `webauthn.uv` | Illustrative PoHP-2 policy match only if the deployment establishes the claimed agent resistance and all other checks succeed. Reject where policy requires PoHP-3. |
| `consent-credential.otp.sample.json` | `HAPS-PoHP-1` / `otp.delivered` | Illustrative policy match **only** where policy permits agent-accessible factors at PoHP-1 and all other checks succeed. MUST be rejected by any policy requiring agent resistance (FACTOR-RP-03). |

`provider-certification.provider-b.sample.json` is a HAPS-PCC certifying `provider-b` for PoHP-1 and
PoHP-2 with the factors `webauthn.uv`, `passkey.biometric` and `otp.delivered`. It deliberately does
**not** certify any liveness factor or PoHP-3/4, so:

- a PoHP-2 `webauthn.uv` credential from `provider-b` is covered;
- a PoHP-4 or `liveness.active` assertion from `provider-b` MUST fail FACTOR-RP-04 and RP-CORE-06.

The `providerCertification.ref` values are intentionally non-resolvable `.example` URLs. The supplied
PCC names `provider-b`; the WebAuthn sample names `provider-a`, so a harness must supply a matching
fixture for `provider-a` separately. Both CC examples use `issuer="did:web:pp.example"`; no issuer-to-`providerId` trust mapping is supplied.
A harness must provide that mapping separately and cannot infer it from a certification URL.
No provider, certification or signature is validated by these files.
Fixed example timestamps require a controlled test clock and are not usable live approvals.
The current reference kit does not execute these policy examples as RP acceptance tests.

## Negative vectors

`invalid/` holds documents an implementation MUST reject rather than process.

| Vector | Defect | Required behaviour |
| --- | --- | --- |
| `invalid/action-intent.fractional-number.sample.json` | `action.parameters.amount.value` is the JSON number `250.75` | Reject before canonicalization (§6.1, RP-CORE-04a / PP-CORE-02a). An implementation that canonicalizes and hashes it is non-conformant, even though the hash it computes may look stable. |

This fixture also lacks the schema-required `constraints` object. Rejection by a schema validator alone
therefore does not isolate fractional-number enforcement. The reference kit sends it directly to the
JSON canonicalization adapter; an RP test needs an otherwise-valid intent containing the forbidden
numeric syntax to establish that particular rejection behavior. The schema itself cannot enforce
lexical restrictions such as rejecting exponent notation.

Monetary amounts are strings in HAPS (`"250.00"`), so the correct form of this document is the payment
vector above. The restriction simplifies serialization and its verification targets; it does not establish
canonicalizer correctness.

## Signatures

None of these vectors are signed. Signature verification (RP-CORE-01) needs a key pair and is
implementation-specific. These files provide canonicalization/hash cases and illustrative binding and
policy claims; they do not test a complete signed credential validation path.

## Signing View derivation

These vectors do **not** establish that an implementation shows the human the right thing. The Signing
View is supplied already derived, and VEC-01 only checks that hashing it reproduces
`expected_presentation_hash.txt`. The transformation from a proposed action to the view actually
displayed — PP-CORE-02 and PP-CORE-03, and P4 in
[verifiable-properties-v0.1.md](https://github.com/iProov/HAPS/blob/main/specification/draft/verifiable-properties-v0.1.md) — is not
exercised by any vector here. An implementation can pass every vector in this directory and still
render a view that misrepresents the action. Closing the derivation gap needs an explicit executable
derivation check against an expected view, which the current suite lacks. Even that would not verify
the actual UI or the human decision; those need separate evaluation.

## Digital Credentials evidence example

`haps-digital-credentials-portrait-liveness.sample.json` is an illustrative evidence shape, not a
cryptographic test vector. Its short `sha256:` strings are placeholders, not complete SHA-256 digests
or published hash expectations; no underlying credential, portrait or attestation bytes are supplied.
The evidence schema accepts a broad digest pattern and does not enforce all mode-specific liveness
requirements. Passing its schema cannot establish digest correctness, signature validity, freshness,
identity, liveness or approval. Keep this distinction when adapting the example.
