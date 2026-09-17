# Reference implementation: status and verification plan

**Status:** Experimental, partial implementation; not production-ready. This plan is informative and is
not part of the normative specification. **Repository:** [iProov/HAPS-reference](https://github.com/iProov/HAPS-reference).

Passing the supplied tests does not establish full protocol conformance or that a person saw and
approved the correct action. See [limitations](limitations.md). The items below distinguish implemented
components from future work; a proposed proof is not an achieved result.

## 1. Current implementation

| Component | Current scope |
| --- | --- |
| `haps-canon` | Rust strict JSON parsing and canonicalization for the draft's integer-only numeric domain; `no_std` with `alloc`, using allocated values and buffers |
| `haps-hash` | SHA-256 using `sha2`, plus the `sha256:` base64url encoding; the hash primitive is outside the proof scope |
| `haps-reference-adapter` | NDJSON test interface for parsing, canonicalization and hashing |
| `conformance/` and interoperability plugin | Python runner and portable kit, plus a Python example adapter; selected canonicalization/hash checks |
| `verification/` | Extracted ordering-module Lean definitions and two comparison-step theorem proofs |

The primary implementation is Rust. The runner and example adapter are Python; implementations in other
languages are welcome. Conformance is defined by the [draft requirements](../specification/draft/conformance-v0.4.md),
not by matching the Rust code.

`haps-core`, `haps-verify`, `haps-jose`, `haps-store` and `hapsd` are proposed components, not available
crates. The current code does not provide the types/policy verifier, credential signature verification,
replay store, HTTP provider, consent UI or MCP service contemplated by this plan.

## 2. Current verification evidence

The [property catalogue](../specification/draft/verifiable-properties-v0.1.md) is a target, not a list of
completed proofs. The reference repository contains:

- Four Kani harnesses: ordering antisymmetry on fixed two-byte inputs, ordering transitivity on fixed
  one-byte inputs, the base64url alphabet over `u8`, and integer formatting bounded to ±10⁶. These are
  local checks, not parser verification, full ordering proofs or crate-wide panic freedom.
- Two Lean proofs about the extracted comparison loop body, conditional on a successful final verdict
  from that step. They do not establish loop termination or whole-string ordering. Aeneas/Charon
  translation fidelity and toolchain correspondence remain assumptions.
- Unit and vector tests and a portable suite with finite examples. These do not cover a complete
  protocol flow, Signing View derivation/display or human approval.

CI invokes tests, linting, kit consistency checks and the bounded Kani harnesses. It does not run
Charon/Aeneas extraction or Lean checking. The Lean project depends on an external Aeneas checkout; see
the reference repository's [toolchain instructions](https://github.com/iProov/HAPS-reference/blob/main/docs/verification-toolchain.md)
for prerequisites and reproducibility limits. This document does not claim a fresh proof run.

## 3. Proposed next work

Possible future work includes whole-string ordering, parser and serialization properties, Signing View
derivation, binding/freshness/policy verification, and abstract replay-state properties with an explicit
contract for concrete storage. These require implementations, precise proof statements and reproducible
checks before any completion claim can be made. CI extraction and Lean checking are also future work.

The integer-only numeric restriction simplifies serialization obligations. It does not itself prove P1
or P2, or remove the need to check integer boundaries, Unicode handling, parsing and resource limits.

Coding restrictions such as avoiding `unsafe`, explicit panics and unchecked indexing reduce some risks.
Lint success does not establish termination, memory availability, absence of every panic or correctness
of the protocol. Kani establishes only the assertions and bounds in each completed harness; Lean checks
only the theorem statements and assumptions actually supplied.

## 4. Trust and deployment boundary

A future verifier argument would still depend on correct cryptography and key trust, accepted input
validation, a trustworthy clock, persistent atomic replay consumption coupled to execution, and faithful
translation from Rust into its proof model. The current parser has tests; it is not proved. The current
hash dependency is not verified by this project. No signature verifier or concrete replay store is
implemented here.

Faithful rendering, human presence, explicit approval, authorization, identity enrollment, provider
honesty and deployment security cannot be inferred from these code properties. They require separate
implementation and evaluation evidence.

## 5. Relationship to the protocol model

The specification repository contains a [Tamarin symbolic model](../formal/tamarin/README.md), not a Lean
protocol model. It represents idealized message flow and assumes symbolic cryptography, deterministic
views and atomic approval events. The reference repository's Lean files concern extracted code. No
refinement proof connects the Tamarin model, the complete specification and the Rust implementation.

Public descriptions should identify the exact artifact, revision, theorem or harness, input bounds,
assumptions and reproduction steps. None of this work supports a claim that all protocol properties are
verified, that the code cannot fail, or that fraud is prevented.
