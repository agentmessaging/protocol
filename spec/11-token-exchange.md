# 11 - Agent Identity Token Exchange

**Status:** Draft
**Version:** 0.1.2

## Overview

This section defines a standard mechanism for AI agents to authenticate with third-party APIs using their AMP Agent Identity. An agent presents its signed Agent Identity document (see [02 - Identity](02-identity.md#agent-card-portable-identity)) and a proof of possession to obtain an OAuth 2.0 access token.

This enables agents to access external services — file storage, databases, SaaS APIs — using their cryptographic Agent Identity instead of managing separate credentials for each service.

> **Note:** This grant type is part of the Agent Identity (AID) protocol layer — a standalone authentication/authorization protocol built on top of AMP's identity primitives. See [github.com/agentmessaging/agent-identity](https://github.com/agentmessaging/agent-identity) for the reference implementation.

## Grant Type

The AMP token exchange uses a custom OAuth 2.0 grant type following the Token Exchange pattern defined in [RFC 8693](https://datatracker.ietf.org/doc/html/rfc8693):

```
urn:aid:agent-identity
```

## Token Request

### Endpoint

```http
POST /{tenant_url_id}/oauth/token
Content-Type: application/x-www-form-urlencoded
```

The `{tenant_url_id}` is the URL identifier for the tenant (organization) on the auth server. This allows multi-tenant auth servers to scope token issuance per organization.

### Parameters

| Parameter | Required | Description |
|-----------|----------|-------------|
| `grant_type` | Yes | Must be `urn:aid:agent-identity` |
| `agent_identity` | Yes | Base64url-encoded signed Agent Identity JSON (see [02 - Identity](02-identity.md#agent-card-format)) |
| `proof` | Yes | Base64url-encoded proof of possession (see [Proof of Possession](#proof-of-possession)) |
| `scope` | No | Space-separated list of requested scopes (e.g., `files:read files:write`) |

### Example Request

```http
POST /acme/oauth/token HTTP/1.1
Host: auth.example.com
Content-Type: application/x-www-form-urlencoded

grant_type=urn%3Aaid%3Aagent-identity
&agent_identity=eyJhbXBfYWdlbnRfY2FyZCI6IjEuMCIsImFkZHJlc3MiOi...
&proof=c2lnbmF0dXJlX2J5dGVzLi4udGltZXN0YW1w...
&scope=files%3Aread+files%3Awrite
```

## Proof of Possession

The proof of possession demonstrates that the agent holds the Ed25519 private key corresponding to the public key in its Agent Card. This prevents stolen or intercepted Agent Cards from being used to obtain tokens.

### Construction

1. Build the signing input string:

```
sign_input = "aid-token-exchange\n" + unix_timestamp + "\n" + issuer_url
```

Where:
- `"aid-token-exchange\n"` is a fixed domain separator (prevents cross-protocol signature reuse)
- `unix_timestamp` is the current time as a decimal string (e.g., `"1706616000"`)
- `issuer_url` is the base URL of the auth server issuing the token (e.g., `"https://auth.example.com"`)

2. Sign the input with the agent's Ed25519 private key:

```
signature_bytes = ed25519_sign(private_key, sign_input)  # 64 bytes
```

3. Concatenate the signature and timestamp, then base64url-encode:

```
proof_bytes = signature_bytes (64 bytes) + timestamp_string (UTF-8 bytes)
proof = base64url_encode(proof_bytes)
```

### Example (Python)

```python
import time
import base64
from cryptography.hazmat.primitives.asymmetric.ed25519 import Ed25519PrivateKey

def create_proof(private_key: Ed25519PrivateKey, issuer_url: str) -> str:
    timestamp = str(int(time.time()))
    sign_input = f"aid-token-exchange\n{timestamp}\n{issuer_url}".encode()

    signature = private_key.sign(sign_input)  # 64 bytes

    proof_bytes = signature + timestamp.encode()
    return base64.urlsafe_b64encode(proof_bytes).decode().rstrip("=")
```

### Example (Bash / OpenSSL)

```bash
TIMESTAMP=$(date +%s)
ISSUER_URL="https://auth.example.com"
SIGN_INPUT="aid-token-exchange\n${TIMESTAMP}\n${ISSUER_URL}"

# Sign with Ed25519
SIGNATURE=$(printf '%b' "$SIGN_INPUT" | openssl pkeyutl -sign -inkey private.pem -rawin | base64)

# Concatenate signature + timestamp and base64url-encode
PROOF=$(printf '%s%s' "$(echo "$SIGNATURE" | base64 -d)" "$TIMESTAMP" | base64 | tr '+/' '-_' | tr -d '=')
```

## Auth Server Verification

The auth server (token issuer) MUST perform the following verification steps, in order:

### 1. Decode and Parse the Agent Identity

Decode the `agent_identity` parameter from base64url and parse as JSON. Verify it conforms to the Agent Identity format defined in [02 - Identity](02-identity.md#agent-card-format).

### 2. Verify the Agent Identity Signature

Follow the Agent Identity verification procedure from [02 - Identity](02-identity.md#verifying-an-agent-card):

1. Extract and remove the `signature` field from the card.
2. Serialize the remaining fields using JCS ([RFC 8785](https://datatracker.ietf.org/doc/html/rfc8785)).
3. Prepend the domain separator: `"amp-agent-card-v1\n"`.
4. Verify the Ed25519 signature against the `public_key` in the card.

If the signature is invalid, return `invalid_grant`.

### 3. Check Agent Identity Expiration

Verify that the identity document's `expires_at` timestamp is in the future. Expired identity documents MUST be rejected with `invalid_grant`.

### 4. Decode and Verify the Proof

1. Base64url-decode the `proof` parameter.
2. Split into signature (first 64 bytes) and timestamp (remaining bytes, UTF-8).
3. Parse the timestamp as a Unix epoch integer.
4. Verify the timestamp is within **5 minutes** of the auth server's current time. If expired or too far in the future, return `invalid_proof`.
5. Reconstruct the signing input: `"aid-token-exchange\n" + timestamp + "\n" + issuer_url`.
6. Verify the Ed25519 signature against the public key from the Agent Card.

If the proof is invalid, return `invalid_proof`.

### 5. Check Agent Registration

The auth server MAY require that the agent's address is pre-registered. If registration is required and the agent is not found, return `agent_not_registered`.

### 6. Validate Scopes

If the `scope` parameter is present, verify that the requested scopes do not exceed the agent's permitted scopes. Permitted scopes are stored server-side and are NOT embedded in the Agent Card.

If any requested scope is not permitted, return `invalid_scope`.

### 7. Check Agent Status

If the agent has been suspended by an administrator, return `agent_suspended`.

### 8. Issue Token

If all checks pass, issue a standard OAuth 2.0 access token.

## Token Response

### Success Response

Standard OAuth 2.0 token response with an additional `agent_address` field:

```http
HTTP/1.1 200 OK
Content-Type: application/json
Cache-Control: no-store

{
  "access_token": "eyJhbGciOiJSUzI1NiIs...",
  "token_type": "Bearer",
  "expires_in": 3600,
  "scope": "files:read files:write",
  "agent_address": "backend-architect@23blocks.crabmail.ai"
}
```

| Field | Type | Description |
|-------|------|-------------|
| `access_token` | string | The issued access token (JWT or opaque) |
| `token_type` | string | Always `"Bearer"` |
| `expires_in` | integer | Token lifetime in seconds |
| `scope` | string | Space-separated granted scopes (may be a subset of requested) |
| `agent_address` | string | The agent's verified AMP address |

The `agent_address` field confirms which Agent Identity was authenticated. Auth servers SHOULD include it so clients can verify the token was issued for the correct agent.

### Token Lifetime

Auth servers SHOULD issue short-lived tokens:

| Use Case | Recommended `expires_in` |
|----------|--------------------------|
| Interactive agent sessions | 3600 (1 hour) |
| Background/batch operations | 900 (15 minutes) |
| Highly privileged scopes | 300 (5 minutes) |

Agents MUST request a new token when the current one expires. Refresh tokens are NOT used — agents re-authenticate with a fresh proof of possession.

## Error Responses

Error responses follow [RFC 6749 Section 5.2](https://datatracker.ietf.org/doc/html/rfc6749#section-5.2) with AMP-specific error codes:

```http
HTTP/1.1 400 Bad Request
Content-Type: application/json

{
  "error": "invalid_proof",
  "error_description": "Proof of possession signature verification failed"
}
```

### Error Codes

| Error Code | HTTP Status | Description |
|------------|-------------|-------------|
| `invalid_grant` | 400 | Agent Identity signature invalid, expired, or malformed |
| `invalid_proof` | 400 | Proof of possession signature verification failed or timestamp expired |
| `agent_not_registered` | 403 | Agent address not registered with this auth server |
| `invalid_scope` | 400 | Requested scopes exceed the agent's permitted scopes |
| `agent_suspended` | 403 | Agent has been suspended by administrator |
| `unsupported_grant_type` | 400 | Auth server does not support the `urn:aid:agent-identity` grant type |

## Security Considerations

### Stolen Agent Identity

An Agent Identity document alone is insufficient to obtain a token. The proof of possession requires the agent's Ed25519 private key to generate a valid signature. An attacker who intercepts or copies an Agent Identity document cannot authenticate without the corresponding private key.

### Replay Attacks

The proof of possession includes a Unix timestamp. Auth servers MUST reject proofs where the timestamp is more than **5 minutes** from the current server time. This limits the window for replay attacks to 5 minutes, and each proof is bound to a specific issuer URL, preventing cross-server replay.

Auth servers SHOULD additionally track recently used proof timestamps per agent to reject exact duplicates within the validity window.

### Scope Escalation

Scopes are stored server-side and managed by administrators. The Agent Card does not contain scope information. This prevents agents from forging or inflating their permissions by modifying the card.

When an agent requests scopes beyond its permissions, the auth server SHOULD either:
- Return `invalid_scope` and reject the request, or
- Issue a token with the intersection of requested and permitted scopes (downscoped)

### Compromised Agent Key

If an agent's Ed25519 private key is compromised:

1. The agent's AMP provider revokes the key (see [07 - Security](07-security.md#key-revocation)).
2. The auth server administrator suspends the agent's registration.
3. The compromised Agent Card's signature becomes invalid once the provider rotates the key.
4. Existing tokens SHOULD be revoked by the auth server.

Auth server implementations SHOULD provide an administrative API to suspend agent registrations immediately upon key compromise notification.

### Agent Impersonation

The proof of possession is the primary defense against impersonation. To impersonate an agent, an attacker would need:

1. A valid, signed Agent Card (obtainable, as cards may be shared)
2. The agent's Ed25519 private key (not obtainable without compromising the agent)

The private key never leaves the agent's local machine and is never transmitted over the network.

### Issuer URL Binding

The proof includes the auth server's issuer URL, preventing a proof generated for one auth server from being replayed at another. Auth servers MUST verify that the issuer URL in the proof matches their own issuer URL exactly.

## Relationship to Existing AMP Concepts

### Agent Identity (Section 02)

This section builds directly on the Agent Card format defined in [02 - Identity](02-identity.md#agent-card-portable-identity). The Agent Identity document serves as the identity assertion in the token exchange. All Agent Card signing and verification rules from Section 02 apply.

### Key Rotation (Section 07)

When an agent rotates its Ed25519 key, it MUST obtain a new Agent Identity document signed with the new key. Old identity documents become invalid for token exchange once the key is revoked. See [07 - Security](07-security.md#key-rotation).

### Provider Resolution (Section 08)

Auth servers MAY use the provider resolution endpoint (`GET /v1/agents/resolve/{address}`) to verify that the agent is currently registered and active with its claimed provider. This provides a real-time check beyond the static Agent Card.

## Implementation Notes

### For Auth Server Implementors

1. **Store scopes server-side.** Never trust scope claims from the Agent Card or client request alone.
2. **Validate the proof timestamp strictly.** The 5-minute window is a maximum — implementations MAY use a shorter window.
3. **Log all token issuance events** including agent address, requested scopes, granted scopes, and client IP for audit purposes.
4. **Support key rotation gracefully.** When a new Agent Identity document arrives with the same address but a different key, verify the signature and update the stored public key.

### For Agent Implementors

1. **Generate fresh proofs for each request.** Do not cache or reuse proof values.
2. **Handle token expiration.** Re-authenticate with a new proof when the access token expires. Do not store tokens beyond their `expires_in` lifetime.
3. **Protect the private key.** The private key file MUST have restricted permissions (0600). Never transmit it.
4. **Include the issuer URL accurately.** The issuer URL in the proof MUST match the auth server's issuer URL exactly, including scheme and port.

---

Previous: [10 - Local Bus](10-local-bus.md) | Next: [Appendix A - Injection Patterns](appendix-a-injection-patterns.md)
