# HAPS Optional Transport Profile: Microsoft Entra Claims Challenge (`entra_claims_challenge`) v0.1

**Status:** Experimental draft; not production-ready.
**Profile ID:** `haps.profile.entra-claims-challenge/v0.1`  
**Applies to:** HAPS v0.4.x  
**Companion adapter:** [`entra_oidc`](entra-oidc-v0.1.md)

HAPS has a partial reference implementation. Passing the supplied tests does not establish full
protocol conformance, this profile's interoperability, or that a person saw and approved the correct
action.

## 1. Purpose

This profile defines how a Presence Provider (PP) may satisfy a HAPS step-up using **Microsoft Entra claims challenge** semantics. It does not create a new identity binding scheme. Instead, it reuses the existing `entra_oidc` adapter and specifies how a PP carries an Entra `claims` request through the OAuth 2.0 Authorization Code + PKCE flow before issuing a HAPS Consent Credential.

The profile is intended for enterprise environments where the relying party or bridge uses Microsoft
Entra Conditional Access / authentication context and wants the resulting identity evidence to
support the HAPS approval flow. It does not establish the quality of that authentication or approval.

## 2. Inputs

The PP MAY receive Entra claims-challenge requirements through:

- `requirements.identity.policy.entraClaimsChallenge`

Earlier integration guidance also named `requirements.identity.schemeParams.entra_claims_challenge`.
That path is not permitted by the [HAPS-CHAL v0.4 schema](../../../schemas/haps-challenge.v0.4.schema.json).
An MCP implementation may support it as an explicitly documented legacy input, but HAPS-CHAL examples
and new integrations use `policy.entraClaimsChallenge`.

The value MAY be:
- a decoded JSON object suitable for the OAuth `claims` parameter, or
- a raw JSON string already prepared for the `claims` parameter.

If no explicit claims challenge is supplied, the PP MAY derive one from existing identity policy inputs such as:
- `requireMfa`
- `requiredAuthContexts`

## 3. PP behavior

A PP claiming this profile:

1. **MUST** execute the Entra identity-binding flow using Authorization Code + PKCE.
2. **MUST** preserve `state`, `nonce`, and PKCE binding in the same HAPS consent session.
3. **MUST** include the Entra `claims` parameter on the authorization request when an explicit or derived claims challenge is present.
4. **SHOULD** include client capability signaling (`xms_cc=cp1`) when talking to Entra resources that require claims-challenge-capable clients.
5. **MUST NOT** treat the refreshed Entra token as a substitute for HAPS consent. The HAPS Consent Credential remains the approval artifact; the refreshed token is enterprise identity evidence supporting the approval.
6. **SHOULD** record the effective claims request in session debug or evidence metadata for auditability.

Sending the requested claims is not evidence that policy was satisfied: the validated result must
meet the [companion adapter's policy checks](entra-oidc-v0.1.md#34-optional-assurance-signals) before
issuing the credential. Failure, cancellation, or an unmet requirement does not authorize issuance.

## 4. RP behavior

An RP or bridge claiming this profile:

1. **MAY** request an explicit Entra claims challenge using the policy field above.
2. **SHOULD** require embedded evidence if it must directly verify the refreshed Entra identity token.
3. **MUST** continue to verify the resulting HAPS Consent Credential in full, including `intent_hash`, `presentation_hash`, audience, expiry, replay controls, and identity binding.
4. **MUST NOT** accept the presence of a refreshed Entra token as proof of human approval on its own.

## 5. Example identity requirements block

This is the `identity` portion of `requirements`; a complete HAPS-CHAL also requires `pohp`, its
challenge fields, and `actionIntent`. The example requests authentication context for a resource
access token, following Microsoft's [claims-challenge format](https://learn.microsoft.com/en-us/entra/identity-platform/claims-challenge).
An access-token result is evidence for its intended resource and is not an ID token for the HAPS PP;
deployments must specify which validated evidence establishes each requested policy condition.

```json
{
  "identity": {
    "mode": "required",
    "schemes": ["entra_oidc"],
    "policy": {
      "requireEmbeddedEvidence": true,
      "requireMfa": true,
      "requiredAuthContexts": ["c1"],
      "entraClaimsChallenge": {
        "access_token": {
          "acrs": {
            "essential": true,
            "values": ["c1"]
          },
          "xms_cc": {
            "values": ["cp1"]
          }
        }
      }
    }
  }
}
```

## 6. Security notes

- The Entra claims challenge transports enterprise authentication requirements; it does not itself prove those requirements were met.
- Session continuity (`state`, `nonce`, PKCE, `challengeId`) remains mandatory.
- The profile is vendor-specific and optional; it should be claimed only when an implementation actually supports it.
- Identity evidence and the PP's signed approval assertion depend on token validation, issuer policy,
  session binding, and an evaluated approval UI. Neither a refreshed token nor a matching hash proves
  what a person saw, understood, or approved, and neither prevents all fraud.
