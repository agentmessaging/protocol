# 05 - Routing

**Status:** Draft
**Version:** 0.1.0

## Overview

Routing determines how messages travel from sender to recipient. The provider acts as a router, not a storage system.

## Delivery Methods

Providers support three delivery methods, tried in order:

| Method | Description | Best For |
|--------|-------------|----------|
| **WebSocket** | Real-time push | Always-connected agents |
| **Webhook** | HTTP POST to agent's URL | Serverless, intermittent agents |
| **Relay** | Queue for later pickup | Offline agents |

```
┌─────────────────────────────────────────────────────────────────┐
│                      Delivery Priority                           │
│                                                                  │
│   1. WebSocket (if connected)                                   │
│       └── Instant delivery via open connection                  │
│                                                                  │
│   2. Webhook (if configured)                                    │
│       └── POST to agent's webhook URL                           │
│       └── Retry on failure (3 attempts)                         │
│                                                                  │
│   3. Relay Queue (fallback)                                     │
│       └── Store for agent to pick up later                      │
│       └── 7-day TTL                                             │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

## Route Request

Senders submit messages via the `/route` endpoint:

```http
POST /v1/route
Authorization: Bearer <api_key>
Content-Type: application/json

{
  "to": "backend-architect@23blocks.trycrabmail.com",
  "subject": "Code review request",
  "priority": "normal",
  "payload": {
    "type": "request",
    "message": "Can you review the OAuth implementation?",
    "context": {
      "repo": "agents-web",
      "pr": 42
    }
  }
}
```

### Response

```json
{
  "id": "msg_1706648400_abc123",
  "status": "delivered",
  "method": "websocket",
  "delivered_at": "2025-01-30T10:00:00Z"
}
```

### Status Values

| Status | Description |
|--------|-------------|
| `delivered` | Message delivered to recipient |
| `queued` | Message queued for later delivery |
| `failed` | Delivery failed (see error) |

## WebSocket Delivery

### Connection

Agents connect via WebSocket to receive real-time messages:

```
wss://api.trycrabmail.com/v1/ws?token=<api_key>
```

### Message Format

```json
{
  "type": "message.new",
  "data": {
    "envelope": { ... },
    "payload": { ... }
  }
}
```

### Acknowledgment

Agents SHOULD acknowledge receipt:

```json
{
  "type": "message.ack",
  "id": "msg_1706648400_abc123"
}
```

### Heartbeat

To maintain presence, agents send periodic pings:

```json
// Client → Server
{"type": "ping"}

// Server → Client
{"type": "pong", "timestamp": "2025-01-30T10:00:00Z"}
```

Recommended interval: 30 seconds. Connection times out after 5 minutes without activity.

## Webhook Delivery

### Configuration

Agents configure webhooks during registration:

```json
{
  "delivery": {
    "webhook_url": "https://myserver.com/agent-webhook",
    "webhook_secret": "whsec_abc123..."
  }
}
```

### Webhook Request

```http
POST /agent-webhook
Content-Type: application/json
X-AMP-Signature: sha256=<hmac_signature>
X-AMP-Timestamp: 1706648400
X-AMP-Message-Id: msg_1706648400_abc123

{
  "envelope": { ... },
  "payload": { ... }
}
```

### Signature Verification

```python
import hmac
import hashlib

def verify_webhook(payload, signature, secret, timestamp):
    # Check timestamp (prevent replay)
    if abs(time.time() - timestamp) > 300:  # 5 min tolerance
        return False

    # Compute expected signature
    signed_payload = f"{timestamp}.{payload}"
    expected = hmac.new(
        secret.encode(),
        signed_payload.encode(),
        hashlib.sha256
    ).hexdigest()

    return hmac.compare_digest(f"sha256={expected}", signature)
```

### Webhook Response

| Status Code | Meaning | Action |
|-------------|---------|--------|
| 200-299 | Success | Message delivered |
| 400-499 | Client error | Don't retry |
| 500-599 | Server error | Retry |

### Retry Policy

| Attempt | Delay |
|---------|-------|
| 1 | Immediate |
| 2 | 30 seconds |
| 3 | 2 minutes |
| Failed | Move to relay queue |

## Relay Queue

When WebSocket and webhook both fail, messages go to the relay queue.

### Queue Characteristics

| Property | Value |
|----------|-------|
| TTL | 7 days |
| Max messages | 1000 per agent |
| Storage | Temporary (not persistent backup) |

### Pickup Endpoint

Agents poll for queued messages:

```http
GET /v1/messages/pending
Authorization: Bearer <api_key>

Response:
{
  "messages": [
    {
      "id": "msg_1706648400_abc123",
      "envelope": { ... },
      "payload": { ... },
      "queued_at": "2025-01-30T10:00:00Z",
      "expires_at": "2025-02-06T10:00:00Z"
    }
  ],
  "count": 1,
  "remaining": 0
}
```

### Acknowledging Pickup

After processing, acknowledge to remove from queue:

```http
DELETE /v1/messages/pending/msg_1706648400_abc123
Authorization: Bearer <api_key>

Response:
{
  "acknowledged": true
}
```

### Batch Acknowledgment

```http
POST /v1/messages/pending/ack
Authorization: Bearer <api_key>
Content-Type: application/json

{
  "ids": ["msg_001", "msg_002", "msg_003"]
}
```

## Routing Algorithm

```python
async def route_message(message, recipient):
    # 1. Same provider?
    if recipient.provider == self.provider:
        return await deliver_local(message, recipient)

    # 2. Different provider - federate
    return await deliver_federated(message, recipient)


async def deliver_local(message, recipient):
    # Try WebSocket
    if recipient.is_connected():
        if await deliver_websocket(message, recipient):
            return {"status": "delivered", "method": "websocket"}

    # Try Webhook
    if recipient.webhook_url:
        if await deliver_webhook(message, recipient):
            return {"status": "delivered", "method": "webhook"}

    # Fall back to relay
    await queue_for_relay(message, recipient)
    return {"status": "queued", "method": "relay"}
```

## Delivery Receipts

Senders can request delivery receipts:

```json
{
  "to": "recipient@tenant.provider",
  "subject": "Important message",
  "options": {
    "receipt": true
  },
  "payload": { ... }
}
```

When the message is delivered, the sender receives:

```json
{
  "type": "message.delivered",
  "data": {
    "id": "msg_1706648400_abc123",
    "to": "recipient@tenant.provider",
    "delivered_at": "2025-01-30T10:00:05Z",
    "method": "websocket"
  }
}
```

## Read Receipts

Recipients can send read receipts:

```http
POST /v1/messages/msg_1706648400_abc123/read
Authorization: Bearer <api_key>

Response:
{
  "read_receipt_sent": true
}
```

Sender receives:

```json
{
  "type": "message.read",
  "data": {
    "id": "msg_1706648400_abc123",
    "read_at": "2025-01-30T10:05:00Z"
  }
}
```

---

Previous: [04 - Messages](04-messages.md) | Next: [06 - Federation](06-federation.md)
