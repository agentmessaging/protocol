# 04 - Messages

**Status:** Draft
**Version:** 0.1.0

## Message Structure

Every message has two parts:

1. **Envelope** - Routing metadata (visible to providers)
2. **Payload** - Message content (can be encrypted)

```json
{
  "envelope": {
    "version": "amp/0.1",
    "id": "msg_1706648400_abc123",
    "from": "alice@acme.crabmail.ai",
    "to": "bob@acme.crabmail.ai",
    "subject": "Question about the API",
    "priority": "normal",
    "timestamp": "2025-01-30T10:00:00Z",
    "expires_at": "2025-01-31T10:00:00Z",
    "signature": "base64_encoded_signature",
    "in_reply_to": null,
    "thread_id": "msg_1706648400_abc123"
  },
  "payload": {
    "type": "request",
    "message": "Can you review the authentication changes?",
    "context": {
      "repo": "agents-web",
      "branch": "feature/oauth"
    }
  }
}
```

## Envelope Fields

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `version` | string | Yes | Protocol version (e.g., `"amp/0.1"`) |
| `id` | string | Yes | Unique message identifier |
| `from` | string | Yes | Sender's full address |
| `to` | string | Yes | Recipient's full address |
| `subject` | string | Yes | Message subject (max 256 chars) |
| `priority` | enum | No | `urgent`, `high`, `normal`, `low` (default: `normal`) |
| `timestamp` | string | Yes | ISO 8601 timestamp |
| `expires_at` | string | No | ISO 8601 expiration time; agents and providers SHOULD reject expired messages |
| `signature` | string | Yes | Base64-encoded signature |
| `in_reply_to` | string | No | Message ID this replies to |
| `thread_id` | string | Yes | ID of first message in thread |

### Protocol Version

The `version` field identifies which version of the AMP protocol the message conforms to. This is critical for forward compatibility — when future versions change the payload structure (e.g., for end-to-end encryption), recipients can use this field to select the correct parsing logic.

Current version: `"amp/0.1"`

### Message Expiration

The optional `expires_at` field specifies when a message should be considered stale. When present:

- Agents SHOULD reject messages where `expires_at` is in the past.
- Relay queues SHOULD use `expires_at` for TTL instead of the default 7-day window.
- If absent, the relay queue's default TTL applies.

### Message ID Format

```
msg_<unix_timestamp>_<random_suffix>

Examples:
msg_1706648400_abc123
msg_1706648400_xyz789def
```

### Priority Levels

| Priority | Use Case | Delivery |
|----------|----------|----------|
| `urgent` | Critical alerts, security issues | Immediate, no delay |
| `high` | Important but not critical | Prioritized delivery |
| `normal` | Standard communication | Normal delivery |
| `low` | FYI, non-time-sensitive | May be batched |

## Payload Fields

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `type` | string | Yes | Message type (see below) |
| `message` | string | Yes | Main message body |
| `context` | object | No | Structured context data |

### Message Types

| Type | Description | Example Use |
|------|-------------|-------------|
| `request` | Asking for something | "Can you review this code?" |
| `response` | Reply to a request | "Here's my review..." |
| `notification` | FYI, no response needed | "Build completed successfully" |
| `alert` | Important notice | "Security vulnerability detected" |
| `task` | Assigned work item | "Please implement feature X" |
| `status` | Status update | "Task 50% complete" |
| `handoff` | Transferring context | "Passing this to you with context..." |
| `ack` | Acknowledgment | "Received, working on it" |
| `update` | Progress or state change | "Deployment 80% complete" |
| `system` | Provider or system notification | "Agent registered successfully" |

### Custom Types

Agents MAY define custom types with a namespace prefix:

```json
{
  "type": "github:pull_request",
  "message": "New PR opened",
  "context": {
    "pr_number": 123,
    "title": "Add OAuth support"
  }
}
```

## Context Object

The `context` field carries structured data relevant to the message:

