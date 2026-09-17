# HAPS Identity Binding Adapter: Microsoft Entra OIDC (entra_oidc) v0.1

**Status:** Experimental draft; not production-ready.
**Adapter ID:** `entra_oidc`  
**Applies to:** HAPS v0.4.x

HAPS has a partial reference implementation. This adapter describes intended behavior, not evidence
that an implementation supplies it. Passing the supplied tests does not establish full protocol
conformance or that a person saw and approved the correct action.

## 1. Purpose

This adapter defines how a Presence Provider (PP) binds a HAPS consent event to a **Microsoft Entra ID** enterprise identity using OpenID Connect.

The normalized identity subject for this scheme is:
- `tid` (tenant id)
- `oid` (user object id)

## 2. Inputs (RP policy)

Within `requirements.identity`:

- `mode`: `none | preferred | required`
- `schemes`: includes `"entra_oidc"`
- `policy` keys (recommended):
  - `allowedTenants`: array of tenant ids allowed
  - `requireMfa`: boolean
  - `requiredAuthContexts`: array of auth context ids (e.g., ["c1"])
  - `requireEmbeddedEvidence`: boolean (embed ID token)

## 3. Protocol (PP side)

### 3.1 OIDC flow

Consistent with [ENTRA-PP-01 and ENTRA-PP-03](../conformance-v0.4.md#entra-adapter-requirements-pp--haps-adapter-entra_oidc-01-pp), PP MUST use:
- Authorization Code Flow with PKCE
- `nonce` binding
- `state` binding

The OIDC `nonce` here is a session/transaction freshness value for the identity flow. It MUST NOT be embedded into AI-INTENT business semantics.
PP MUST bind this flow to the same HAPS approval session and output `nonceHash` as required by
ENTRA-PP-03. A hash copied from the token alone is not independent evidence that the token belongs
to the expected session.

Recommended endpoints:
- Authorization endpoint: `https://login.microsoftonline.com/{tenant}/oauth2/v2.0/authorize`
- Token endpoint: `https://login.microsoftonline.com/{tenant}/oauth2/v2.0/token`

### 3.2 Token validation

PP MUST validate:
- JWT signature using the configured permitted algorithms and trusted issuer keys
- `iss` matches expected issuer for the tenant / v2 endpoint
- `aud` matches PP’s Entra client id
- `exp`/`nbf`/`iat`
- `nonce` matches the authorization request
- `tid` and `oid` are present for subject normalization; `tid` satisfies `allowedTenants` when supplied

PP MUST apply bounded clock skew when checking `exp`/`nbf`/`iat` (RECOMMENDED <= 300 seconds).

### 3.3 Subject normalization

PP MUST output the normalized subject in `identityBinding`. This illustrative fragment uses
placeholders, not a verified identity:

```json
{
  "identityBinding": {
    "mode": "verified",
    "scheme": "entra_oidc",
    "idp": {
      "issuer": "https://login.microsoftonline.com/<tid>/v2.0",
      "tenantId": "<tid>"
    },
    "subject": {
      "type": "entra_oid_tid",
      "tid": "<tid>",
      "oid": "<oid>"
    }
  }
}
```

### 3.4 Optional assurance signals

If present, PP SHOULD propagate:
- `auth_time`
- `amr`
- `acrs` / auth context indicators

If policy requires MFA or auth contexts, PP MUST enforce them. Requesting a claim or completing a
token refresh does not establish that a required condition was met. The deployment needs a documented
mapping from validated token evidence to policy, including how absent claims are handled. Microsoft
documents the available claims in its [ID token claims reference](https://learn.microsoft.com/en-us/entra/identity-platform/id-token-claims-reference).

### 3.5 Evidence embedding

If `requireEmbeddedEvidence=true`, PP MUST embed:
- the ID token (and optionally JWKS reference or JWK thumbprint)
and mark `identityBinding.evidence.embedded=true`.

If embedding is not required, PP MAY use a `tokenHash` instead of embedding the ID token; `nonceHash`
is still required by ENTRA-PP-03. An RP without the underlying token and session evidence relies on
the PP's signed assertion and its configured trust policy. Certification does not make an omitted
token independently verifiable.

## 4. Verification (RP side)

If RP requires identity binding, RP MUST:
- verify PP signature + trust
- verify `identityBinding.scheme == "entra_oidc"`
- verify `tid+oid` match the RP’s authenticated user identity

If RP requires embedded evidence, RP MUST:
- verify the embedded ID token signature (using JWKS),
- validate the expected issuer, PP client audience, and token time claims,
- verify `nonce` / `nonceHash` binding,
- enforce tenant restrictions and any required assurance conditions.

RP SHOULD apply bounded clock skew to embedded-token time validation (`exp`/`nbf`/`iat`) consistently with RP credential-validation policy.

## 5. Limitations

An OIDC token supplies issuer assertions about authentication and identity. It does not establish
human actuation, correct display, approval of the Action Intent, or authority to execute it. Account
compromise, federation policy, token validation, session binding, and the approval UI remain relevant
trust boundaries. A HAPS-CC carries the PP's approval assertion and still requires all
[core RP checks](../haps-v0.4.0.md#12-verification-requirements-rp).

This draft names `nonceHash` but does not specify a complete interoperable evidence encoding,
nonce-hash construction, or RP session-state exchange. Deployments must document those details; the
example subject block and tests do not establish end-to-end Entra interoperability.
