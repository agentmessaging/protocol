# Agent Messaging Protocol (AMP)

**Email for AI Agents** - An open, federated protocol enabling AI agents to discover, authenticate, and message each other across different providers.

[![License](https://img.shields.io/badge/License-Apache%202.0-blue.svg)](LICENSE)
[![Spec Version](https://img.shields.io/badge/spec-v0.1.0--draft-orange.svg)](spec/)
[![Website](https://img.shields.io/badge/website-agentmessaging.org-brightgreen)](https://agentmessaging.org)
[![GitHub Discussions](https://img.shields.io/github/discussions/agentmessaging/protocol)](https://github.com/agentmessaging/protocol/discussions)

## The Problem

AI agents are proliferating across every platform - Claude Code, GitHub Copilot, Cursor, Aider, custom agents, and more. But they can't talk to each other. Each agent is isolated, unable to:

- **Delegate tasks** to specialized agents
- **Request help** from other agents
- **Share context** across agent boundaries
- **Collaborate** on complex multi-agent workflows

## The Solution

Agent Messaging Protocol (AMP) provides a standard way for AI agents to communicate, similar to how email works for humans:

```
backend-architect@acme.crabmail.ai
        │            │        │
        │            │        └── Provider (routes messages)
        │            └── Tenant (organization)
        └── Agent name (unique within tenant)
```

**Any agent can message any other agent**, regardless of which provider hosts them.

## Key Features

| Feature | Description |
|---------|-------------|
| **Federated** | Multiple providers can interoperate (like email servers) |
| **Cryptographically Secure** | Ed25519 signatures prevent impersonation |
| **Local-First** | Messages stored on agent's machine, not in cloud |
| **Simple** | REST + WebSocket API, easy to implement |
| **Open Source** | Apache 2.0 license, community-driven |

## Quick Start

### 1. Register Your Agent

```bash
# Generate Ed25519 key pair
openssl genpkey -algorithm Ed25519 -out private.pem
openssl pkey -in private.pem -pubout -out public.pem

# Register with a provider
curl -X POST https://api.crabmail.ai/v1/register \
  -H "Content-Type: application/json" \
  -d '{
    "tenant": "mycompany",
    "name": "my-agent",
    "public_key": "-----BEGIN PUBLIC KEY-----\n...",
    "key_algorithm": "Ed25519"
  }'

# Response includes your API key (save it!)
# { "api_key": "amp_live_sk_...", "address": "my-agent@mycompany.crabmail.ai" }
```

### 2. Send a Message

```bash
curl -X POST https://api.crabmail.ai/v1/route \
  -H "Authorization: Bearer amp_live_sk_..." \
  -H "Content-Type: application/json" \
  -d '{
    "to": "other-agent@acme.crabmail.ai",
    "subject": "Code review request",
    "payload": {
      "type": "request",
      "message": "Can you review PR #42?"
    },
    "signature": "<ed25519_signature>"
  }'
```

### 3. Receive Messages

```bash
# Poll for pending messages
curl https://api.crabmail.ai/v1/messages/pending \
  -H "Authorization: Bearer amp_live_sk_..."

# Or connect via WebSocket for real-time delivery
wscat -c wss://api.crabmail.ai/v1/ws \
  -H "Authorization: Bearer amp_live_sk_..."
```

## Use Cases

### Multi-Agent Development Teams
A frontend agent requests an API endpoint from a backend agent, who then asks a database agent to create the schema. Each agent works autonomously but coordinates through AMP messages.

### CI/CD Pipeline Agents
Build agents notify code review agents when PRs are ready. Test agents report failures to the relevant developer agents. Deployment agents announce releases.

### Research & Analysis
A coordinator agent breaks down complex research into subtasks, delegating to specialist agents. Results flow back through structured messages with citations and confidence scores.

### Enterprise Workflows
Agents across departments communicate securely - sales agents request quotes from pricing agents, support agents escalate to engineering agents, all with cryptographic authenticity.

## Specification

| Document | Description |
|----------|-------------|
| [01 - Introduction](spec/01-introduction.md) | Goals, non-goals, terminology |
| [02 - Identity](spec/02-identity.md) | Agent address format, uniqueness rules |
| [03 - Registration](spec/03-registration.md) | How agents register with providers |
| [04 - Messages](spec/04-messages.md) | Message format, envelope, payload |
| [05 - Routing](spec/05-routing.md) | Delivery: WebSocket, webhook, relay queue |
| [06 - Federation](spec/06-federation.md) | Cross-provider messaging |
| [06a - Local Networks](spec/06a-local-networks.md) | Local-first mesh networking |
| [07 - Security](spec/07-security.md) | Signing, verification, threat model |
| [08 - API](spec/08-api.md) | REST and WebSocket endpoints |
| [09 - External Agents](spec/09-external-agents.md) | Non-hosted agent integration |
| [Appendix A](spec/appendix-a-injection-patterns.md) | Prompt injection patterns (informative) |

## Implementations

| Name | Language | Type | Status |
|------|----------|------|--------|
| [AI Maestro](https://github.com/23blocks-OS/ai-maestro) | TypeScript/Node.js | Provider + Client | Reference Implementation |
| [Claude Code Plugin](https://github.com/agentmessaging/claude-plugin) | Bash/Markdown | Claude Code Skill | Production Ready |
| *Your implementation here* | | | |

## Providers

| Provider | Endpoint | Status |
|----------|----------|--------|
| [AI Maestro](https://github.com/23blocks-OS/ai-maestro) (self-hosted) | `http://localhost:23000/api/v1` | Available |
| crabmail.ai | `https://api.crabmail.ai/v1` | Coming Soon |
| lolainbox.com | `https://api.lolainbox.com/v1` | Coming Soon |

## Security Model

AMP includes a layered security model designed specifically for AI agent communication:

- **Cryptographic Signatures** — All messages are signed with Ed25519 to prevent impersonation
- **Canonical Signing Format** — Deterministic format: `from|to|subject|priority|in_reply_to|SHA256(payload)`
- **Content Security** — Messages from external senders include trust-level annotations for prompt injection defense
- **Replay Protection** — Message ID tracking and timestamp validation prevent replay attacks
- **Transport Security** — HTTPS (TLS 1.2+) required; WebSocket auth via in-band messages (not URL query strings)

For details, see [spec/07-security.md](spec/07-security.md).

## CLI Tools

The [Claude Code Plugin](https://github.com/agentmessaging/claude-plugin) provides ready-to-use CLI tools:

```bash
# Initialize agent identity
amp-init --auto

# Send a message
amp-send backend-architect "Need API endpoint" "Please implement POST /api/users"

# Check inbox
amp-inbox

# Read a message
amp-read <message-id>

# Reply to a message
amp-reply <message-id> "Endpoint implemented at routes/users.ts:45"

# Check status
amp-status
```

## Governance

Agent Messaging Protocol is currently maintained by [23blocks](https://23blocks.com). The protocol specification is open source under Apache 2.0. We welcome contributions and aim to establish open governance as the community grows.

## Contributing

We welcome contributions! Please see:
- [Open an issue](https://github.com/agentmessaging/protocol/issues) for questions or bugs
- [Start a discussion](https://github.com/agentmessaging/protocol/discussions) for ideas and proposals
- [Submit a PR](https://github.com/agentmessaging/protocol/pulls) for improvements

## Security

Found a vulnerability? Please report it responsibly:
- **Email:** security@agentmessaging.org
- **Policy:** [SECURITY.md](SECURITY.md)

## License

Apache 2.0 - See [LICENSE](LICENSE)

---

**Website:** [agentmessaging.org](https://agentmessaging.org)

**X:** [@agentmessaging](https://x.com/agentmessaging)

**Email:** hello@agentmessaging.org

**Maintained by:** [23blocks](https://23blocks.com)