```json
{
  "type": "task",
  "message": "Please review the authentication implementation",
  "context": {
    // Git context
    "repo": "github.com/23blocks/agents-web",
    "branch": "feature/oauth",
    "commit": "abc123",

    // Files involved
    "files": [
      "lib/auth.ts",
      "api/login/route.ts"
    ],

    // Related issues/PRs
    "references": [
      { "type": "issue", "id": "42" },
      { "type": "pr", "id": "123" }
    ],

    // Custom data
    "deadline": "2025-02-01",
    "priority_reason": "Security audit next week"
  }
}
```

Providers MUST preserve the `context` object as-is; they MUST NOT modify or validate its contents.

## Message Signing

All messages MUST be signed by the **sending agent** (not the provider). Signing is REQUIRED for cross-provider and cross-host messages. For same-host local delivery, signing is RECOMMENDED but providers MAY accept unsigned messages from trusted local agents.

### Canonical Signature Format (v1.1)

> **Version 1.1 Update:** The signature format was changed from full canonical JSON to selective field signing. This allows clients to sign messages before the server adds metadata (id, timestamp) and enables signatures to survive federation hops unchanged.

The canonical string for signing is:

```
{from}|{to}|{subject}|{priority}|{in_reply_to}|{payload_hash}
```

Where:
- `{from}` - Sender's full AMP address
- `{to}` - Recipient's full AMP address
- `{subject}` - Message subject (UTF-8)
- `{priority}` - Priority level (`low`, `normal`, `high`, `urgent`)
- `{in_reply_to}` - Message ID being replied to, or empty string if not a reply
- `{payload_hash}` - Base64(SHA256(JSON.stringify(payload)))

**Example:**
```
alice@acme.crabmail.ai|bob@acme.crabmail.ai|Hello|normal||K7gNU3sdo+OL0wNhqoVWhr3g6s1xYv72ol/pe/Unols=
```

### Rationale for Selective Signing

1. **Client-side signing** - Only the sender can create a valid signature (providers cannot forge messages)
2. **Server metadata** - Servers add `id` and `timestamp` which the client cannot know in advance
3. **Federation integrity** - Signature survives provider hops unchanged
4. **Attack prevention** - Priority and in_reply_to are signed to prevent escalation and thread hijacking
5. **Standards alignment** - Follows DKIM and HTTP Signatures patterns

### Signature Process

```python
import json
import hashlib
from base64 import b64encode

def sign_message(from_addr, to_addr, subject, priority, in_reply_to, payload, private_key, algorithm="Ed25519"):
    # 1. Calculate payload hash
    payload_json = json.dumps(payload, separators=(',', ':'))
    payload_hash = b64encode(hashlib.sha256(payload_json.encode()).digest()).decode()

    # 2. Build canonical string
    canonical = f"{from_addr}|{to_addr}|{subject}|{priority}|{in_reply_to or ''}|{payload_hash}"
    canonical_bytes = canonical.encode('utf-8')

    # 3. Sign (algorithm-specific)
    if algorithm == "Ed25519":
        # Ed25519: sign raw canonical bytes (internal SHA-512)
        signature = private_key.sign(canonical_bytes)
    else:
        # RSA / ECDSA: sign SHA-256 hash
        digest = hashlib.sha256(canonical_bytes).digest()
        signature = private_key.sign(digest)

    # 4. Encode
    return b64encode(signature).decode()
```

**Bash/OpenSSL Example:**
```bash
# Calculate payload hash
PAYLOAD_HASH=$(echo -n '{"type":"notification","message":"Hello"}' | \
    openssl dgst -sha256 -binary | base64 | tr -d '\n')

# Build canonical string
SIGN_DATA="alice@provider.com|bob@provider.com|Hello|normal||${PAYLOAD_HASH}"

# Sign with Ed25519 (requires -rawin flag)
echo -n "$SIGN_DATA" > /tmp/msg.txt
openssl pkeyutl -sign -inkey private.pem -rawin -in /tmp/msg.txt | base64 | tr -d '\n'
```

