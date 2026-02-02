# 09 - Channel Bridging

**Status:** Draft
**Version:** 0.1.0

## Overview

Channel bridges connect external communication platforms (email, SMS, chat services) to the AMP network. A bridge is a specialized AMP agent that translates between an external protocol and AMP messages, enabling AI agents to communicate with users and systems on any platform.

```
┌──────────────┐     ┌─────────────────┐     ┌───────────────┐
│  WhatsApp    │────▸│  WhatsApp       │────▸│  AMP Agent    │
│  User        │◂────│  Bridge         │◂────│  (recipient)  │
└──────────────┘     └─────────────────┘     └───────────────┘

┌──────────────┐     ┌─────────────────┐     ┌───────────────┐
│  Email       │────▸│  Email          │────▸│  AMP Agent    │
│  Sender      │◂────│  Bridge         │◂────│  (recipient)  │
└──────────────┘     └─────────────────┘     └───────────────┘

┌──────────────┐     ┌─────────────────┐     ┌───────────────┐
│  Slack       │────▸│  Slack          │────▸│  AMP Agent    │
│  User        │◂────│  Bridge         │◂────│  (recipient)  │
└──────────────┘     └─────────────────┘     └───────────────┘
```

Channel bridging is a core differentiator of the AMP protocol. While agent-to-agent messaging operates within the AMP network, channel bridges extend the protocol's reach to the entire external communication landscape.

## Bridge Agent Requirements

A channel bridge MUST:

1. Register as a regular AMP agent with `agent_type: "bridge"` in registration metadata
2. Include channel metadata in all forwarded messages (see Channel Metadata)
3. Apply content security scanning to all inbound external content (see [07-security.md](07-security.md))
4. Wrap untrusted external content in `<external-content>` tags per spec 07
5. Translate replies from AMP agents back to the originating external channel
6. Maintain connection state with the external platform and report status via health endpoints

A channel bridge SHOULD:

1. Support bidirectional communication (receive from and send to external channels)
2. Preserve threading context across the bridge (e.g., Slack thread → AMP thread → Slack reply)
3. Track message delivery status on the external platform and report back via AMP receipts
4. Implement rate limiting appropriate to the external platform's constraints

### Lightweight Bridges

For same-provider deployments where the bridge and agents share a provider, bridges MAY use API key authentication only (without message signing). This simplifies deployment for internal bridges that don't cross federation boundaries. Bridges that forward messages across providers MUST use full cryptographic signing per [04-messages.md](04-messages.md).

## Registration

Bridges register like any AMP agent, with additional metadata:

```json
{
  "tenant": "23blocks",
  "name": "whatsapp-bridge",
  "public_key": "-----BEGIN PUBLIC KEY-----\n...",
  "key_algorithm": "Ed25519",
  "alias": "WhatsApp Bridge",
  "metadata": {
    "agent_type": "bridge",
    "channel_type": "whatsapp",
    "channel_config": {
      "account": "+1234567890",
      "capabilities": ["text", "media", "reactions"]
    }
  },
  "delivery": {
    "prefer_websocket": true
  }
}
```

## Channel Metadata

Messages originating from external channels MUST include a `channel` object in the payload context. This metadata enables reply routing, audit trails, and trust evaluation.

### Inbound Message Format

```json
{
  "envelope": {
    "version": "amp/0.1",
    "id": "msg_1706648400_abc123",
    "from": "whatsapp-bridge@23blocks.trycrabmail.com",
    "to": "support-agent@23blocks.trycrabmail.com",
    "subject": "WhatsApp message from John Doe",
    "priority": "normal",
    "timestamp": "2026-02-01T10:00:00Z",
    "signature": "...",
    "thread_id": "msg_1706648400_abc123"
  },
  "payload": {
    "type": "request",
    "message": "Hello, I need help with my order",
    "context": {
      "channel": {
        "type": "whatsapp",
        "sender": "+1987654321",
        "sender_name": "John Doe",
        "platform_message_id": "wamid.HBgNMTIzNDU2Nzg5MA",
        "thread_id": "wa_+1987654321",
        "bridge_agent": "whatsapp-bridge@23blocks.trycrabmail.com",
        "received_at": "2026-02-01T10:00:00Z"
      }
    },
    "security": {
      "trust": "untrusted",
      "source": "whatsapp",
      "scanned": true,
      "injection_flags": [],
      "wrapped": true,
      "scanned_at": "2026-02-01T10:00:01Z"
    }
    // NOTE: The security object follows the content security metadata
    // structure defined in 07-security.md. Bridges MUST use the same
    // schema — this is not a bridge-specific format.
  }
}
```

