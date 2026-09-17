# Experimental status and limitations

HAPS is an **experimental draft specification and partial reference implementation**. It is **not
production-ready**. Passing the supplied tests does not establish full protocol conformance or that a
person saw and approved the correct action. The draft describes intended behavior under implementation
and deployment conditions; neither repository establishes general fraud prevention or failure-free code.

## What is implemented and checked

The [partial reference implementation](https://github.com/iProov/HAPS-reference) supplies Rust JSON
parsing, canonicalization and hashing, a test adapter, and a portable canonicalization/hash test kit.
It does not implement a complete Presence Provider, relying-party credential verifier, consent UI,
Signing View derivation pipeline, identity adapter, signature validation or replay store.

The supplied vectors cover selected canonical bytes, hashes and malformed inputs. Unsigned policy and
identity examples illustrate intended assertions; they are not live evidence or complete verifier tests.
The kit does not execute the factor-policy examples as acceptance tests. It hashes the supplied Signing
View without deriving or displaying it. Its finite checks are not exhaustive properties. The fractional
negative fixture also lacks a required schema field, so a schema-only rejection does not isolate the
numeric restriction.

The reference repository includes four bounded Kani harnesses and two Lean theorem proofs about a
successful terminating comparison step. They do not establish whole-string ordering, parser correctness,
all P1–P18 properties, or crate-wide panic freedom. Lean extraction and checking require external
components and are not CI gates. The [Tamarin model](../formal/tamarin/README.md) analyzes symbolic
Challenge Mode events under explicit assumptions; there is no proof connecting it to the Rust code.

## What credential checks can and cannot establish

- **Display and approval:** `presentation_hash` binds data. A matching hash does not attest to screen
  pixels, layout, truncation, overlays, accessibility output, what a human perceived, or an explicit
  approval event. The provider and UI must faithfully derive and render the view and bind the captured
  decision to it. The Tamarin model assumes an atomic `HumanApproved` event; it does not validate a UI.
- **Action execution:** an RP must compare the credential against the intended action and enforce that
  same action, including envelope constraints, at execution. Data or state changing after verification,
  retries, partial failures and duplicate execution need application-level handling.
- **Presence and identity:** factor names, levels and signed claims are assertions. Their meaning depends
  on the actual authenticator, human interaction, enrollment, evidence verification, evaluator, attack
  model and deployment. Authentication or a biometric match alone does not establish transaction consent.
- **Provider and key trust:** signatures identify use of a key under cryptographic assumptions; they do
  not establish the truth of the signed statements. A dishonest provider or compromised signing key can
  assert an approval that did not occur. Trust, key provisioning, rotation and revocation need deployment
  controls. Certification is scoped evidence, not a promise against every attack.
- **Freshness and replay:** clocks, policy and durable, atomic state across processes and failures are
  required. The current code supplies no replay store or transactionally coupled action executor. The
  symbolic model has no wall-clock expiry or persistence/crash behavior.
- **Authorization and fraud:** the RP must separately establish authority to act. Neither the draft nor
  its checks establish understanding, lack of coercion, honest intent, protection from compromised
  devices, or prevention of all fraud.
- **Runtime and privacy:** the current implementation is not established to be safe for arbitrary hostile
  workloads. Allocation, recursion, resource exhaustion, dependencies and side channels are outside the
  supplied proof coverage. A prohibition on raw biometrics in a credential does not enforce data handling
  by providers, logs, transports or extensible evidence objects.

## Reading the draft and examples

The Generic Profile lists required view content but does not supply a complete canonical object
construction algorithm. The UI annex is mapping guidance, and the payment profile identifier in the
vectors is illustrative. Independent implementations need an agreed deterministic derivation; matching
the supplied view hash does not establish such agreement.

MUST/SHOULD statements specify obligations for an implementation claiming the applicable conformance
class; they are not statements that the reference code implements them. The property catalogue also
contains research targets and strengthened formulations, identified there. JSON Schema validation alone
cannot enforce all semantic, cryptographic, numeric-lexeme, session or display requirements. For example, the Digital
Credentials evidence schema permits short digest placeholders and does not require liveness evidence
conditionally for every liveness mode. The examples and their schema do not form a complete verifier.

Example domains and credentials are illustrative. Example URLs under `.example` are intentionally
non-resolvable; fixed timestamps, unsigned claims and certification payloads are not usable live
credentials. Real integrations need their own trusted keys, policy, current session data and independent
assessment. See [conformance](../specification/draft/conformance-v0.4.md) for the requirements and
[test vectors](../test_vectors/v0.4/README.md) for their limited coverage.
