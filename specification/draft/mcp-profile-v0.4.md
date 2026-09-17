# HAPS over MCP v0.4 (Profile)

**Status:** Experimental draft; not production-ready.
**Version:** 0.4  
**Tool namespace:** `haps.*`
**MCP transport revision:** `2025-11-25`

HAPS has a partial reference implementation. Passing the supplied tests does not establish full
protocol conformance, MCP interoperability, or that a person saw and approved the correct action.

## 1. Summary

This profile proposes how a Presence Provider (PP) is exposed as an **MCP Server** that can issue HAPS Consent Credentials.

It uses:
- MCP `tools/list` for discovery
- MCP `tools/call` for invocation
- **URL-mode** consent UI for sensitive interactions (PoHP + identity binding)

The URL flow below targets [MCP 2025-11-25 elicitation](https://modelcontextprotocol.io/specification/2025-11-25/client/elicitation).
Its `-32042` error is version-specific; this profile does not establish compatibility with earlier
or later MCP revisions. Implementations need matching protocol negotiation and URL elicitation
capability support.

## 2. Required capabilities

### Presence Provider MCP Server
- MUST implement MCP Tools.
- MUST expose tool `haps.request`.

### MCP Host/Client
- MUST declare support for URL elicitation (`capabilities.elicitation.url`) for this flow.
- MUST support safe navigation to provider-controlled UI (URL-mode), including:
  - clear domain display
  - user consent before navigation

## 3. Tool: `haps.request`

### 3.1 Purpose
Request a HAPS Consent Credential (HAPS-CC) for an Action Intent (AI-INTENT), optionally driven by an RP Challenge (HAPS-CHAL).

### 3.2 Input arguments (logical model)

`haps.request` accepts either:

A) Agent-initiated:
- `actionIntent` (AI-INTENT)
- `requirements` (PoHP + optional identity binding)

B) RP Challenge Mode:
- `challenge` (HAPS-CHAL)

For high-risk actions, MCP Hosts/Clients SHOULD prefer RP Challenge Mode over agent-initiated mode.

All requests SHOULD include:
- `requestId` (client-generated stable id to correlate retries)
- `return.format` (`jwt` or `vc+json`)

These are logical arguments, not a complete published MCP input schema. Implementations need to
document their accepted argument shape, supported credential formats, retry behavior, and optional
profiles; a mention here is not evidence that the partial reference implements them.

### 3.3 Output

If a completed approval session satisfies the request and issuance requirements, the tool returns:
- `structuredContent` containing the [credential envelope](../../schemas/haps-consent-credential.v0.4.schema.json)
- if request input used `challenge`, returned claims MUST carry matching `challengeId`

If user interaction is required, PP SHOULD return a JSON-RPC error:
- `code: -32042`
- `data.elicitations[]` including:
  - `mode: "url"`
  - `elicitationId`
  - `url`
  - `message`

After the user's navigation consent, the host opens the URL and may retry the tool call after
completion. Opening the URL, receiving a completion notification, or retrying is not itself action
approval. The PP must check the session's actual approval and required evidence before issuance.

## 4. Message flow (example)

### 4.1 First call (not yet approved)
`tools/call` → error with URL elicitation required

### 4.2 Provider collects required evidence and an explicit approval event in its UI

### 4.3 Second call (session approved and requirements satisfied)
`tools/call` → returns HAPS-CC; the RP still performs all core verification and authorization checks.

## 5. Notes

- Sensitive interactions (biometrics, enterprise login) MUST be hosted on the PP domain (URL-mode), not in-band form fields.
- The PP MUST bind the issued credential to:
  - `intent_hash`
  - `presentation_hash`
  - `aud` (RP audience)
- For challenge-mode requests, PP MUST bind and return `claims.challengeId` equal to the supplied challenge.

URL mode supplies a transport pattern; it does not establish that the browser displayed the correct
action or that the person understood and approved it. The deployment depends on a trustworthy PP,
protected signing keys, correct UI and session binding, evaluated factors, and RP enforcement of
freshness, single use, and execution of the validated action. MCP tool discovery and a matching
`presentation_hash` do not independently establish those conditions or prevent fraud.

## Digital Credentials + portrait/liveness adapter

When a Presence Provider supports the [Digital Credentials profile](adapters/digital-credentials-portrait-liveness-v0.1.md), an MCP request MAY include policy requirements indicating plain Digital Credentials presentment, liveness-backed presentment, or trusted liveness issuer mode. After the requested policy and approval requirements are satisfied, the provider SHOULD return a normal HAPS Consent Credential with `identityBinding.scheme="digital_credentials_portrait_liveness"`; raw portrait and biometric material MUST NOT be returned in the MCP result.
