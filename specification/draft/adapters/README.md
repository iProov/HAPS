# Identity binding adapters

**Status:** Experimental draft specifications; not production-ready. The separate
[HAPS-reference](https://github.com/iProov/HAPS-reference) is a partial reference implementation;
listing an adapter here does not establish that it is implemented or evaluated. Passing the supplied
tests does not establish full protocol conformance or that a person saw and approved the correct action.

Identity binding in HAPS is pluggable. The [core specification](../haps-v0.4.0.md#9-identity-binding-pluggable-adapters) defines the binding
block, the policy knob, and what any adapter must specify. Each adapter below is the single normative
source for its own scheme.

Adapters are **optional**. A relying party that does not require identity binding claims none of them,
and HAPS core does not favour any identity provider.

| Adapter | Scheme / profile id | Purpose |
| --- | --- | --- |
| [entra-oidc-v0.1.md](entra-oidc-v0.1.md) | `entra_oidc` | Binds consent to a Microsoft Entra ID enterprise identity over OIDC, normalizing the subject as tenant id + object id. |
| [entra-claims-challenge-v0.1.md](entra-claims-challenge-v0.1.md) | `haps.profile.entra-claims-challenge/v0.1` | Transport profile carrying Entra claims-challenge semantics through the identity flow for enterprise step-up. |
| [digital-credentials-portrait-liveness-v0.1.md](digital-credentials-portrait-liveness-v0.1.md) | `haps.profile.digital-credentials-portrait-liveness/v0.1` | Describes evidence for a step-up using a wallet credential presentation, optionally combined with an independent portrait/liveness result, subject to RP policy. |

## Writing a new adapter

Per [core §9.3](../haps-v0.4.0.md#93-adapter-requirements), an adapter specification MUST define its scheme identifier, the normalized
subject it emits, the claims it validates and with what clock skew, how it binds its freshness value to
the HAPS approval session, what evidence an RP can re-verify, and its scheme-specific policy keys.

Per [core §9.4](../haps-v0.4.0.md#94-authentication-step-up-is-not-approval), an adapter supplies identity evidence under its issuer and session-binding assumptions; it does not establish that a human approved. Authentication step-up carries
no implication of human actuation, and several common enterprise step-up factors are classified as
agent-accessible in [the factor registry](../factors-v0.1.md). Presence requirements belong on `assurance`, not on identity
policy.

A signed identity assertion, wallet authentication, or liveness result alone does not demonstrate a
faithful display, informed approval, authority to act, or freedom from fraud. Those depend on the
approval UI, factor implementation, trusted issuers, session binding, and RP enforcement.
