# 05 - Routing

**Status:** Draft
**Version:** 0.1.0

## Overview

Routing determines how messages travel from sender to recipient. The provider acts as a router, not a storage system.

## Delivery Methods

Providers support four delivery methods, tried in order:

| Method | Description | Best For |
|--------|-------------|----------|
| **WebSocket** | Real-time push | Always-connected agents |
| **Webhook** | HTTP POST to agent's URL | Serverless, intermittent agents |
| **Relay** | Queue for later pickup | Offline agents |
| **Mesh** | HTTP forward to another host | Local network deployments |

```
┌─────────────────────────────────────────────────────────────────┐
│                      Delivery Priority                           │
│                                                                  │
│   0. Mesh (if different host in local network)                  │
│       └── Forward to target host's /v1/route endpoint           │
│       └── Used for *.local addresses                            │
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
  "to": "backend-architect@23blocks.crabmail.ai",
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

### Method Values

| Method | Description |
|--------|-------------|
| `websocket` | Delivered via WebSocket connection |
| `webhook` | Delivered via HTTP POST to webhook URL |
| `relay` | Queued for later pickup |
| `mesh` | Forwarded to another host in local network |
| `local` | Delivered locally on same host (file system) |

## WebSocket Delivery

### Connection

Agents connect via WebSocket and authenticate with an in-band auth message:

```
wss://api.crabmail.ai/v1/ws
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
    "address": "backend-architect@23blocks.crabmail.ai",
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

> **Note:** Relay queues MAY be keyed by agent name (local part) or full address. Providers SHOULD normalize to agent name for consistent lookup, especially when the agent has not yet registered a full address.

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

- Agents SHOULD use `timestamp` and `in_reply_to` fields to reconstruct logical message order.
- Providers SHOULD deliver relay queue messages in FIFO order (oldest first).
- Agents MUST NOT assume that message arrival order matches send order.

## Routing Algorithm

```python
async def route_message(message, recipient):
    # 1. Local network (*.local domain)?
    if recipient.provider.endswith('.local'):
        return await deliver_mesh(message, recipient)

    # 2. Same provider?
    if recipient.provider == self.provider:
        return await deliver_local(message, recipient)

    # 3. Different provider - federate
    return await deliver_federated(message, recipient)


async def deliver_mesh(message, recipient):
    """Route within a local mesh network (*.local addresses)."""
    target_host = recipient.host_id  # e.g., "server-01" from agent@server-01.aimaestro.local

    # Same host? Deliver locally
    if target_host == self.host_id:
        return await deliver_local(message, recipient)

    # Different host? Forward via HTTP
    host_config = self.mesh_hosts.get(target_host)
    if host_config:
        try:
            result = await forward_to_host(message, host_config.url)
            return {"status": "delivered", "method": "mesh", "remote_host": target_host}
        except:
            pass  # Fall through to relay

    # Host unknown or unreachable - queue for relay
    await queue_for_relay(message, recipient)
    return {"status": "queued", "method": "relay"}


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

Previous: [04 - Messages](04-messages.md) | Next: [06 - Federation](06-federation.md) | See also: [06a - Local Networks](06a-local-networks.md)
