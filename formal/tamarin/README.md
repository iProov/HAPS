# Symbolic models — HAPS Challenge Mode

HAPS is an experimental draft specification with a partial reference implementation. It is not
production-ready. These Tamarin models examine selected symbolic message-flow properties of
[Challenge Mode and RP verification](../../specification/draft/haps-v0.4.0.md). Their results do not
establish full protocol conformance or that a person saw and approved the correct action.

## What the events mean

`HumanApproved` is a model event, not an observation of a human. The `Human_Approve` rule emits it upon
receiving a network `SHOW` message. The attacker can construct or replay that input. The rule does not
require a preceding `Showed` event, faithful rendering, physical presence, comprehension, or a real
approval gesture. In the co-signature variant, the same network input also causes an abstract holder
key to sign; there is no passkey or authenticator user-interface ceremony in the model.

`view/1` is a public free constructor, without equations: different intent terms yield different
symbolic view terms by assumption. It does not implement Signing View projection, JSON
canonicalization, formatting, localization, truncation, accessibility, or pixel rendering. `Accept`
records symbolic acceptance, not execution of a real action. Correspondence between these events
therefore does not establish correspondence between an actual display, a person's decision, and an
executed action. Human approval and security are intended protocol outcomes subject to implementation
and deployment conditions beyond these models.

## Threat model and assumptions

The Dolev–Yao attacker controls network messages and can intercept, replay, reorder, and construct
messages from known terms across concurrent sessions. Cryptography uses Tamarin's symbolic signing
and hashing abstractions; key secrecy and the absence of collisions hold according to that algebra,
not measurements or implementation testing.

Provider compromise is an explicit rule that releases a signing key. The co-signature variant also
has a holder-key compromise rule. Read each lemma's exact condition: an exclusion such as
`not (Ex #r. Compromised(pp) @ r)` excludes compromise anywhere in the entire trace, not just before
acceptance. Registration and RP trust are abstract facts; the models do not implement enrollment,
account authorization, issuer discovery, certification, or key lifecycle policy.

A fresh challenge and a linear `RP_Outstanding` fact represent one-time consumption in the base and
co-signature models. This assumes the storage behavior that a real implementation must supply. It
does not test database atomicity, distributed races, crash recovery, or an RP's replay cache.

## Recorded local model results

A local run on 2026-09-09 used Tamarin 1.12.0 and Maude 3.5.1 with the commands below. The tables record
results for these particular formulas and model assumptions. “Formula established” corresponds to
Tamarin's `verified` status for an `all-traces` lemma; “witness found” corresponds to `verified` for an
`exists-trace` query. A witness can be an attack or compromise trace, so that status is not itself a
security success. These results do not verify the specification or reference implementation.

### Base model: six lemmas

| Lemma | Formula scope | Recorded result |
| --- | --- | --- |
| `executable` | Some `Accept` trace exists without provider compromise. This checks reachability, not model adequacy. | Witness found |
| `action_binding` | Without provider compromise throughout the trace, `Accept` has an earlier `HumanApproved` event for the same holder, provider, challenge, and `view(intent)`. | Formula established |
| `injective_agreement` | Two identical `Accept` tuples have the same event position if a matching `HumanApproved` exists and the provider is uncompromised. Approval existence is a separate condition in `action_binding`. | Formula established |
| `no_approval_reuse` | Given a `HumanApproved` event, acceptances sharing provider, holder, and challenge have the same position, under the provider compromise exclusion. | Formula established |
| `no_forgery_without_compromise` | Without provider compromise, an acceptance has some matching `HumanApproved` event; this formula does not require the same holder or an earlier event time. | Formula established |
| `compromised_provider_forges` | A trace contains provider compromise and acceptance without a matching `HumanApproved` event. | Witness found |

The compromise witness documents a limitation of this model's trust assumptions. It does not establish
that key compromise is the only way a real implementation can fail, or that other deployment risks
have been addressed.

### Co-signature variant: four lemmas

[`variants/cosigned-holder-key.spthy`](variants/cosigned-holder-key.spthy) adds an abstract holder
signature and an RP check alongside the provider signature. It is an exploratory model variant, not
an implemented passkey flow or evidence of a PoHP level.

