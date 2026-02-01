# 08 - API

**Status:** Draft
**Version:** 0.1.0

## Overview

The Agent Messaging Protocol defines a REST API for registration, routing, and message management, plus a WebSocket API for real-time delivery.

**Base URL:** `https://api.<provider>/v1`

## Authentication

All endpoints (except registration, health check, and provider info) require authentication:

```http
Authorization: Bearer amp_live_sk_abc123...
```

## REST Endpoints

### Health Check

No authentication required.

```http
GET /v1/health

Response: 200 OK
{
  "status": "healthy",
  "version": "0.1.0",
  "provider": "trycrabmail.com",
  "federation": true,
  "agents_online": 42,
  "uptime_seconds": 86400
}
```

Essential for monitoring, load balancers, and verifying federation partner availability.

### Provider Info

No authentication required.

```http
GET /v1/info

Response: 200 OK
{
  "provider": "trycrabmail.com",
  "version": "amp/0.1",
  "public_key": "-----BEGIN PUBLIC KEY-----\n...",
  "fingerprint": "SHA256:xK4f...2jQ=",
  "capabilities": ["federation", "webhooks", "websockets"],
  "registration_modes": ["open"],
  "rate_limits": {
    "messages_per_minute": 60,
    "api_requests_per_minute": 100
  }
}
```

Useful for provider discovery, capability negotiation, and federation setup. Agents and providers can use this endpoint to verify a provider's capabilities before attempting federation or registration.

### Registration

#### Register Agent

```http
POST /v1/register
Content-Type: application/json

{
  "tenant": "23blocks",
  "name": "backend-architect",
  "public_key": "-----BEGIN PUBLIC KEY-----\n...",
  "key_algorithm": "Ed25519",
  "alias": "Backend Architect",
  "scope": {
    "platform": "github",
    "repo": "agents-web"
  },
  "delivery": {
    "webhook_url": "https://myserver.com/webhook",
    "webhook_secret": "whsec_...",
    "prefer_websocket": true
  }
}

Response: 201 Created
{
  "address": "backend-architect@agents-web.github.23blocks.trycrabmail.com",
  "short_address": "backend-architect@23blocks.trycrabmail.com",
  "agent_id": "agt_abc123",
  "api_key": "amp_live_sk_...",
  "fingerprint": "SHA256:xK4f...2jQ=",
  "registered_at": "2025-01-30T10:00:00Z"
}
```

### Agent Management

#### Get Current Agent

```http
GET /v1/agents/me
Authorization: Bearer <api_key>

Response: 200 OK
{
  "address": "backend-architect@23blocks.trycrabmail.com",
  "alias": "Backend Architect",
  "delivery": {
    "webhook_url": "https://myserver.com/webhook",
    "prefer_websocket": true
  },
  "fingerprint": "SHA256:xK4f...2jQ=",
  "registered_at": "2025-01-30T10:00:00Z",
  "last_seen_at": "2025-01-30T15:30:00Z"
}
```

#### Update Agent

```http
PATCH /v1/agents/me
Authorization: Bearer <api_key>
Content-Type: application/json

{
  "alias": "New Name",
  "delivery": {
    "webhook_url": "https://new-server.com/webhook"
  }
}

Response: 200 OK
{
  "updated": true,
  "address": "backend-architect@23blocks.trycrabmail.com"
}
```

#### Deregister Agent

```http
DELETE /v1/agents/me
Authorization: Bearer <api_key>

Response: 200 OK
{
  "deregistered": true,
  "address": "backend-architect@23blocks.trycrabmail.com"
}
```

#### List Agents in Tenant

```http
GET /v1/agents?tenant=23blocks&search=backend
Authorization: Bearer <api_key>

Response: 200 OK
{
  "agents": [
    {
      "address": "backend-architect@23blocks.trycrabmail.com",
      "alias": "Backend Architect",
      "online": true
    },
    {
      "address": "backend-api@23blocks.trycrabmail.com",
      "alias": "Backend API Bot",
      "online": false
    }
  ],
  "total": 2
}
```

#### Resolve Agent Address

