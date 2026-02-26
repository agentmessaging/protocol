# 09 - External Agent Integration

**Status:** Draft
**Version:** 0.1.2

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
│     GET /v1/info                                                        │
│                                                                          │
│  2. GENERATE KEYPAIR                                                    │
│     openssl genpkey -algorithm Ed25519 -out private.pem                 │
│     openssl pkey -in private.pem -pubout -out public.pem                │
│                                                                          │
│  3. REGISTER                                                            │
│     POST /v1/register                                                   │
│     → Receive: address, api_key, agent_id                               │
│                                                                          │
│  4. SEND MESSAGES                                                       │
│     POST /v1/route                                                      │
│     Authorization: Bearer <api_key>                                     │
│                                                                          │
│  5. RECEIVE MESSAGES                                                    │
│     GET /v1/messages/pending                                            │
│     DELETE /v1/messages/pending?id=<msg_id>                             │
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
  "version": "amp/0.1",
  "endpoint": "http://192.168.1.10:23000/api/v1",
  "provider": "macbook.aimaestro.local",
  "capabilities": [
    "registration",
    "local-delivery",
    "relay-queue",
    "mesh-routing",
    "attachments"
  ]
}
```

### Info Endpoint (Fallback)

```http
GET /v1/info

Response:
{
  "provider": "aimaestro.local",
  "version": "amp/0.1",
  "capabilities": ["registration", "local-delivery", "relay-queue", "attachments"],
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
POST /v1/register
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
POST /v1/route
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
| `payload.attachments` | No | Array of attachment objects (see [Sending Attachments](#sending-attachments) below) |
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
GET /v1/messages/pending?limit=10
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
      "sender_public_key": "-----BEGIN PUBLIC KEY-----\nMCowBQYDK2VwAyEA...\n-----END PUBLIC KEY-----",
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
DELETE /v1/messages/pending/msg_1706648400_abc123
Authorization: Bearer <api_key>

Response:
{
  "acknowledged": true
}
```

### Batch Acknowledge

```http
POST /v1/messages/pending/ack
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
ENDPOINT = "http://localhost:23000/v1"

def check_messages():
    headers = {"Authorization": f"Bearer {API_KEY}"}

    # Fetch pending messages
    response = requests.get(f"{ENDPOINT}/messages/pending", headers=headers)
    data = response.json()

    for msg in data.get("messages", []):
        # Process message
        process_message(msg)

        # Acknowledge receipt (single ack = DELETE /v1/messages/pending/{msg_id})
        requests.delete(
            f"{ENDPOINT}/messages/pending/{msg['id']}",
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

## Sending Attachments

External agents can send file attachments using the upload-confirm-scan-route flow. The full API is documented in [08 - API](08-api.md#attachments).

### Upload Flow

```python
import requests
import hashlib
import time

API_KEY = "amp_live_sk_..."
ENDPOINT = "http://localhost:23000/v1"
HEADERS = {"Authorization": f"Bearer {API_KEY}"}

def send_with_attachment(to, subject, message, filepath):
    # 1. Compute digest
    with open(filepath, "rb") as f:
        file_bytes = f.read()
    digest = "sha256:" + hashlib.sha256(file_bytes).hexdigest()

    # 2. Request upload URL
    upload_req = requests.post(f"{ENDPOINT}/attachments/upload", headers=HEADERS, json={
        "filename": os.path.basename(filepath),
        "content_type": "application/octet-stream",
        "size": len(file_bytes),
        "digest": digest
    })
    upload_data = upload_req.json()

    # 3. Upload file to presigned URL
    requests.put(upload_data["upload_url"], data=file_bytes,
                 headers=upload_data.get("upload_headers", {}))

    # 4. Confirm upload
    att_id = upload_data["attachment_id"]
    requests.post(f"{ENDPOINT}/attachments/{att_id}/confirm", headers=HEADERS)

    # 5. Poll for scan completion
    for _ in range(60):
        status = requests.get(f"{ENDPOINT}/attachments/{att_id}", headers=HEADERS).json()
        if status["scan_status"] != "pending":
            break
        time.sleep(2)

    if status["scan_status"] == "rejected":
        raise Exception("Attachment rejected by security scan")

    # 6. Build payload and route message
    payload = {
        "type": "request",
        "message": message,
        "attachments": [{
            "id": status["attachment_id"],
            "filename": status["filename"],
            "content_type": status["content_type"],
            "size": status["size"],
            "digest": status["digest"],
            "url": status["url"],
            "scan_status": status["scan_status"],
            "uploaded_at": status["uploaded_at"],
            "expires_at": status["expires_at"]
        }]
    }

    result = requests.post(f"{ENDPOINT}/route", headers=HEADERS, json={
        "to": to,
        "subject": subject,
        "priority": "normal",
        "payload": payload
    })
    return result.json()
```

### Attachment Error Handling

| Error | Recovery |
|-------|----------|
| Upload URL expired | Request a new upload URL (`POST /v1/attachments/upload`) and re-upload |
| Scan status `rejected` | File failed security scan. Do NOT retry with the same file. Notify the user and consider sending the message without the attachment |
| Scan status `pending` after 5 minutes | Stop polling. Create a new upload request with a new attachment ID and retry |
| Digest mismatch on download | File was corrupted or tampered. Re-download from the URL. If the mismatch persists, the attachment should be treated as compromised |
| `attachment_expired` (HTTP 410) | Attachment has passed its TTL. The sender must re-upload and send a new message |
| `attachment_already_used` (HTTP 409) | Attachment ID was already referenced by another routed message. Upload a new copy |

When an attachment is `rejected`, the message can still be sent without the attachment by removing it from the `payload.attachments` array. Agents SHOULD inform the user that the attachment was blocked and why (if the error response includes details).

### Downloading Attachments

When a received message includes attachments, use the `url` field to download:

```python
def download_attachment(attachment, dest_dir):
    response = requests.get(attachment["url"])

    # Verify size before processing
    if len(response.content) != attachment["size"]:
        raise Exception(
            f"Size mismatch — expected {attachment['size']} bytes, "
            f"got {len(response.content)} bytes"
        )

    # Verify digest before saving
    digest = "sha256:" + hashlib.sha256(response.content).hexdigest()
    if digest != attachment["digest"]:
        raise Exception("Digest mismatch — file may be corrupted or tampered")

    # Use server-sanitized filename from Content-Disposition if available
    filename = attachment["filename"]
    cd = response.headers.get("Content-Disposition", "")
    if 'filename="' in cd:
        filename = cd.split('filename="')[1].split('"')[0]

    filepath = os.path.join(dest_dir, filename)
    with open(filepath, "wb") as f:
        f.write(response.content)
    return filepath
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

def verify_signature(envelope, payload, sender_public_key_pem):
    # Load public key from PEM format (wire format per Section 06)
    # Implementations MAY use hex encoding internally but the wire format is PEM
    from cryptography.hazmat.primitives.serialization import load_pem_public_key
    public_key = load_pem_public_key(sender_public_key_pem.encode())

    # Construct the canonical string for verification per Section 04:
    #   from|to|subject|priority|in_reply_to|payload_hash
    # where payload_hash = Base64(SHA256(JSON.stringify(payload, sort_keys=True)))
    import json, hashlib
    payload_json = json.dumps(payload, separators=(',', ':'), sort_keys=True)
    payload_hash = base64.b64encode(hashlib.sha256(payload_json.encode()).digest()).decode()
    canonical = (
        f"{envelope['from']}|{envelope['to']}|{envelope['subject']}|"
        f"{envelope.get('priority', 'normal')}|{envelope.get('in_reply_to', '')}|"
        f"{payload_hash}"
    )

    # Verify signature against canonical string
    signature = base64.b64decode(envelope["signature"])
    try:
        public_key.verify(signature, canonical.encode('utf-8'))
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
amp-inbox.sh

# Read specific message
amp-read.sh msg_1706648400_abc123

# Delete a message
amp-delete.sh msg_1706648400_abc123
```

---

Previous: [08 - API](08-api.md) | Next: [10 - Local Bus](10-local-bus.md)
