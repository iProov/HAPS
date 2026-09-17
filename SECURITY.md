# Security policy

HAPS is an **experimental draft specification with a partial reference implementation**. It is not
production-ready. See [docs/limitations.md](docs/limitations.md) for what the specification, vectors
and proof experiments do and do not establish.

## Reporting a vulnerability

Email **security@iproov.com**, marking the subject line `HAPS specification`. This address is
monitored and works regardless of repository visibility or GitHub account status. It is the primary
route, and the one to use if anything below is unavailable.

You may instead use GitHub's
[private vulnerability reporting form](https://github.com/iProov/HAPS/security/advisories/new)
(**Security → Report a vulnerability**) when it is available. That form requires a public repository
with private vulnerability reporting enabled; maintainers enable and monitor it at publication.
Reports submitted through it are private.

Please do not open a public issue for a suspected vulnerability, and please do not report one through a
pull request.

We aim to acknowledge a report within five working days. If you receive no acknowledgement, please
resend rather than assuming the report was judged out of scope.

Useful things to include: which document, schema, vector or requirement is affected; the assumption you
believe is broken; and a concrete trace or worked example if you have one. For the symbolic models, a
falsifying trace from Tamarin is ideal.

We will acknowledge a report and tell you whether we consider it in scope. We do not operate a bounty.

## What counts as a vulnerability here

This repository is a specification, so the interesting failures are design-level:

- A way to satisfy the requirements while breaking the binding between an approval and the action it
  approved — the property the whole design exists to provide.
- A replay, substitution or confusion attack that the conformance requirements permit.
- A gap where an implementation can pass every published vector and still be unsafe.
- An assumption stated in the [Tamarin models](formal/tamarin/README.md) that does not hold in a
  realistic deployment, or a lemma that is vacuous.
- An error in the canonicalization rules that allows two distinct parsed values to produce the same
  hash, or one value to produce two hashes.

Known limitations are recorded in
[docs/limitations.md](docs/limitations.md) and in
[verifiable-properties-v0.1.md](specification/draft/verifiable-properties-v0.1.md). A report that the
Signing View derivation is untested, or that the Lean proofs cover only a comparison step, is
already documented. Concrete security consequences, missing limitations and misleading claims are
still welcome reports; documenting a gap does not make its consequences harmless.

## Reference implementation

Report issues in the Rust code, the Kani harnesses or the Lean proofs against
[iProov/HAPS-reference](https://github.com/iProov/HAPS-reference) using the same private form there.
