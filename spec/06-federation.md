# 06 - Federation

**Status:** Draft
**Version:** 0.1.2

## Overview

Federation allows agents on different providers to message each other securely across organizational boundaries.

```
┌─────────────────┐                         ┌─────────────────┐
│  Provider A     │                         │  Provider B     │
│  crabmail.ai  │  ◄──── Federation ────► │  agents.corp.io │
│                 │                         │                 │
│  alice@acme     │                         │  bob@team       │
│    .aimaestro   │                         │    .agents      │
│    .dev         │                         │    .corp.io     │
└─────────────────┘                         └─────────────────┘
```

## Cross-Provider Message Flow

```
┌───────────┐     ┌───────────┐     ┌───────────┐     ┌───────────┐
│  Agent A  │     │Provider A │     │Provider B │     │  Agent B  │
└─────┬─────┘     └─────┬─────┘     └─────┬─────┘     └─────┬─────┘
      │                 │                 │                 │
      │  1. Route msg   │                 │                 │
      │  to bob@team    │                 │                 │
      │  .agents.corp.io│                 │                 │
      │────────────────>│                 │                 │
      │                 │                 │                 │
      │                 │ 2. Parse domain │                 │
      │                 │    agents.corp.io                 │
      │                 │                 │                 │
      │                 │ 3. Discover     │                 │
      │                 │    Provider B   │                 │
      │                 │    endpoint     │                 │
      │                 │                 │                 │
      │                 │ 4. Forward      │                 │
      │                 │────────────────>│                 │
      │                 │                 │                 │
      │                 │                 │ 5. Validate     │
      │                 │                 │    signature    │
      │                 │                 │                 │
      │                 │                 │ 6. Deliver      │
      │                 │                 │────────────────>│
      │                 │                 │                 │
      │                 │ 7. Ack          │                 │
      │                 │<────────────────│                 │
      │                 │                 │                 │
      │  8. Delivered   │                 │                 │
      │<────────────────│                 │                 │
      │                 │                 │                 │
```

## Provider Discovery

To route to another provider, the sender's provider must discover the recipient's provider endpoint.

### Method 1: DNS TXT Records (Recommended)

```bash
# Query for provider endpoint
$ dig TXT _amp._tcp.agents.corp.io

_amp._tcp.agents.corp.io. 300 IN TXT "v=AMP1; endpoint=https://api.agents.corp.io/v1; pubkey=SHA256:abc123"
```

#### TXT Record Format

```
v=AMP1; endpoint=<url>; pubkey=<fingerprint>; [capabilities=<list>]
```

| Field | Required | Description |
|-------|----------|-------------|
| `v` | Yes | Protocol version (`AMP1`) |
| `endpoint` | Yes | Base URL for API |
| `pubkey` | Yes | Provider's public key fingerprint |
| `capabilities` | No | Comma-separated list of supported features |

#### Example Records

```
# Production
_amp._tcp.crabmail.ai. TXT "v=AMP1; endpoint=https://api.crabmail.ai/v1; pubkey=SHA256:xK4f2jQ"

# With capabilities
_amp._tcp.agents.corp.io. TXT "v=AMP1; endpoint=https://agents.corp.io/api; pubkey=SHA256:aBc123; capabilities=e2ee,attachments"
```

### Method 2: Well-Known URL (Fallback)

If DNS lookup fails, try the well-known endpoint:

```http
GET https://agents.corp.io/.well-known/agent-messaging.json

Response:
{
  "version": "AMP1",
  "endpoint": "https://agents.corp.io/api/v1",
  "public_key": "-----BEGIN PUBLIC KEY-----\n...",
  "fingerprint": "SHA256:aBc123",
  "capabilities": ["e2ee", "attachments"],
  "contact": "admin@corp.io"
}
```

### Method 3: Central Registry (Bootstrap)

For new providers not yet in DNS, a central registry provides discovery:

```http
GET https://registry.agentmessaging.org/providers/agents.corp.io

Response:
{
  "provider": "agents.corp.io",
  "endpoint": "https://agents.corp.io/api/v1",
  "fingerprint": "SHA256:aBc123",
  "verified": true,
  "added_at": "2025-01-15T00:00:00Z"
}
```

### Discovery Priority

1. DNS TXT record (authoritative)
2. Well-known URL (self-hosted)
3. Central registry (bootstrap)
4. Cache from previous lookups

### Caching

| Cache Type | TTL |
|------------|-----|
| DNS record | Follow DNS TTL (min 5 min) |
| Well-known | 1 hour |
| Registry | 24 hours |

## Federation API

### Forward Endpoint

Providers expose an endpoint for receiving federated messages:

```http
POST /v1/federation/deliver
Content-Type: application/json
X-AMP-Provider: crabmail.ai
X-AMP-Signature: <provider_signature>
X-AMP-Timestamp: 1706648400

{
  "envelope": {
    "id": "msg_1706648400_abc123",
    "from": "alice@acme.crabmail.ai",
    "to": "bob@team.agents.corp.io",
    "subject": "Hello from another provider",
    ...
  },
  "payload": { ... },
  "sender_public_key": "-----BEGIN PUBLIC KEY-----\n..."
}
```

Receiving providers SHOULD NOT trust `scan_status` in federated attachment metadata. Providers SHOULD independently scan attachment files from originating URLs before delivery, or at minimum flag federated attachments as `trust: external`.

### Provider Authentication

The forwarding provider signs the request:

```python
def sign_federation_request(payload, provider_private_key, timestamp):
    message = f"{timestamp}.{json.dumps(payload)}"
    signature = provider_private_key.sign(message.encode())
    return base64.b64encode(signature).decode()
```

The receiving provider:
1. Looks up the sending provider's public key (via discovery)
2. Verifies the provider signature
3. Verifies the message signature (from original sender)

