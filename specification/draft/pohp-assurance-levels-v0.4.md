# HAPS PoHP Assurance Levels v0.4 (Draft)

**Status:** Experimental draft; not production-ready.
**Applies to:** HAPS v0.4.x
**History:** Replaces the v0.3 assurance terminology; older drafts are not included in this bundle.

PoHP levels describe intended **attack resistance** and **verification rigor** requirements for
presence evidence under an evaluated threat model. They are claims to substantiate, not guarantees
created by including a level string in a credential. Factor classes are registered in
[factors-v0.1.md](factors-v0.1.md).

HAPS has a partial reference implementation. Passing the supplied tests does not establish full
protocol conformance, any assurance level, or that a person saw and approved the correct action.
This draft supplies no operational certification scheme or evaluated factor implementations.

## 1. Properties

A PoHP claim is characterised by which of the following its evidence supports. Resistance claims
need stated attacks, evaluation scope, and implementation and deployment assumptions.

| Property | Meaning |
| --- | --- |
| **B — Action binding** | The reported approval event is bound to an Action Intent and the Signing View the PP is required to display (`intent_hash`, `presentation_hash`, `aud`, freshness, single use). Matching hashes do not establish actual display. |
| **H — Human actuation** | Evidence supports resistance to software-only completion under the assessed threat model, including access to the user's device, session, and credentials. See [agent resistance](factors-v0.1.md#2-agent-resistance). |
| **P — Presentation-attack resistance** | Resistant to replayed or presented artefacts (photograph, mask, recording, cloned card surface). |
| **I — Injection resistance** | Resistant to synthetic or injected input that bypasses the capture path (deepfake, virtual camera, emulated authenticator, relayed signature request). |
| **K — Key or hardware binding** | The result is bound to hardware or key material with evaluated protection against export or unauthorized use. Non-exportability alone does not establish human actuation or prevent software-requested signatures. |
| **C — Certification** | The provider holds a current [HAPS-PCC](../../schemas/provider-certification-credential.v0.4.schema.json) for the level and factor class asserted, under the trusted evaluator's stated evaluation, re-certification, and audit scope. |

## 2. Levels

| Level | Required properties | Intended use |
| --- | --- | --- |
| **HAPS-PoHP-1** | B | Intended for low-risk, reversible, low-value actions where RP policy accepts a reported bound acknowledgement. Carries **no** claim that a human, rather than an agent, produced it. |
| **HAPS-PoHP-2** | B, H, P | Intended to support consequential actions when evaluated human-actuation and presentation-attack resistance meet RP policy. |
| **HAPS-PoHP-3** | B, H, P, I, K | Intended to support higher-risk actions when injection resistance and hardware/key binding are also substantiated. |
| **HAPS-PoHP-4** | B, H, P, I, K, C, plus audit | Intended to support additional certification and audit requirements; does not establish regulatory compliance or suitability for an irreversible action. |

An RP MUST enforce a minimum level per endpoint policy. Because level and factor class are orthogonal,
an RP SHOULD also constrain accepted factor classes; a minimum level alone does not tell an RP whether
the approval could have been produced by an agent (see §4).

## 3. Where factors land

Illustrative candidates only; these are not assigned or certified levels. Actual evidence must
substantiate every required property. UV, attestation, a signature, or a liveness label alone does not
determine a level.

| Factor (`assurance.method`) | Candidate level | Evidence still needed |
| --- | --- | --- |
| `acknowledge` | 1 | no H |
| `otp.delivered` | 1 | no H — agent-retrievable |
| `push.approval` | 1 | no H — device/session acceptance is insufficient |
| `webauthn.uv` | 1–2 | H and P depend on actual UV method and agent access; PIN UV may be agent-accessible |
| `passkey.biometric` | 2–3 | H/P and, for level 3, I/K across biometric, fallback, authenticator, and session paths |
| `liveness.passive` | 2 | H/P and capture integrity; no automatic I claim |
| `liveness.active` | 2–3 | H/P and, for level 3, I/K; active interaction alone supplies neither |
| `smartcard.signature` | 2–3 | H/P and, for level 3, I/K; key use alone does not establish a human action |
| `hardware.actuated` | 2–3 | H/P and, for level 3, I/K in the assessed device and deployment |
| factor implementations meeting B, H, P, I, K | 4 | current certification (C) and required audit; certification cannot upgrade an agent-accessible factor |

## 4. Certification and self-assertion

Without a verifiable HAPS-PCC, a claimed level is **self-asserted by whoever holds the signing key**. A
provider advertising a level MUST supply certification evidence, and an RP requiring PoHP-4 MUST verify
it. An RP that cannot verify a claimed level SHOULD treat that level as unattested. At most, an event
with substantiated binding may be classified as PoHP-1 for analysis; this is not permission to accept
it. Failed mandatory certification checks or an unmet minimum level require rejection under
[core §12](haps-v0.4.0.md#12-verification-requirements-rp); silently downgrading the requested policy
would bypass that requirement.

Level and certification are separate assertions. Checking a certification signature establishes who
issued it under the configured trust policy; it does not independently reproduce the evaluator's
work or establish that the current deployment still meets the evaluated conditions.

Assurance certification is also distinct from conformance with the protocol specification; the two are
separated, with requirements, in [core §16](haps-v0.4.0.md#16-two-kinds-of-certification). Passing the
supplied vectors is not evidence of **H**, **P**, or **I**, or a faithful displayed view. Compromised
providers or devices, coercion, misleading UI, and execution that differs from the validated intent
remain deployment risks; no level is a promise that fraud cannot occur.

## 5. Changes from v0.3

v0.3 defined PoHP-2 as "verified liveness (passive PAD)" and PoHP-3 as "deepfake-resistant liveness
(active/hybrid)". That made one modality the definition of assurance: a smart-card signature, a
physically actuated secure element, or a biometric passkey could not be placed above PoHP-1 on merit,
however agent-resistant it was in practice.

v0.4 keeps the four levels and their intent, restating them as properties (§1) so that:

- passive and active liveness remain candidate factors for levels 2 and 3, respectively, subject to
  evidence for every required property; earlier technology labels do not establish those properties;
- non-liveness factors are placeable;
- **H (human actuation)** becomes explicit, separating agent-resistant from agent-accessible factors;
- **C (certification)** becomes a stated property of PoHP-4 rather than a general exhortation.

The four-level structure is retained. The historical `AAIF-PoHP-2` identifier became `HAPS-PoHP-2`
as part of the [v0.4 namespace migration](haps-v0.4.0.md#changes-in-v040), a breaking wire change.
Historical labels do not automatically substantiate the explicit v0.4 property requirements.
