# HAPS UI Confirmation Mapping — `ai_ui_confirm_v1` (Optional)

**Status:** Experimental draft mapping; not production-ready. HAPS has a partial reference
implementation. Passing the supplied tests does not establish full protocol conformance or that a
person saw and approved the correct action.

If your AG-UI Mobile Profile (or any other UI profile) integrates with HAPS, it SHOULD target the general HAPS AI-INTENT profile `ai_ui_confirm_v1`.

`ai_ui_confirm_v1` is the identifier used by this draft; `v1` names the optional profile revision,
not the HAPS core version. This annex does not define a `haps.profile.*` alias. Profile strings are
included in canonical intent data, so silently renaming one changes the intent hash.

## What changes vs the earlier mobile-specific mapping?
- The HAPS profile is UI-agnostic; your UI protocol only needs to map its confirm/approval primitive into the AI-INTENT `display` block.
- The mapping is intended for confirmation UIs on mobile, desktop, web, or embedded surfaces; each
  platform still needs its own display, accessibility, and approval-path evaluation.

## Minimal mapping guidance

Under the [Action Intent v0.4 schema](../../../schemas/action-intent.v0.4.schema.json), `display` is
a closed object (`additionalProperties: false`) with three required members and one optional member:

- confirmation title/heading → `display.title` (≤ 120 characters)
- what the action does, including its effects, as one human-readable statement → `display.summary` (≤ 600 characters)
- risk level, undo and mitigation wording → optional `display.riskNotice` (≤ 600 characters)
- locale → `display.language`

There is no top-level `ui` block, and `display` has no member for an ordered effects list, a separate
action label, permissions/scopes, a structured reversibility field, or a timezone. Do not add those
members to `display` or invent new top-level AI-INTENT members. Business parameters belong in
`action.parameters`, and applicable constraints belong in `constraints` under the schema and profile
rules. Platform details such as notifications, deep links, and handoff stay in the client protocol.

## Limits and unresolved profile detail

This annex supplies display-field mapping guidance, not a complete executable Signing View derivation.
Follow [core §7](../haps-v0.4.0.md#7-signing-view-and-presentation_hash-wysiwys), including the generic
rules when a profile is absent or unknown. Implementations need to agree on a complete deterministic
derivation and evaluate whether the resulting display faithfully represents the action. A title or
summary alone may omit material parameters or effects.

A matching `presentation_hash` checks the represented Signing View data. It does not establish what
pixels were rendered, whether the view was visible, whether a person read it, or whether approval was
free and informed. The mapping does not add those guarantees to a UI protocol.
