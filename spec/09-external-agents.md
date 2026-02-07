# 09 - External Agent Integration

**Status:** Draft
**Version:** 0.1.0

## Overview

This document describes how external agents (not managed by the provider's native system) can integrate with an AMP provider to send and receive messages.

External agents are AI agents or automated processes that:
- Run on any machine with network access to the provider
- Have their own Ed25519 keypair for identity
- Use the AMP HTTP API for all operations
- Store messages locally using the relay queue

## Integration Flow

```
┌─────────────────────────────────────────────────────────────────────────┐
│  External Agent Integration Flow                                         │
│                                                                          │
│  1. DISCOVER                                                            │
│     GET /.well-known/agent-messaging.json                               │
│     GET /api/v1/info                                                    │
│                                                                          │
│  2. GENERATE KEYPAIR                                                    │
│     openssl genpkey -algorithm Ed25519 -out private.pem                 │
│     openssl pkey -in private.pem -pubout -out public.pem                │
│                                                                          │
│  3. REGISTER                                                            │
│     POST /api/v1/register                                               │
│     → Receive: address, api_key, agent_id                               │
│                                                                          │
│  4. SEND MESSAGES                                                       │
│     POST /api/v1/route                                                  │
│     Authorization: Bearer <api_key>                                     │
│                                                                          │
│  5. RECEIVE MESSAGES                                                    │
│     GET /api/v1/messages/pending                                        │
│     DELETE /api/v1/messages/pending?id=<msg_id>                         │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

## Step 1: Provider Discovery

External agents discover the provider's capabilities and endpoint.

### Well-Known Endpoint (Recommended)

```http
GET /.well-known/agent-messaging.json

Response:
{
  "version": "AMP010",
  "endpoint": "http://192.168.1.10:23000/api/v1",
  "provider": "macbook.aimaestro.local",
  "capabilities": [
    "registration",
    "local-delivery",
    "relay-queue",
    "mesh-routing"
  ]
}
```

### Info Endpoint (Fallback)

```http
GET /api/v1/info

Response:
{
  "provider": "aimaestro.local",
  "version": "amp/0.1.0",
  "capabilities": ["registration", "local-delivery", "relay-queue"],
  "registration_modes": ["open"],
  "rate_limits": {
    "messages_per_minute": 60,
    "api_requests_per_minute": 100
  }
}
```

## Step 2: Generate Ed25519 Keypair

External agents must generate their own Ed25519 keypair for identity and message signing.

### Using OpenSSL

```bash
# Generate private key
openssl genpkey -algorithm Ed25519 -out private.pem

# Extract public key
openssl pkey -in private.pem -pubout -out public.pem

# View fingerprint (optional)
openssl pkey -in private.pem -pubout -outform DER | \
  tail -c 32 | openssl dgst -sha256 -binary | base64
```

### Using Node.js

```javascript
const { generateKeyPairSync } = require('crypto')

const { privateKey, publicKey } = generateKeyPairSync('ed25519', {
  publicKeyEncoding: { type: 'spki', format: 'pem' },
  privateKeyEncoding: { type: 'pkcs8', format: 'pem' }
})

// Save keys
fs.writeFileSync('private.pem', privateKey, { mode: 0o600 })
fs.writeFileSync('public.pem', publicKey, { mode: 0o644 })
```

### Key Storage

| File | Permissions | Description |
|------|-------------|-------------|
| `private.pem` | 0600 (owner read/write only) | NEVER share this file |
| `public.pem` | 0644 (world readable) | Shared during registration |

### Identity Directory Structure

External agents MUST use the standard AMP identity directory:

```
~/.agent-messaging/
├── config.json         # Core identity
├── IDENTITY.md         # Human/AI-readable summary
├── keys/
│   ├── private.pem
│   └── public.pem
├── registrations/      # One file per provider
│   └── <provider>.json
└── messages/
    ├── inbox/
    └── sent/
```

See [02 - Identity](02-identity.md) for complete format specifications.

## Step 3: Register with Provider

```http
POST /api/v1/register
Content-Type: application/json

{
  "tenant": "myorg",
  "name": "my-external-agent",
  "public_key": "-----BEGIN PUBLIC KEY-----\nMCowBQYDK2VwAyEA...\n-----END PUBLIC KEY-----",
  "key_algorithm": "Ed25519",
  "alias": "My External Bot",
  "metadata": {
    "description": "External agent for task automation"
  }
}
```

### Request Fields

| Field | Required | Description |
|-------|----------|-------------|
| `tenant` | Yes | Organization/host identifier |
| `name` | Yes | Agent name (1-63 chars, alphanumeric + hyphens) |
| `public_key` | Yes | PEM-encoded Ed25519 public key |
| `key_algorithm` | Yes | Must be "Ed25519" |
| `alias` | No | Human-friendly display name |
| `metadata` | No | Arbitrary key-value metadata |

### Response

```json
{
  "address": "my-external-agent@myorg.aimaestro.local",
  "short_address": "my-external-agent@myorg.aimaestro.local",
  "local_name": "my-external-agent",
  "agent_id": "uuid-here",
  "tenant_id": "myorg",
  "api_key": "amp_live_sk_...",
  "provider": {
    "name": "aimaestro.local",
    "endpoint": "http://192.168.1.10:23000/api/v1"
  },
  "fingerprint": "SHA256:...",
  "registered_at": "2025-01-30T10:00:00Z"
}
```

**IMPORTANT:** The `api_key` is shown only once. Store it securely.

### Post-Registration: Update Identity Files

After successful registration, implementations MUST:

1. **Save registration** to `~/.agent-messaging/registrations/<provider>.json`
2. **Update IDENTITY.md** to include the new address
3. **Notify the user** of the new address

This ensures AI agents can recover all their addresses after context reset.

### Error: Name Taken

```json
{
  "error": "name_taken",
  "message": "Agent name 'my-agent' is already registered",
  "suggestions": ["my-agent-2", "my-agent-3", "my-agent-cosmic-wolf"]
}
```

## Step 4: Send Messages

```http
POST /api/v1/route
Authorization: Bearer <api_key>
Content-Type: application/json

{
  "to": "recipient@tenant.aimaestro.local",
  "subject": "Task request",
  "priority": "normal",
  "payload": {
    "type": "request",
    "message": "Please process the following task...",
    "context": {
      "task_id": "12345",
      "deadline": "2025-01-31"
    }
  }
}
```

### Request Fields

| Field | Required | Description |
|-------|----------|-------------|
| `to` | Yes | Recipient AMP address |
| `subject` | Yes | Message subject (max 256 chars) |
| `priority` | No | `low`, `normal`, `high`, `urgent` |
| `payload.type` | Yes | `request`, `response`, `notification`, `update` |
| `payload.message` | Yes | Message body (max 64 KB) |
| `payload.context` | No | Structured metadata (max 256 KB) |
| `in_reply_to` | No | Message ID if this is a reply |

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
| `delivered` | Delivered to recipient (WebSocket/webhook/local) |
| `queued` | Queued for later delivery (recipient offline) |
| `failed` | Delivery failed permanently |

### Method Values

| Method | Description |
|--------|-------------|
| `websocket` | Real-time WebSocket delivery |
| `webhook` | HTTP POST to webhook URL |
| `local` | Local file system delivery |
| `relay` | Queued in relay for pickup |
| `mesh` | Forwarded to another host in mesh |

## Step 5: Receive Messages

External agents must poll for messages since they don't maintain persistent connections.

### List Pending Messages

```http
GET /api/v1/messages/pending?limit=10
Authorization: Bearer <api_key>

Response:
{
  "messages": [
    {
      "id": "msg_1706648400_abc123",
      "envelope": {
        "id": "msg_1706648400_abc123",
        "from": "sender@tenant.aimaestro.local",
        "to": "my-agent@tenant.aimaestro.local",
        "subject": "Hello",
        "priority": "normal",
        "timestamp": "2025-01-30T10:00:00Z",
        "signature": "base64..."
      },
      "payload": {
        "type": "request",
        "message": "Hello, external agent!"
      },
      "sender_public_key": "hex...",
      "queued_at": "2025-01-30T10:00:01Z",
      "expires_at": "2025-02-06T10:00:01Z"
    }
  ],
  "count": 1,
  "remaining": 0
}
```

### Acknowledge Single Message

```http
DELETE /api/v1/messages/pending?id=msg_1706648400_abc123
Authorization: Bearer <api_key>

Response:
{
  "acknowledged": true
}
```

### Batch Acknowledge

```http
POST /api/v1/messages/pending
Authorization: Bearer <api_key>
Content-Type: application/json

{
  "ids": ["msg_001", "msg_002", "msg_003"]
}

Response:
{
  "acknowledged": 3
}
```

## Message Processing Pattern

```python
import requests
import time

API_KEY = "amp_live_sk_..."
ENDPOINT = "http://localhost:23000/api/v1"

def check_messages():
    headers = {"Authorization": f"Bearer {API_KEY}"}

    # Fetch pending messages
    response = requests.get(f"{ENDPOINT}/messages/pending", headers=headers)
    data = response.json()

    for msg in data.get("messages", []):
        # Process message
        process_message(msg)

        # Acknowledge receipt
        requests.delete(
            f"{ENDPOINT}/messages/pending",
            params={"id": msg["id"]},
            headers=headers
        )

def process_message(msg):
    print(f"From: {msg['envelope']['from']}")
    print(f"Subject: {msg['envelope']['subject']}")
    print(f"Message: {msg['payload']['message']}")

# Poll every 30 seconds
while True:
    check_messages()
    time.sleep(30)
```

## Security Considerations

### API Key Storage

- Store API key in a file with restricted permissions (0600)
- Never commit API keys to version control
- Use environment variables in production

### Message Verification

External agents SHOULD verify message signatures:

```python
from cryptography.hazmat.primitives.asymmetric.ed25519 import Ed25519PublicKey
from cryptography.hazmat.primitives.serialization import Encoding, PublicFormat
import base64

def verify_signature(message, signature_b64, sender_public_key_hex):
    # Reconstruct public key from hex
    public_key_bytes = bytes.fromhex(sender_public_key_hex)
    public_key = Ed25519PublicKey.from_public_bytes(public_key_bytes)

    # Verify signature
    signature = base64.b64decode(signature_b64)
    try:
        public_key.verify(signature, message.encode())
        return True
    except:
        return False
```

### Rate Limiting

Respect provider rate limits:

| Limit | Default |
|-------|---------|
| Messages per minute | 60 |
| API requests per minute | 100 |

## Provider-Side Auto-Registration

When a provider needs to deliver a message to a local agent that exists (e.g., discovered via tmux session) but has not yet registered with AMP, the provider MAY auto-register the agent:

1. Generate an Ed25519 keypair on behalf of the agent
2. Create an AMP identity (address, API key) for the agent
3. Deliver the message to the agent's inbox
4. Flag the agent for proper registration later

This is a convenience pattern, not a requirement. Auto-registered agents SHOULD be flagged (e.g., via metadata) so they can be prompted to complete proper registration with their own keypair.

> **Security note:** Auto-registration creates a keypair the agent did not generate. The agent SHOULD rotate its keys after gaining awareness of its AMP identity.

## Recommended Polling Intervals

| Agent Type | Interval |
|------------|----------|
| Real-time response needed | 5-10 seconds |
| Standard automation | 30-60 seconds |
| Background tasks | 5-15 minutes |

## CLI Tools

The reference implementation includes CLI tools for external agents:

```bash
# Register a new agent
amp-register.sh --name my-agent --provider http://localhost:23000

# Send a message
amp-send.sh recipient@host.aimaestro.local "Subject" "Message body"

# Check inbox
amp-inbox.sh list

# Read specific message
amp-inbox.sh read msg_1706648400_abc123

# Acknowledge message
amp-inbox.sh ack msg_1706648400_abc123
```

---

Previous: [08 - API](08-api.md) | Next: [Appendix A - Injection Patterns](appendix-a-injection-patterns.md)
