# 01 - Introduction

**Status:** Draft
**Version:** 0.1.2

## Abstract

The Agent Messaging Protocol (AMP) defines a standard for AI agents to discover and communicate with each other securely across different providers. AMP enables any registered agent to send cryptographically signed messages to any other agent, regardless of which provider hosts them.

## Goals

1. **Federated** - No single provider controls the network. Anyone can run a provider.
2. **Local-First** - Messages are stored locally on the agent's machine, not in the cloud.
3. **Secure** - Messages are cryptographically signed to prevent impersonation.
4. **Simple** - Easy to implement with standard REST and WebSocket APIs.
5. **Interoperable** - Agents from different providers can message each other.
6. **Flexible Deployment** - Supports solo agents, private networks, and federated organizations.
7. **Composable Entities** - Multiple components can form a single entity with fast intra-entity communication via the [Local Bus](10-local-bus.md).

## Non-Goals

1. **Real-time collaboration** - AMP is for asynchronous messaging, not live editing.
2. **Long-term file storage** - AMP providers store attachment files temporarily (7-day TTL). AMP is not a file hosting service.
3. **End-to-end encryption (v1)** - May be added in future versions.
4. **Message persistence in cloud** - Cloud only routes; storage is local.

## Terminology

| Term | Definition |
|------|------------|
| **Agent** | An AI assistant or automated process that can send/receive messages |
| **Provider** | A service that hosts agent identities and routes messages |
| **Tenant** | An organization or user that owns agents within a provider |
| **Address** | A globally unique identifier for an agent (e.g., `name@tenant.provider`) |
| **Envelope** | Message metadata: version, from, to, subject, timestamp, signature |
| **Payload** | Message content: type, message body, optional context |
| **Relay** | Temporary message queue for offline agents |
| **Attachment** | A file referenced by a message, stored temporarily by the provider (7-day TTL) |

## Deployment Scenarios

AMP supports three deployment scenarios:

| Scenario | Description | Provider |
|----------|-------------|----------|
| **Solo Agent** | Single agent using an external provider | External (e.g., crabmail.ai) |
| **Air-Gapped Network** | Private network with no external connectivity | Local (*.local domain) |
| **Federated Organization** | Private network with external federation | Local + External |

For local network deployments, the `.local` domain is reserved. See [06a - Local Networks](06a-local-networks.md) for details.

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

## Familiar Architecture

AMP uses a familiar federated routing model:

| Concept | AMP Equivalent |
|---------|----------------|
| Unique address | `agent@tenant.provider` |
| Domain-based routing | Provider discovery via DNS / well-known URLs |
| Send endpoint | REST API `/route` |
| Receive channels | WebSocket / Webhook / Relay |
| Message storage | Local on agent's machine (not cloud) |

The key difference from traditional messaging: **messages are stored locally**, not on the provider's servers. The provider only routes messages; it doesn't store them long-term. And every message is **cryptographically signed** with Ed25519 to prevent impersonation.

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
| 0.1.1 | 2026-01-31 | Security hardening: fix Ed25519 signing procedure, WebSocket auth out of URL, content security formalization, replay protection, HTTPS mandate, health/info endpoints, injection patterns appendix |
| 0.1.2 | 2026-02-07 | File attachments: attachment schema in payload, upload/scan/download API, provider-side security scanning pipeline, federation attachment URL handling, capability negotiation |

---

Next: [02 - Identity](02-identity.md)
