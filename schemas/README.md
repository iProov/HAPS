# Loading the draft schemas

These JSON Schemas describe experimental draft object shapes. Schema validation does not establish
protocol conformance, cryptographic validity, human presence or approval. See [limitations](../docs/limitations.md).

Load the schemas into your validator's local registry under their exact `$id` values before validating
an object. The identifiers are URNs, not URLs to fetch. In particular, the challenge schema references
`urn:haps:schema:action-intent:v0.4`, supplied by `action-intent.v0.4.schema.json` in this directory.
A relative filename cannot be resolved against a URN base as a sibling web or filesystem path.

Use JSON Schema Draft 2020-12 and explicitly enable `format` checks if your validator treats them as
annotations by default. Additional protocol checks remain necessary, including duplicate-key and
numeric-lexeme rejection before parsing loses that information, trusted signature verification,
freshness, replay consumption, assurance policy and faithful Signing View derivation/display.
