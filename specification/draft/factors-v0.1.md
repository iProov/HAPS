# HAPS Human-Presence Factors v0.1 (Draft)

**Status:** Experimental draft; not production-ready.
**Applies to:** HAPS v0.4.x

HAPS has a partial reference implementation. Passing the supplied tests does not establish full
protocol conformance, the resistance of a factor, or that a person saw and approved the correct action.

## 1. Abstract

HAPS does not define how to detect human participation. It specifies how evidence of a claimed
participation event is **bound** to a specific Action Intent and **checked** at the relying party
boundary. A factor may support a HAPS step-up when its implementation, deployment, and evaluated
properties satisfy the RP's policy.

This document registers the factor classes HAPS composes with, the `assurance.method` values that name
them on the wire, and — the property that matters most for a human-approval protocol — whether each
factor is **agent-resistant** or **agent-accessible**.

## 2. Agent resistance

A factor is **agent-resistant** when evidence supports resistance to software-only completion under
an explicitly stated threat model, including the agent's access to the user's device, session, and
credentials. A factor is **agent-accessible** when an agent with that access can complete it.
These are conditional assessments of an implementation and its deployment, not universal properties
of a technology name. Neither classification establishes that the human saw or approved the action.

An RP may choose agent-accessible factors for low-risk, reversible, low-value actions. They MUST NOT be
presented as evidence of human presence for consequential actions. An RP MUST be able to require
agent-resistant factors for a given endpoint, independently of the PoHP level claimed.

## 3. Factor registry

The resistance column identifies a candidate classification only. An RP needs evidence for the
actual device, verification method, capture path, and deployment; the method string is not evidence.

| `assurance.method` | Factor class | Agent resistance | Conditions and limitations |
| --- | --- | --- | --- |
| `hardware.actuated` | Physically actuated secure hardware | conditional | Requires an evaluated physical actuation path; a secure element or non-exportable key alone does not establish human actuation or a correct display. |
| `smartcard.signature` | Smart-card signature or human-held signing key | conditional | A signature establishes use of a key under cryptographic assumptions. Human actuation depends on reader, PIN handling, and per-operation controls; software may otherwise request signatures. |
| `passkey.biometric` | Biometric passkey | conditional | Requires evidence of the actual verification method and enforced UV. Biometric fallback, device compromise, and authenticator behavior affect the assessment. |
| `webauthn.uv` | WebAuthn with user verification | conditional | UV alone does not establish agent resistance: an agent may have access to a PIN or another accepted verification path. |
| `liveness.passive` | Passive-PAD liveness | conditional | Resistance depends on evaluated attacks and capture integrity; the label does not establish resistance to injection or all presentation attacks. |
| `liveness.active` | Active or hybrid liveness | conditional | Active interaction alone does not establish injection or deepfake resistance; the complete capture and decision path needs evaluation. |
| `otp.delivered` | One-time passcode | **agent-accessible** | An agent with mailbox, SMS, or device access can retrieve it. Low-risk use only. |
| `push.approval` | Push approval / tap-to-approve | **agent-accessible** | Records an acceptance event from a device or session; may be automated and does not establish human presence. |
| `acknowledge` | Explicit acknowledgement | **agent-accessible** | Records an acknowledgement reported by the UI; does not establish faithful display, human presence, or informed intent. |

Values are extensible. An implementation MAY emit a method not listed here; an RP MUST treat an
unrecognised method as agent-accessible and unclassified unless its policy states otherwise.

## 4. Relationship to PoHP levels

Factor class and assurance level are separate claims. The factor names *what kind of evidence* was
reported; the PoHP level names the intended resistance requirements in
[PoHP assurance levels](pohp-assurance-levels-v0.4.md). A single factor class can span levels — a passkey with enforced UV on
certified hardware is not the same evidence as one with UV requested and unverified.

An RP policy therefore has two independent knobs, and needs both:

- minimum PoHP level, and
- accepted factor classes (or, minimally, "agent-resistant only").

A self-asserted PoHP level alone does not establish that the event resisted agent completion.

## 5. Binding requirements

Regardless of factor class, the Consent Credential MUST bind the approval to the action:

- `intent_hash` over the canonical Action Intent,
- `presentation_hash` over the Signing View the PP is required to display,
- `aud` naming the relying party,
- `challengeId` when the RP issued a HAPS-CHAL,
- a freshness window, and single-use enforcement via `jti`.

A factor with no binding is an authentication event, not a HAPS approval. Strength of factor never
substitutes for binding.

These checks are intended to support action binding when the PP renders the correct view, collects
the intended approval event, protects its signing key, and the RP validates the evidence and enforces
execution and replay policy. A hash or signature cannot independently establish actual screen output,
human understanding, absence of coercion, or prevention of fraud.

## 6. Certification

`providerCertification` carries the provider's HAPS-PCC. Per [PoHP assurance levels](pohp-assurance-levels-v0.4.md), a provider
advertising a PoHP level MUST be certified for it, and the certification names both the levels and the
factor classes the provider is certified to deliver. An RP that cannot verify the certification of a
claimed level and factor SHOULD treat the level as unattested. That classification does not permit
acceptance when a required level or certification check fails. The draft defines a certification
format, not an operating certification scheme or an evaluation of these factor classes.

## 7. Non-goals

HAPS does not specify biometric algorithms, PAD test methodology, authenticator attestation formats,
OTP delivery, or push transport. It specifies how reported outcomes are named, bound, and checked.