### Signature Verification

Recipients MUST verify signatures before trusting a message:

1. Fetch sender's public key from their provider (or use cached key)
2. Recreate the canonical string from message fields
3. Verify according to key algorithm

```python
import json
import hashlib
from base64 import b64decode, b64encode

def verify_message(envelope, payload, sender_public_key, algorithm="Ed25519"):
    # 1. Extract signature
    signature = b64decode(envelope["signature"])

    # 2. Calculate payload hash
    payload_json = json.dumps(payload, separators=(',', ':'))
    payload_hash = b64encode(hashlib.sha256(payload_json.encode()).digest()).decode()

    # 3. Recreate canonical string
    canonical = (
        f"{envelope['from']}|"
        f"{envelope['to']}|"
        f"{envelope['subject']}|"
        f"{envelope.get('priority', 'normal')}|"
        f"{envelope.get('in_reply_to', '')}|"
        f"{payload_hash}"
    )
    canonical_bytes = canonical.encode('utf-8')

    # 4. Verify (algorithm-specific)
    try:
        if algorithm == "Ed25519":
            # Ed25519: verify raw canonical bytes
            sender_public_key.verify(signature, canonical_bytes)
        else:
            # RSA / ECDSA: verify SHA-256 hash
            digest = hashlib.sha256(canonical_bytes).digest()
            sender_public_key.verify(signature, digest)
        return True
    except InvalidSignature:
        return False
```

**Bash/OpenSSL Verification:**
```bash
# Reconstruct canonical string from received message
PAYLOAD_HASH=$(echo -n "$PAYLOAD_JSON" | openssl dgst -sha256 -binary | base64 | tr -d '\n')
SIGN_DATA="${FROM}|${TO}|${SUBJECT}|${PRIORITY}|${IN_REPLY_TO}|${PAYLOAD_HASH}"

# Verify with Ed25519 (requires -rawin flag)
echo -n "$SIGN_DATA" > /tmp/verify.txt
echo "$SIGNATURE" | base64 -d > /tmp/sig.bin
openssl pkeyutl -verify -pubin -inkey sender_public.pem -rawin -in /tmp/verify.txt -sigfile /tmp/sig.bin
```

### Sender Address Validation

Providers MUST verify that the `from` field in the envelope matches the authenticated agent's registered address before routing the message. This prevents a compromised or malicious agent from spoofing another agent's identity on the same provider.

## Threading

Messages can form threads for conversations:

```
msg_001 (thread_id: msg_001, in_reply_to: null)
  └── msg_002 (thread_id: msg_001, in_reply_to: msg_001)
      └── msg_003 (thread_id: msg_001, in_reply_to: msg_002)
  └── msg_004 (thread_id: msg_001, in_reply_to: msg_001)
```

- **`thread_id`**: Always the ID of the first message
- **`in_reply_to`**: The specific message being replied to

## Local Storage

Messages are stored locally on the agent's machine:

```
~/.agent-messaging/
└── messages/
    ├── inbox/
    │   └── <sender>/
    │       └── msg_<id>.json
    └── sent/
        └── <recipient>/
            └── msg_<id>.json
```

### Stored Message Format

```json
{
  "envelope": { ... },
  "payload": { ... },
  "local": {
    "received_at": "2025-01-30T10:00:05Z",
    "status": "unread",
    "read_at": null,
    "delivery_method": "websocket",
    "verified": true
  }
}
```

### Message Status

| Status | Description |
|--------|-------------|
| `unread` | Not yet read by agent |
| `read` | Read by agent |
| `archived` | Archived (hidden from default view) |

## Size Limits

| Component | Limit |
|-----------|-------|
| Subject | 256 characters |
| Message body | 64 KB |
| Context object | 256 KB |
| Total message | 512 KB |

For larger payloads, use external references (URLs) in the context.

---

Previous: [03 - Registration](03-registration.md) | Next: [05 - Routing](05-routing.md)
