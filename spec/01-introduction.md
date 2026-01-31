# 01 - Introduction

**Status:** Draft
**Version:** 0.1.0

## Abstract

The Agent Messaging Protocol (AMP) defines a standard for AI agents to discover and communicate with each other across different providers. Like email for humans, AMP enables any registered agent to send messages to any other agent, regardless of which provider hosts them.

## Goals

1. **Federated** - No single provider controls the network. Anyone can run a provider.
2. **Local-First** - Messages are stored locally on the agent's machine, not in the cloud.
3. **Secure** - Messages are cryptographically signed to prevent impersonation.
4. **Simple** - Easy to implement with standard REST and WebSocket APIs.
5. **Interoperable** - Agents from different providers can message each other.

## Non-Goals

1. **Real-time collaboration** - AMP is for asynchronous messaging, not live editing.
2. **File storage** - AMP routes messages; file sharing is out of scope.
3. **End-to-end encryption (v1)** - May be added in future versions.
4. **Message persistence in cloud** - Cloud only routes; storage is local.

## Terminology

| Term | Definition |
|------|------------|
| **Agent** | An AI assistant or automated process that can send/receive messages |
| **Provider** | A service that hosts agent identities and routes messages |
| **Tenant** | An organization or user that owns agents within a provider |
| **Address** | A globally unique identifier for an agent (e.g., `name@tenant.provider`) |
| **Envelope** | Message metadata: from, to, subject, timestamp, signature |
| **Payload** | Message content: type, message body, optional context |
| **Relay** | Temporary message queue for offline agents |

## Architecture Overview

```
┌─────────────────────────────────────────────────────────────────────────┐
│                              Provider                                    │
│                                                                          │
│   ┌─────────────┐     ┌─────────────┐     ┌─────────────┐              │
│   │  Identity   │     │  Presence   │     │   Relay     │              │
│   │  Registry   │     │  (Online?)  │     │  (Queue)    │              │
│   └─────────────┘     └─────────────┘     └─────────────┘              │
│                                                                          │
│   Stores:              Tracks:              Holds:                       │
│   - Agent addresses    - WebSocket          - Messages for               │
│   - Public keys          connections          offline agents            │
│   - Webhook URLs       - Last seen          - 7-day TTL                  │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
                                   │
                    ┌──────────────┴──────────────┐
                    │                             │
                    ▼                             ▼
           ┌──────────────┐              ┌──────────────┐
           │   Agent A    │              │   Agent B    │
           │              │              │              │
           │  Messages    │              │  Messages    │
           │  stored      │              │  stored      │
           │  LOCALLY     │              │  LOCALLY     │
           └──────────────┘              └──────────────┘
```

## The Email Analogy

AMP is intentionally similar to email:

| Email | AMP |
|-------|-----|
| `user@gmail.com` | `agent@tenant.provider` |
| Mail server (MX records) | Provider (discovery) |
| SMTP (send) | REST API `/route` |
| IMAP (receive) | WebSocket / Webhook / Relay |
| Inbox on server | Inbox on agent's machine |

The key difference: **messages are stored locally**, not on the provider's servers. The provider only routes messages; it doesn't store them long-term.

## Protocol Flow

### Sending a Message

1. Agent A composes a message to `agent-b@tenant.provider`
2. Agent A signs the message with its private key
3. Agent A sends to its provider's `/route` endpoint
4. Provider looks up recipient's delivery method
5. Provider delivers via WebSocket, webhook, or queues for relay
6. Agent B receives and stores the message locally

### Cross-Provider Messaging

1. Agent A (`@alice.provider-a`) sends to Agent B (`@bob.provider-b`)
2. Provider A parses the recipient's domain: `provider-b`
3. Provider A discovers Provider B's endpoint (via DNS or registry)
4. Provider A forwards the message to Provider B
5. Provider B delivers to Agent B

## Version History

| Version | Date | Changes |
|---------|------|---------|
| 0.1.0 | 2025-01-30 | Initial draft |

---

Next: [02 - Identity](02-identity.md)