| Lemma | Formula scope | Recorded result |
| --- | --- | --- |
| `executable` | Some `Accept` trace exists without provider or holder compromise. | Witness found |
| `action_binding_survives_provider_compromise` | `Accept` has an earlier matching `HumanApproved` event if the holder key is uncompromised throughout the trace; provider compromise is allowed. | Formula established |
| `holder_key_compromise_forges` | A trace has holder compromise and acceptance without a matching `HumanApproved` event. Provider compromise is also allowed. | Witness found |
| `no_approval_reuse` | Acceptances sharing provider, holder, and challenge have the same position when an associated `HumanApproved` event exists. Neither compromise event is excluded. | Formula established |

`holder_key_compromise_forges` does **not** exclude provider compromise. Its witness therefore does
not establish that holder compromise alone is sufficient, or that co-signing simply transfers all
trust from provider to holder. Correct enrollment, key use, display, and human interaction remain
outside the model.

### Negative controls: five lemmas each

These variants deliberately change model behavior to check sensitivity of selected formulas.

| Lemma | [NC1: challenge not consumed](negative-controls/nc1-challenge-not-consumed.spthy) | [NC2: changed intent/view binding](negative-controls/nc2-no-presentation-hash.spthy) |
| --- | --- | --- |
| `executable` | Witness found | Witness found |
| `action_binding` | Formula established | Counterexample found |
| `injective_agreement` | Counterexample found | Formula established |
| `no_approval_reuse` | Counterexample found | Formula established |
| `compromised_provider_forges` | Witness found | Witness found |

NC1 makes `RP_Outstanding` persistent, permitting repeated acceptance. NC2 both omits the presentation
hash **and** permits issuance for a separately supplied network intent instead of the pending shown
intent. Its counterexample cannot be attributed to removal of the hash alone. Unlike the base
model's similarly named lemma, both controls' `compromised_provider_forges` formulas omit an explicit
compromise requirement. Inspect each formula and trace instead of interpreting the name as its
statement. These controls test sensitivity within the abstraction; they do not validate its
correspondence to real deployments.

## Further limits

- Only selected Challenge Mode behavior is represented. The `jti` path without an RP-issued challenge
  and its replay cache are outside the models.
- There is no time model: expiry, `nbf`, freshness windows, deadlines, and clock skew are omitted.
  An unconsumed stale challenge is not rejected on age.
- Identity binding, account matching, factor semantics, liveness, biometric quality, coercion,
  accessibility, and compromised displays or operating systems are not represented. See the
  separate [factor draft](../../specification/draft/factors-v0.1.md) for intended factor requirements.
- Keys are symbolic terms. Registration can repeat; key rotation, revocation, certification,
  enrollment integrity, authenticator recovery, and side channels are not modelled.
- Real cryptographic libraries, serialization, canonicalization, parsers, application code, action
  execution, and operational failures are not checked. There is no implementation-to-model
  refinement proof or computational security bound here.

## Reproducing from the repository root

With `tamarin-prover` and `maude` installed and available on `PATH`, run these commands serially:

```sh
tamarin-prover --version
mkdir -p /tmp/haps-tamarin-review
tamarin-prover --prove formal/tamarin/haps-challenge-mode.spthy \
  > /tmp/haps-tamarin-review/base.log 2>&1
tamarin-prover --prove formal/tamarin/variants/cosigned-holder-key.spthy \
  > /tmp/haps-tamarin-review/cosigned.log 2>&1
tamarin-prover --prove formal/tamarin/negative-controls/nc1-challenge-not-consumed.spthy \
  > /tmp/haps-tamarin-review/nc1.log 2>&1
tamarin-prover --prove formal/tamarin/negative-controls/nc2-no-presentation-hash.spthy \
  > /tmp/haps-tamarin-review/nc2.log 2>&1
```

Inspect each log's lemma summary and any counterexample or witness trace. A zero process exit code
does not mean every safety lemma holds: the negative controls also exited zero in this run. Retain
the source revision, tool versions, commands, and output when reporting a new result, and rerun after
model changes. Do not describe a tool result as full protocol verification or proof of real human
approval.
