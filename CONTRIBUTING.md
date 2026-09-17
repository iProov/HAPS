# Contributing

Independent implementations and review are the point of publishing this. Conformance is defined by the
specification and the test vectors, not by any codebase — including ours.

## What is most useful

- Review of the security argument, especially attacks the conformance requirements would permit.
- Test vectors, particularly negative ones, and cases the current set misses.
- Extensions to the [symbolic models](formal/tamarin/README.md), or a falsifying trace against them.
- Clarity fixes where two implementers could read a requirement differently.
- Independent implementations in any language, and reports of where the document was ambiguous.

## Workflow

1. Open an issue describing the change and its motivation before a large PR.
2. Keep diffs small and reviewable.
3. For a normative change, include:
   - a requirement ID in [conformance-v0.4.md](specification/draft/conformance-v0.4.md), and
   - at least one positive and one negative test vector.

## If you change a vector or an identifier

Identifiers appear inside the objects that get hashed, so renaming one changes the published hashes.
If a change touches `test_vectors/`, `schemas/`, or any `haps.*` / `urn:haps:*` identifier:

- derive the affected expected values from RFC 8785 and the draft requirements; use the reference
  implementation as one check and, where possible, cross-check with an independent implementation;
- update every vector that carries those hashes as literals (the consent-credential vectors do);
- state in the PR which hashes moved and why. A failing implementation's output alone is not an
  independent oracle and must not be copied into an expected value merely to make its test pass.

A PR that edits a hash without saying why will be asked the question anyway.

## House rules on claims

This project has a standing preference for claiming less. If a change would strengthen a claim about
what is proved, tested or guaranteed, it must be accompanied by the evidence — otherwise reduce the
claim instead. Do not describe any part of this work as "formally verified" or "production-ready".

## Code of Conduct and licensing

Participation is governed by [CODE_OF_CONDUCT.md](CODE_OF_CONDUCT.md). By contributing you agree your
contributions are licensed under this repository's licence (Apache-2.0; see [LICENSE](LICENSE)) on the
terms in its Section 5. Note the scope limits in [NOTICE](NOTICE).