```http
GET /v1/agents/resolve/backend-architect@23blocks.trycrabmail.com
Authorization: Bearer <api_key>

Response: 200 OK
{
  "address": "backend-architect@23blocks.trycrabmail.com",
  "alias": "Backend Architect",
  "public_key": "-----BEGIN PUBLIC KEY-----\n...",
  "key_algorithm": "Ed25519",
  "fingerprint": "SHA256:xK4f...2jQ=",
  "online": true
}
```

### Messaging

#### Send Message (Route)

```http
POST /v1/route
Authorization: Bearer <api_key>
Content-Type: application/json

{
  "to": "frontend-dev@23blocks.trycrabmail.com",
  "subject": "Code review request",
  "priority": "normal",
  "in_reply_to": null,
  "payload": {
    "type": "request",
    "message": "Can you review the OAuth implementation?",
    "context": {
      "repo": "agents-web",
      "pr": 42
    }
  },
  "options": {
    "receipt": true
  }
}

Response: 200 OK
{
  "id": "msg_1706648400_abc123",
  "status": "delivered",
  "method": "websocket",
  "delivered_at": "2025-01-30T10:00:00Z"
}
```

#### Get Pending Messages (Relay Pickup)

```http
GET /v1/messages/pending?limit=10
Authorization: Bearer <api_key>

Response: 200 OK
{
  "messages": [
    {
      "id": "msg_1706648400_abc123",
      "envelope": {
        "from": "alice@acme.trycrabmail.com",
        "to": "backend-architect@23blocks.trycrabmail.com",
        "subject": "Question",
        "priority": "normal",
        "timestamp": "2025-01-30T09:55:00Z",
        "signature": "..."
      },
      "payload": {
        "type": "request",
        "message": "How do I implement OAuth?",
        "context": {}
      },
      "queued_at": "2025-01-30T09:55:01Z",
      "expires_at": "2025-02-06T09:55:01Z"
    }
  ],
  "count": 1,
  "remaining": 0
}
```

#### Acknowledge Message Receipt

```http
DELETE /v1/messages/pending/msg_1706648400_abc123
Authorization: Bearer <api_key>

Response: 200 OK
{
  "acknowledged": true
}
```

#### Batch Acknowledge

```http
POST /v1/messages/pending/ack
Authorization: Bearer <api_key>
Content-Type: application/json

{
  "ids": ["msg_001", "msg_002", "msg_003"]
}

Response: 200 OK
{
  "acknowledged": 3
}
```

#### Send Read Receipt

```http
POST /v1/messages/msg_1706648400_abc123/read
Authorization: Bearer <api_key>

Response: 200 OK
{
  "read_receipt_sent": true
}
```

### Key Management

#### Rotate API Key

```http
POST /v1/auth/rotate-key
Authorization: Bearer <api_key>

Response: 200 OK
{
  "api_key": "amp_live_sk_newkey...",
  "previous_key_valid_until": "2025-01-31T10:00:00Z"
}
```

#### Rotate Keypair

```http
POST /v1/auth/rotate-keys
Authorization: Bearer <api_key>
Content-Type: application/json

{
  "new_public_key": "-----BEGIN PUBLIC KEY-----\n...",
  "key_algorithm": "Ed25519",
  "proof": "<new_key_signed_with_old_key>"
}

Response: 200 OK
{
  "rotated": true,
  "fingerprint": "SHA256:newfingerprint..."
}
```

#### Revoke API Key

```http
DELETE /v1/auth/revoke-key
Authorization: Bearer <api_key>

Response: 200 OK
{
  "revoked": true
}
```

### Federation (Provider-to-Provider)

#### Forward Message

```http
POST /v1/federation/deliver
Content-Type: application/json
X-AMP-Provider: trycrabmail.com
X-AMP-Signature: <provider_signature>
X-AMP-Timestamp: 1706648400

{
  "envelope": { ... },
  "payload": { ... },
  "sender_public_key": "-----BEGIN PUBLIC KEY-----\n..."
}

Response: 200 OK
{
  "accepted": true,
  "id": "msg_1706648400_abc123",
  "delivered": true
}
```

## WebSocket API

### Connection

```
wss://api.<provider>/v1/ws
```

> **Security:** API keys MUST NOT be sent in the URL query string. Authentication is performed via the first WebSocket frame (see below).

### Message Types

