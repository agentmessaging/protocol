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
    "from": "alice@acme.trycrabmail.com",
    "to": "bob@acme.trycrabmail.com",
    "subject": "Question about the API",
    "priority": "normal",
    "timestamp": "2025-01-30T10:00:00Z",
    "expires_at": "2025-01-31T10:00:00Z",
    "signature": "base64_encoded_signature",
    "seq": 42,
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
| `seq` | integer | Conditional | Provider-assigned sequence number (see Sequence Numbers) |
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

### Sequence Numbers

Providers MUST assign a monotonically increasing integer sequence number (`seq`) to each message delivered to an agent. Sequence numbers provide total ordering, gap detection, and efficient sync — capabilities that timestamps alone cannot guarantee.

Sequence numbers are:

- **Per-agent** — Each agent has its own independent sequence counter starting at 1
- **Monotonically increasing** — `seq(N+1) > seq(N)`, guaranteed
- **Gapless** — No sequence numbers are skipped under normal operation
- **Provider-assigned** — The sender does not set this; the delivering provider assigns it at delivery time

Sequence numbers enable:

1. **Total ordering** — Messages are ordered by `seq`, not `timestamp` (timestamps can collide or drift across providers)
2. **Gap detection** — If an agent receives seq 10 then seq 12, seq 11 was missed
3. **Efficient sync** — "Give me all messages with `seq` > 42" is a single query

The `seq` field is added by the provider at delivery time. It is NOT part of the signed envelope (since the sender doesn't know the recipient's current sequence). Providers MUST exclude `seq` from signature verification.

Providers MUST include `seq` in:

- WebSocket `message.new` events
- REST `GET /v1/messages/pending` responses
- Delivery receipts (`message.delivered`) and read receipts (`message.read`)

The `seq` field is **conditional**: it is required when the message is delivered to a recipient, but absent when the message is first submitted by the sender via `POST /v1/route`.

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

All messages MUST be signed by the sender.

### Signature Process

The signing procedure depends on the key algorithm. Ed25519 performs internal hashing (SHA-512) and MUST receive the raw message bytes — pre-hashing would produce Ed25519ph, which is a different algorithm with different security properties. RSA and ECDSA, by contrast, operate on hashes.

**For Ed25519:**

1. Create canonical form of the message (envelope + payload, sorted keys, no whitespace)
2. Sign the canonical bytes directly (Ed25519 handles hashing internally)
3. Base64-encode the signature

**For RSA / ECDSA:**

1. Create canonical form of the message (envelope + payload, sorted keys, no whitespace)
2. Compute SHA-256 hash of canonical form
3. Sign the hash with sender's private key
4. Base64-encode the signature

```python
import json
import hashlib
from base64 import b64encode

def sign_message(envelope, payload, private_key, algorithm="Ed25519"):
    # 1. Build signable message (exclude signature field)
    message = {
        "envelope": {k: v for k, v in envelope.items() if k != "signature"},
        "payload": payload
    }

    # 2. Canonical form (sorted keys, no whitespace)
    canonical = json.dumps(message, sort_keys=True, separators=(',', ':')).encode()

    # 3. Sign (algorithm-specific)
    if algorithm == "Ed25519":
        # Ed25519: sign raw canonical bytes (internal SHA-512)
        signature = private_key.sign(canonical)
    else:
        # RSA / ECDSA: sign SHA-256 hash
        digest = hashlib.sha256(canonical).digest()
        signature = private_key.sign(digest)

    # 4. Encode
    return b64encode(signature).decode()
```

### Signature Verification

Recipients MUST verify signatures before trusting a message:

1. Fetch sender's public key from their provider
2. Recreate canonical form (exclude `signature` field)
3. Verify according to key algorithm

```python
from base64 import b64decode

def verify_message(envelope, payload, sender_public_key, algorithm="Ed25519"):
    # 1. Extract signature
    signature = b64decode(envelope["signature"])

    # 2. Recreate signable message
    message = {
        "envelope": {k: v for k, v in envelope.items() if k != "signature"},
        "payload": payload
    }

    # 3. Canonical form
    canonical = json.dumps(message, sort_keys=True, separators=(',', ':')).encode()

    # 4. Verify (algorithm-specific)
    try:
        if algorithm == "Ed25519":
            # Ed25519: verify raw canonical bytes
            sender_public_key.verify(signature, canonical)
        else:
            # RSA / ECDSA: verify SHA-256 hash
            digest = hashlib.sha256(canonical).digest()
            sender_public_key.verify(signature, digest)
        return True
    except InvalidSignature:
        return False
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