### Channel Types

| Type | Platform | Sender Format | Example |
|------|----------|---------------|---------|
| `email` | Email (SMTP) | Email address | `user@example.com` |
| `whatsapp` | WhatsApp | E.164 phone number | `+1234567890` |
| `slack` | Slack | Slack user ID | `U0123ABCDEF` |
| `sms` | SMS | E.164 phone number | `+1234567890` |
| `discord` | Discord | Discord user ID | `123456789012345678` |
| `telegram` | Telegram | Telegram user ID | `123456789` |
| `webchat` | Web widget | Session ID | `sess_abc123` |

Custom channel types SHOULD use a namespace prefix (e.g., `custom:intercom`, `custom:zendesk`).

### Required Channel Fields

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `type` | string | Yes | Channel type from table above |
| `sender` | string | Yes | External sender identifier |
| `sender_name` | string | No | Display name if available |
| `platform_message_id` | string | No | Original platform's message ID (for deduplication) |
| `thread_id` | string | No | Conversation thread identifier on external platform |
| `bridge_agent` | string | Yes | Full AMP address of the bridge agent |
| `received_at` | string | Yes | ISO 8601 timestamp of when bridge received the message |

### Optional Channel Fields

| Field | Type | Description |
|-------|------|-------------|
| `attachments` | array | List of attachment references (URLs or file metadata) |
| `reply_to_platform_id` | string | Platform message ID this is a reply to |
| `channel_name` | string | Channel/group name if applicable (e.g., Slack channel name) |
| `channel_id` | string | Channel/group ID on external platform |

## Reply Routing

When an AMP agent replies to a bridged message, the bridge MUST route the reply back to the original external channel and thread.

### Reply Detection

A bridge determines a message is a reply to an external channel message when:

1. `in_reply_to` references a message with channel metadata — route to that channel
2. `to` addresses a bridge agent with `channel_reply` context — route to specified channel

### Reply Format

AMP agents reply using standard AMP messages. They MAY include channel-specific instructions in the context:

```json
{
  "envelope": {
    "from": "support-agent@23blocks.trycrabmail.com",
    "to": "whatsapp-bridge@23blocks.trycrabmail.com",
    "subject": "Re: WhatsApp message from John Doe",
    "in_reply_to": "msg_1706648400_abc123",
    "thread_id": "msg_1706648400_abc123",
    "signature": "..."
  },
  "payload": {
    "type": "response",
    "message": "Hi John! I can see your order #12345. It's being shipped today.",
    "context": {
      "channel_reply": {
        "thread_id": "wa_+1987654321",
        "quote_original": true
      }
    }
  }
}
```

### Reply Routing Rules

| Condition | Bridge Action |
|-----------|--------------|
| `in_reply_to` references bridged message | Route to original sender on original platform |
| `channel_reply.thread_id` present | Route to specified thread on external platform |
| `channel_reply.quote_original` is true | Include quoted text of original message (platform-dependent) |
| No channel context, message to bridge | Bridge SHOULD reject or hold for manual routing |

### Delivery Confirmation

After posting the reply to the external platform, the bridge SHOULD send a delivery receipt back to the AMP agent:

```json
{
  "type": "notification",
  "message": "Reply delivered to WhatsApp +1987654321",
  "context": {
    "channel_delivery": {
      "platform_message_id": "wamid.HBgNMTIzNDU2Nzg5MB",
      "delivered_at": "2026-02-01T10:05:00Z",
      "status": "sent"
    }
  }
}
```

## Trust Model for Bridged Messages

All content from external channels is untrusted by default. Content security processing is mandatory for bridges.

| Source | Trust Level | Required Action |
|--------|-------------|-----------------|
| External channel user | `untrusted` | Content security scan + `<external-content>` wrapping |
| Verified operator (phone/email match) | `external` | Content security scan, operator-level trust, no wrapping |
| Same-tenant AMP agent (not bridged) | `verified` | No scanning required |

### Content Security Processing

Bridges MUST:

1. Scan all inbound external content for prompt injection patterns (see [Appendix A](appendix-a-injection-patterns.md))
2. Include `security` metadata on all forwarded messages using the content security metadata structure defined in [07-security.md](07-security.md)
3. Wrap untrusted content in `<external-content>` tags:

```
<external-content source="whatsapp" sender="+1987654321" trust="untrusted">
Hello, I need help with my order
</external-content>
```

4. Flag suspicious content with `injection_flags` array listing matched pattern categories

### Operator Verification

Bridges MAY verify that an external sender is a known operator by matching against a configured list:

- **Email bridges**: Match sender email address (require SPF/DKIM/DMARC pass)
- **WhatsApp/SMS bridges**: Match phone number against operator phone list
- **Slack bridges**: Match Slack user ID against operator user ID list

Verified operators receive `external` trust level (scanned but not wrapped).

## Bridge Health

Bridges SHOULD report channel-specific health information via the standard AMP health mechanism and/or their own management endpoint:

```json
{
  "status": "healthy",
  "channel": {
    "type": "whatsapp",
    "connected": true,
    "account": "+1234567890",
    "capabilities": ["text", "media"],
    "last_message_at": "2026-02-01T10:00:00Z",
    "messages_today": {
      "inbound": 42,
      "outbound": 38
    }
  }
}
```

When the external channel connection is degraded or lost, the bridge SHOULD update its presence status to reflect this:

```json
{
  "type": "presence.set",
  "data": {
    "status": "busy",
    "status_text": "WhatsApp disconnected, reconnecting..."
  }
}
```

## Bridge Discovery

Agents can discover available bridges in their tenant using the standard agent listing with type filter:

```http
GET /v1/agents?type=bridge&tenant=23blocks
Authorization: Bearer <api_key>

Response: 200 OK
{
  "agents": [
    {
      "address": "whatsapp-bridge@23blocks.trycrabmail.com",
      "alias": "WhatsApp Bridge",
      "online": true,
      "metadata": {
        "agent_type": "bridge",
        "channel_type": "whatsapp",
        "channel_config": {
          "account": "+1234567890",
          "capabilities": ["text", "media", "reactions"]
        }
      },
      "presence": {
        "status": "online",
        "status_text": null,
        "instances": 1,
        "last_active_at": "2026-02-01T10:30:00Z"
      }
    },
    {
      "address": "email-bridge@23blocks.trycrabmail.com",
      "alias": "Email Bridge",
      "online": true,
      "metadata": {
        "agent_type": "bridge",
        "channel_type": "email",
        "channel_config": {
          "domains": ["3metas.com"],
          "capabilities": ["text", "html", "attachments"]
        }
      },
      "presence": {
        "status": "online",
        "status_text": null,
        "instances": 1,
        "last_active_at": "2026-02-01T10:25:00Z"
      }
    }
  ],
  "total": 2
}
```

## Sending to External Channels

AMP agents can initiate outbound messages to external channels (not just reply) by sending to a bridge with explicit channel targeting:

```json
{
  "to": "whatsapp-bridge@23blocks.trycrabmail.com",
  "subject": "Outbound WhatsApp message",
  "payload": {
    "type": "task",
    "message": "Hello! Your appointment is confirmed for tomorrow at 2pm.",
    "context": {
      "channel_send": {
        "to": "+1987654321",
        "thread_id": "wa_+1987654321"
      }
    }
  }
}
```

The bridge translates this into a message on the external platform and sends a delivery confirmation back.

### Send Fields

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `channel_send.to` | string | Yes | Target on external platform (phone, email, user ID) |
| `channel_send.thread_id` | string | No | Continue existing thread |
| `channel_send.format` | string | No | Message format hint (`text`, `html`, `markdown`) |

## Implementation Notes

### Idempotency

Bridges SHOULD deduplicate inbound messages using `platform_message_id`. If the same external message is received twice (e.g., due to webhook retry), it SHOULD NOT be forwarded to AMP agents a second time.

### Rate Limiting

Bridges SHOULD respect the external platform's rate limits and queue outbound messages accordingly. When rate-limited, the bridge SHOULD hold the AMP message and retry rather than dropping it.

### Multi-Tenant Bridges

A single bridge instance MAY serve multiple tenants (e.g., one WhatsApp number handling messages for different agent teams). In this case, the bridge is responsible for routing inbound messages to the correct tenant based on configuration (e.g., keyword routing, phone number mapping).

---

Previous: [08 - API](08-api.md) | Next: [Appendix A - Injection Patterns](appendix-a-injection-patterns.md)