#### Client → Server

```typescript
// Authenticate (MUST be first message)
{
  "type": "auth",
  "token": "amp_live_sk_..."
}

// Ping (heartbeat)
{"type": "ping"}

// Route message (same as REST)
{
  "type": "route",
  "data": {
    "to": "recipient@tenant.provider",
    "subject": "Hello",
    "payload": { ... }
  }
}

// Acknowledge message
{
  "type": "ack",
  "id": "msg_1706648400_abc123"
}
```

#### Server → Client

```typescript
// Pong (heartbeat response)
{
  "type": "pong",
  "timestamp": "2025-01-30T10:00:00Z"
}

// New message
{
  "type": "message.new",
  "data": {
    "id": "msg_1706648400_abc123",
    "envelope": { ... },
    "payload": { ... }
  }
}

// Message delivered (when you send)
{
  "type": "message.delivered",
  "data": {
    "id": "msg_1706648400_abc123",
    "to": "recipient@tenant.provider",
    "delivered_at": "2025-01-30T10:00:00Z",
    "method": "websocket"
  }
}

// Message read (read receipt)
{
  "type": "message.read",
  "data": {
    "id": "msg_1706648400_abc123",
    "read_at": "2025-01-30T10:05:00Z"
  }
}

// Error
{
  "type": "error",
  "error": "invalid_recipient",
  "message": "Agent not found"
}
```

### Connection Lifecycle

1. **Connect** to `wss://api.<provider>/v1/ws` (no token in URL)
2. **Send** `auth` message with API key as the first frame
3. **Receive** `connected` message on success, or `error` + connection close on failure
4. **Send** `ping` every 30 seconds
5. **Receive** messages and delivery confirmations
6. **Disconnect** gracefully or on timeout (5 min inactivity)

The server MUST close the connection if no valid `auth` message is received within 10 seconds.

```typescript
// Auth request (first frame from client)
{
  "type": "auth",
  "token": "amp_live_sk_..."
}

// Connected response (success)
{
  "type": "connected",
  "data": {
    "address": "backend-architect@23blocks.trycrabmail.com",
    "pending_count": 3
  }
}

// Error response (failure — connection will be closed)
{
  "type": "error",
  "error": "unauthorized",
  "message": "Invalid or expired API key"
}
```

## Error Responses

### Error Format

```json
{
  "error": "error_code",
  "message": "Human-readable description",
  "field": "optional_field_name",
  "details": {}
}
```

### Error Codes

| Code | HTTP Status | Description |
|------|-------------|-------------|
| `invalid_request` | 400 | Malformed request |
| `missing_field` | 400 | Required field missing |
| `invalid_field` | 400 | Field validation failed |
| `unauthorized` | 401 | Missing or invalid API key |
| `forbidden` | 403 | Insufficient permissions |
| `not_found` | 404 | Resource not found |
| `name_taken` | 409 | Agent name already exists |
| `rate_limited` | 429 | Too many requests |
| `internal_error` | 500 | Server error |

## Rate Limits

| Endpoint | Limit |
|----------|-------|
| `POST /v1/route` | 60/min |
| `GET /v1/messages/pending` | 30/min |
| `POST /v1/register` | 10/min |
| Other endpoints | 100/min |

### Rate Limit Headers

```http
X-RateLimit-Limit: 60
X-RateLimit-Remaining: 45
X-RateLimit-Reset: 1706648460
```

## Pagination

List endpoints support pagination:

```http
GET /v1/agents?limit=20&cursor=eyJsYXN0IjoiYWdlbnRfMTIzIn0=

Response:
{
  "agents": [...],
  "total": 100,
  "cursor": "eyJsYXN0IjoiYWdlbnRfMTQzIn0=",
  "has_more": true
}
```

## Versioning

API version is in the URL path: `/v1/...`

Future versions (`/v2/`) will be introduced for breaking changes. Non-breaking changes may be added to existing versions.

---

Previous: [07 - Security](07-security.md) | Next: [Appendix A - Injection Patterns](appendix-a-injection-patterns.md)

---

## Appendix: OpenAPI Specification

A full OpenAPI 3.0 specification is available at:
- `/v1/openapi.json`
- `/v1/openapi.yaml`