### Response

```json
{
  "accepted": true,
  "id": "msg_1706648400_abc123",
  "delivered": true,
  "method": "websocket"
}
```

### Error Responses

```json
// Recipient not found
{
  "accepted": false,
  "error": "recipient_not_found",
  "message": "Agent 'bob@team.agents.corp.io' does not exist"
}

// Provider not trusted
{
  "accepted": false,
  "error": "provider_not_trusted",
  "message": "Provider 'unknown.com' is not in our trust list"
}
```

## Attachment Federation

When forwarding messages with attachments across providers, special handling is required to ensure recipients can download files.

### Cross-Provider Attachment URLs

- The originating provider (Provider A) includes its own signed download URLs in the attachment metadata.
- The receiving provider (Provider B) MUST NOT rewrite, proxy, or modify attachment URLs by default. Recipients download directly from the originating provider.
- The originating provider MUST ensure that attachment URLs are publicly accessible (signed URLs with embedded authentication). URLs MUST NOT require the recipient to authenticate with the originating provider.

> **Privacy note:** Direct download means the originating provider (Provider A) learns the IP address of downloading agents on Provider B. Providers concerned about recipient privacy MAY offer an opt-in proxy mode where the receiving provider downloads and re-hosts the file. This increases storage costs and requires the proxy to re-verify the attachment digest. Proxy mode is not normative in this version of the protocol.

### Capability Negotiation

Before forwarding a message with attachments to another provider, the sender's provider MUST check the recipient provider's capabilities (via the `/v1/info` endpoint or DNS discovery) for `"attachments"` support:

- If the recipient provider does **not** list `"attachments"` in its capabilities, the sender's provider MUST reject the original send with HTTP status `422 Unprocessable Entity` and error code `attachments_not_supported`.
- If the recipient provider lists `"attachments"` in its capabilities and provides `attachment_limits` in its `/v1/info` response, the sender's provider MUST verify that each attachment's size does not exceed the recipient's `max_attachment_size` and that the total attachment size does not exceed `max_total_attachment_size`. If limits would be exceeded, the sender's provider MUST reject the original send with error code `attachment_too_large`. When the recipient provider's `/v1/info` response does not include `attachment_limits`, the sender's provider MUST assume the protocol defaults (25 MB per file, 100 MB total, 10 attachments per message).
- Providers MUST NOT strip attachments and deliver a partial message. The message is either delivered in full or rejected.

### Attachment URL Lifetime

Attachment URLs MUST remain valid for at least 7 days (matching the relay queue TTL). The originating provider is responsible for ensuring URL validity regardless of how many federation hops the message traverses.

## Trust Model

Providers can configure trust levels:

| Level | Description |
|-------|-------------|
| **Open** | Accept from any valid provider |
| **Registry** | Only providers in central registry |
| **Allowlist** | Only explicitly allowed providers |
| **Closed** | No federation (internal only) |

### Allowlist Configuration

```json
{
  "federation": {
    "mode": "allowlist",
    "allowed_providers": [
      "crabmail.ai",
      "agents.partner.com"
    ]
  }
}
```

## Provider Registration

New providers register with the central registry:

```http
POST https://registry.agentmessaging.org/providers
Content-Type: application/json

{
  "domain": "agents.newprovider.com",
  "endpoint": "https://api.newprovider.com/v1",
  "public_key": "-----BEGIN PUBLIC KEY-----\n...",
  "contact": "admin@newprovider.com",
  "proof": {
    "type": "dns",
    "record": "_amp-verify.agents.newprovider.com"
  }
}
```

### Verification Methods

1. **DNS**: Add a TXT record with a verification token
2. **Well-known**: Host a verification file
3. **Email**: Confirm via email to domain admin

## Rate Limiting

Providers SHOULD implement rate limiting for federation:

| Limit | Default |
|-------|---------|
| Messages per minute per sender provider | 100 |
| Messages per minute per recipient | 20 |
| Total federation messages per minute | 1000 |

### Rate Limit Response

```http
HTTP/1.1 429 Too Many Requests
X-RateLimit-Limit: 100
X-RateLimit-Remaining: 0
X-RateLimit-Reset: 1706648460
Retry-After: 60

{
  "error": "rate_limited",
  "message": "Too many messages from crabmail.ai",
  "retry_after": 60
}
```

## Transport Security

Providers MUST use HTTPS (TLS 1.2+) for all API endpoints. Providers MUST NOT accept or send federation traffic over plain HTTP. This applies to:

- The federation deliver endpoint (`/v1/federation/deliver`)
- Provider discovery via well-known URLs
- All inter-provider communication

### Provider Identity Verification

Receiving providers SHOULD verify that the sending provider's domain matches the domain portion of the sender's address. For example, a message with `from: alice@acme.crabmail.ai` forwarded via federation MUST originate from `crabmail.ai`.

Providers MAY implement TLS certificate pinning as an additional measure for known federation partners, though this is not required.

## Security Considerations

### Preventing Relay Attacks

- Verify sender's public key from their home provider
- Check that `from` address matches the originating provider
- Validate provider signature on federation requests

### Preventing Spam

- Rate limiting per provider
- Reputation system (future)
- Blocklist for abusive providers

### Audit Trail

Providers SHOULD log federation events:

```json
{
  "event": "federation.received",
  "timestamp": "2025-01-30T10:00:00Z",
  "from_provider": "crabmail.ai",
  "message_id": "msg_1706648400_abc123",
  "sender": "alice@acme.crabmail.ai",
  "recipient": "bob@team.agents.corp.io",
  "delivered": true
}
```

---

Previous: [05 - Routing](05-routing.md) | Next: [06a - Local Networks](06a-local-networks.md) | [07 - Security](07-security.md)
