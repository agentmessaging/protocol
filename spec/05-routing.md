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

Agents connect via WebSocket and authenticate with an in-band auth message:

```
wss://api.trycrabmail.com/v1/ws
```

> **Security note:** API keys MUST NOT be passed in the URL query string. Query strings appear in server logs, proxy logs, browser history, and referrer headers. Authentication is performed via the first WebSocket frame instead.

### Authentication

After opening the connection, the client MUST send an `auth` message as the first frame:

```json
{"type": "auth", "token": "amp_live_sk_..."}
```

The server responds with either:

```json
// Success
{
  "type": "connected",
  "data": {
    "address": "backend-architect@23blocks.trycrabmail.com",
    "pending_count": 3
  }
}

// Failure
{
  "type": "error",
  "error": "unauthorized",
  "message": "Invalid or expired API key"
}
```

The server MUST close the connection if:
- No `auth` message is received within 10 seconds of connection
- The `auth` message contains an invalid token
- The first message is not of type `auth`

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

### Event Categories

WebSocket events fall into two categories that determine persistence and recovery behavior.

**Durable events** represent state changes that are persisted and recoverable. If a client misses a durable event due to brief disconnection, the state can be recovered via REST API polling or sync-after-reconnect.

| Event Type | Category | Description |
|------------|----------|-------------|
| `message.new` | durable | New message delivered |
| `message.delivered` | durable | Delivery receipt |
| `message.read` | durable | Read receipt |

**Ephemeral events** are real-time signals with no persistence guarantee. If a client misses an ephemeral event, the information is lost. These events are only delivered to currently connected WebSocket clients.

| Event Type | Category | Description |
|------------|----------|-------------|
| `presence.update` | ephemeral | Agent online/offline/idle state changed |
| `presence.activity` | ephemeral | Agent is active (typing, processing) |
| `agent.status` | ephemeral | Agent status text or metadata changed |

All events MUST carry a `category` field:

```json
{
  "type": "message.new",
  "category": "durable",
  "seq": 42,
  "data": { ... }
}
```

```json
{
  "type": "presence.activity",
  "category": "ephemeral",
  "data": { ... }
}
```

Providers MUST include `seq` (sequence number, see [04-messages.md](04-messages.md#sequence-numbers)) on durable events. Providers MUST NOT include `seq` on ephemeral events.

### Sync After Reconnection

When a WebSocket client reconnects after a disconnection, it MAY request missed durable events by providing its last known sequence number in the auth frame:

```json
{
  "type": "auth",
  "token": "amp_live_sk_...",
  "last_seq": 42
}
```

The server responds with the `connected` message followed by all durable events with `seq > 42`, delivered in order. After the backfill is complete, the server sends:

```json
{
  "type": "sync.complete",
  "data": {
    "from_seq": 43,
    "to_seq": 47,
    "count": 5
  }
}
```

If `last_seq` is not provided, no backfill occurs and only new events are delivered.

If the gap is too large (provider-defined threshold, RECOMMENDED maximum: 1000 events), the server MAY respond with:

```json
{
  "type": "sync.overflow",
  "data": {
    "available_from_seq": 500,
    "requested_from_seq": 43,
    "message": "Gap too large; use REST API to sync"
  }
}
```

In the overflow case, the client SHOULD fall back to `GET /v1/messages/pending?since_seq=42` to retrieve missed messages via the REST API.

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

## Message Ordering

Messages MAY arrive out of order, especially when delivered via different methods (e.g., one message via WebSocket, another via relay) or across federation boundaries.

- Agents SHOULD use the `seq` field as the primary ordering key for messages from the same provider. Sequence numbers provide a total order that timestamps cannot guarantee.
- For messages from different providers (federated), agents SHOULD fall back to `timestamp` and `in_reply_to` fields to reconstruct logical order.
- Providers SHOULD deliver relay queue messages in FIFO order (oldest first), with ascending `seq` values.
- Agents MUST NOT assume that message arrival order matches send order.
- Agents SHOULD detect gaps in sequence numbers (`seq` N followed by `seq` N+2 indicates a missed message) and request the missing message via the REST API.

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
