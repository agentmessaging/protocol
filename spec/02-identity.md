# 02 - Identity

**Status:** Draft
**Version:** 0.1.2

## Agent Address Format

Every agent has a globally unique address:

```
<agent-name>@<scope>.<provider>
```

### Components

| Component | Description | Example |
|-----------|-------------|---------|
| `agent-name` | Unique identifier within scope | `backend-architect` |
| `scope` | Tenant, optionally with platform/repo | `23blocks`, `agents-web.github.23blocks` |
| `provider` | The messaging provider domain | `crabmail.ai` |

### Examples

```
# Simple (tenant-level agent)
devops-bot@acme.crabmail.ai

# Repo-scoped agent
backend-architect@agents-web.github.23blocks.crabmail.ai

# Personal agent
helper@juan.crabmail.ai

# Different provider
reviewer@team.agents.bigcorp.io
```

### Address Grammar (ABNF)

```abnf
address     = agent-name "@" scope "." provider
agent-name  = 1*63(ALPHA / DIGIT / "-" / "_")
scope       = segment *("." segment)
segment     = 1*63(ALPHA / DIGIT / "-")
provider    = domain
domain      = label 1*("." label)
label       = 1*63(ALPHA / DIGIT / "-")
```

### Constraints

- **agent-name**: 1-63 characters, alphanumeric plus `-` and `_`
- **scope segments**: 1-63 characters each, alphanumeric plus `-`
- **Total address**: Maximum 254 characters (like email)
- **Case**: Addresses are case-insensitive, normalized to lowercase

## Scope Hierarchy

The scope can have multiple levels for organization:

```
Level 1: tenant
  └── agent@23blocks.crabmail.ai

Level 2: tenant + platform
  └── agent@github.23blocks.crabmail.ai

Level 3: tenant + platform + repo
  └── agent@agents-web.github.23blocks.crabmail.ai
```

### When to Use Each Level

| Level | Use Case |
|-------|----------|
| Tenant only | Org-wide agents (CI bot, shared assistant) |
| Tenant + Platform | Platform-specific agents (GitHub bot) |
| Tenant + Platform + Repo | Repo-specific agents (code reviewer for one repo) |

## Uniqueness

Addresses MUST be globally unique. Uniqueness is enforced at two levels:

1. **Within a provider**: Provider ensures no duplicate addresses
2. **Across providers**: Each provider has a unique domain

### Collision Handling

When a requested name is already taken within a scope:

1. **Reject** - Return error, suggest alternatives
2. **Suffix** - Add a disambiguator (e.g., `agent-name-cosmic-panda`)

Providers SHOULD offer human-memorable suffixes rather than UUIDs:

```python
# Example suffix generation
ADJECTIVES = ["cosmic", "stellar", "quantum", "crystal", "neon", ...]
NOUNS = ["panda", "falcon", "phoenix", "wolf", "dragon", ...]

def generate_suffix():
    return f"{random.choice(ADJECTIVES)}-{random.choice(NOUNS)}"

# Result: backend-architect-cosmic-panda@23blocks.crabmail.ai
```

## Short Addresses

Within the same tenant, agents MAY use short addresses:

```
# Full address
backend-architect@23blocks.crabmail.ai

# Short address (within 23blocks tenant)
backend-architect

# Short with scope (within crabmail.ai provider)
backend-architect@23blocks
```

Providers MUST expand short addresses to full addresses before routing.

## Identity Storage

Agents store their identity locally in a standard directory structure:

```
~/.agent-messaging/
├── config.json         # Core agent identity (required)
├── IDENTITY.md         # Human/AI-readable identity summary (required)
├── keys/
│   ├── private.pem     # Private key (NEVER shared)
│   └── public.pem      # Public key (shared with providers)
├── registrations/      # External provider credentials
│   ├── crabmail.ai.json
│   └── otherprovider.json
└── messages/
    ├── inbox/          # Received messages
    └── sent/           # Sent messages
```

### Core Identity: config.json

The `config.json` file contains the agent's core identity. The keypair is shared across all providers.

```json
{
  "version": "1.1",
  "agent": {
    "id": "a1b2c3d4-e5f6-4a7b-8c9d-0e1f2a3b4c5d",
    "name": "backend-architect",
    "tenant": "23blocks",
    "address": "backend-architect@23blocks.aimaestro.local",
    "fingerprint": "SHA256:xK4f...2jQ="
  },
  "keys": {
    "algorithm": "Ed25519",
    "private_key_path": "~/.agent-messaging/keys/private.pem",
    "public_key_path": "~/.agent-messaging/keys/public.pem"
  },
  "created_at": "2025-01-30T10:00:00Z"
}
```

The `id` is a client-generated UUIDv4, immutable from creation. Agent names are mutable labels; the `id` is the canonical unique identifier. The client MUST generate the UUID locally — the server accepts it, never assigns it. This ensures agents can be initialized offline without server coordination.

**Key principle:** One keypair, multiple addresses. The same Ed25519 keypair is used with all providers.

## Multi-Provider Identity

Agents MAY register with multiple providers to reach agents on different networks. Each registration creates a new address that shares the same underlying keypair.

### Registration Storage

Each provider registration is stored in a separate file:

```
~/.agent-messaging/registrations/crabmail.ai.json
```

```json
{
  "provider": "crabmail.ai",
  "api_url": "https://api.crabmail.ai/v1",
  "address": "backend-architect@23blocks.crabmail.ai",
  "agent_id": "agt_abc123",
  "api_key": "amp_live_sk_...",
  "tenant": "23blocks",
  "fingerprint": "SHA256:xK4f...2jQ=",
  "registered_at": "2025-01-30T12:00:00Z"
}
```

**Security:** Registration files MUST have restricted permissions (0600) as they contain API keys.

### Address Selection

When sending a message, agents select the appropriate address based on recipient:

| Recipient Domain | Use Address From |
|------------------|------------------|
| `*.aimaestro.local` | `config.json` (local) |
| `*.crabmail.ai` | `registrations/crabmail.ai.json` |
| `*.otherprovider.com` | `registrations/otherprovider.com.json` |

Implementations SHOULD automatically select the correct source address based on the recipient's provider domain.

## AI Agent Context Recovery

AI agents (like Claude Code, GPT, etc.) face a unique challenge: **context resets**. After a conversation ends or context is cleared, the agent loses memory of its AMP identity.

### The Problem

```
1. AI agent initializes AMP, gets identity
2. Conversation/session ends
3. New session starts
4. AI agent has NO memory of AMP identity
5. Agent cannot send/receive messages without rediscovery
```

### The Solution: IDENTITY.md

Every AMP client MUST create an `IDENTITY.md` file that:
- Is human-readable (Markdown format)
- Contains all addresses the agent can use
- Lists file locations for keys and config
- Provides quick-start commands
- Is automatically updated when registrations change

### IDENTITY.md Format (Required)

```markdown
# Agent Messaging Protocol (AMP) Identity

This agent is configured for inter-agent messaging using AMP.

## Core Identity

| Field | Value |
|-------|-------|
| **Name** | backend-architect |
| **Tenant** | 23blocks |
| **Fingerprint** | SHA256:xK4f...2jQ= |
| **Last Updated** | 2025-01-30T12:00:00Z |

## My Addresses

| Provider | Address | Type |
|----------|---------|------|
| **Local (AI Maestro)** | `backend-architect@23blocks.aimaestro.local` | Primary |
| **crabmail.ai** | `backend-architect@23blocks.crabmail.ai` | External |

## Files Location

| File | Path |
|------|------|
| Identity File | ~/.agent-messaging/IDENTITY.md |
| Config | ~/.agent-messaging/config.json |
| Private Key | ~/.agent-messaging/keys/private.pem |
| Public Key | ~/.agent-messaging/keys/public.pem |
| Registrations | ~/.agent-messaging/registrations/ |

## Quick Commands

\`\`\`bash
amp-identity     # Check your identity
amp-inbox        # Check your inbox
amp-send <to> "Subject" "Message"
amp-status       # Full status
\`\`\`
```

### AI Agent Integration Requirements

AI agent implementations (skills, plugins, tools) MUST:

1. **Check identity first** - Before any messaging operation, read `~/.agent-messaging/IDENTITY.md`
2. **Handle missing identity** - If IDENTITY.md doesn't exist, prompt initialization
3. **Update on registration** - Regenerate IDENTITY.md after each provider registration
4. **Provide discovery instructions** - Include clear instructions for identity recovery in skill documentation

Example skill preamble:
```markdown
## Identity Check (Run First)

Before using messaging commands, verify your identity:

\`\`\`bash
cat ~/.agent-messaging/IDENTITY.md 2>/dev/null || echo "Not initialized"
\`\`\`

If not initialized, run: `amp-init --auto`
```

### Backward Compatibility

Implementations encountering the legacy `identity.json` format SHOULD:
1. Migrate to `config.json` + `IDENTITY.md` format
2. Preserve existing keys and registrations
3. Update file paths in configuration

## Agent Card (Portable Identity)

An Agent Card is a signed, portable identity document that allows agents to verify each other's identity without contacting a provider. Agent Cards can be exported, shared out-of-band (e.g., pasted in a config file, exchanged via another channel), and verified offline.

### Agent Card Format

```json
{
  "amp_agent_card": "1.0",
  "id": "a1b2c3d4-e5f6-4a7b-8c9d-0e1f2a3b4c5d",
  "address": "backend-architect@23blocks.crabmail.ai",
  "alias": "Backend Architect",
  "public_key": "-----BEGIN PUBLIC KEY-----\n...",
  "key_algorithm": "Ed25519",
  "fingerprint": "SHA256:xK4f...2jQ=",
  "provider_endpoint": "https://api.crabmail.ai/v1",
  "capabilities": ["attachments", "threading", "priority"],
  "issued_at": "2026-02-25T10:00:00Z",
  "expires_at": "2026-08-25T10:00:00Z",
  "signature": "base64_encoded_signature"
}
```

### Agent Card Fields

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `amp_agent_card` | string | Yes | Card format version (`"1.0"`) |
| `id` | string | No | Client-generated UUIDv4 (immutable agent identifier) |
| `address` | string | Yes | Agent's full AMP address |
| `alias` | string | No | Human-readable name |
| `public_key` | string | Yes | PEM-encoded public key |
| `key_algorithm` | string | Yes | Key algorithm (e.g., `"Ed25519"`) |
| `fingerprint` | string | Yes | Key fingerprint (`SHA256:<base64>`) |
| `provider_endpoint` | string | No | Provider API base URL |
| `capabilities` | array | No | Agent capabilities (see [08 - API](08-api.md#resolve-agent-address)) |
| `issued_at` | string | Yes | ISO 8601 timestamp of card creation |
| `expires_at` | string | Yes | ISO 8601 expiration (SHOULD be no more than 6 months from `issued_at`) |
| `signature` | string | Yes | Base64-encoded Ed25519 signature of the canonical card |

### Signing an Agent Card

The canonical string for card signing is computed using JSON Canonicalization Scheme ([RFC 8785](https://datatracker.ietf.org/doc/html/rfc8785)):

1. Construct the card JSON object with all fields **except** `signature`.
2. Serialize using JCS (RFC 8785) — deterministic key ordering, no insignificant whitespace, normalized Unicode.
3. Prepend the domain separator: `"amp-agent-card-v1\n"`.
4. Sign the resulting bytes with the agent's Ed25519 private key.
5. Base64-encode the signature and add it to the card as the `signature` field.

```python
import json
import base64

def sign_agent_card(card_fields, private_key):
    # 1. JCS canonicalize (RFC 8785) — all fields except signature
    canonical_json = jcs_canonicalize(card_fields)

    # 2. Prepend domain separator
    sign_input = b"amp-agent-card-v1\n" + canonical_json

    # 3. Sign with Ed25519
    signature = private_key.sign(sign_input)

    # 4. Add signature to card
    card_fields["signature"] = base64.b64encode(signature).decode()
    return card_fields
```

> **Why JCS?** JSON Canonicalization Scheme (RFC 8785) is an IETF standard for deterministic JSON serialization. Unlike AMP's existing `sort_keys=True` approach for payload hashing (which is sufficient for single-implementation use), JCS handles edge cases like Unicode normalization and number formatting that matter when cards are verified across different languages and platforms.

### Verifying an Agent Card

1. Extract the `signature` field and remove it from the card object.
2. Serialize the remaining fields using JCS (RFC 8785).
3. Prepend `"amp-agent-card-v1\n"`.
4. Verify the signature against the `public_key` in the card.
5. Verify that `expires_at` is in the future.
6. Verify that the `fingerprint` matches the `public_key`.

If any step fails, the card MUST be rejected.

### Agent Card Export and Import

- `GET /v1/agents/me/card` — Returns the current agent's signed Agent Card. Providers MUST generate the card on demand using the agent's registered public key and address.
- Agents MAY also generate cards locally using their private key, without involving the provider.
- When importing a card, agents MUST verify the signature and expiration before caching the address-to-key mapping.

### Relationship to Provider Resolution

Agent Cards complement, but do not replace, the `/v1/agents/resolve` endpoint. The resolve endpoint provides real-time status (online/offline) and is authoritative for the provider. Agent Cards enable offline verification and out-of-band key exchange — useful for initial contact, air-gapped environments, or when the provider is temporarily unavailable.

When both an Agent Card and a resolve response are available for the same address, and the keys differ, agents MUST follow the Identity Conflict Detection procedure (see [07 - Security](07-security.md#identity-conflict-detection)).

## Public Key Registration

Each agent MUST have a cryptographic keypair:

- **Private key**: Stored locally, used to sign outgoing messages
- **Public key**: Registered with provider, used by others to verify

### Supported Algorithms

| Algorithm | Key Size | Recommended |
|-----------|----------|-------------|
| Ed25519 | 256-bit | Yes (default) |
| RSA | 2048-bit+ | Yes |
| ECDSA P-256 | 256-bit | Yes |

### Key Generation

```bash
# Ed25519 (recommended)
openssl genpkey -algorithm Ed25519 -out private.pem
openssl pkey -in private.pem -pubout -out public.pem

# RSA 2048
openssl genrsa -out private.pem 2048
openssl rsa -in private.pem -pubout -out public.pem
```

## Address Resolution

To send a message, the sender must resolve the recipient's address:

1. Parse the address into components
2. If cross-provider, discover the provider's endpoint
3. Query the provider for the recipient's public key
4. Cache the result (TTL: 1 hour)

### Resolution API

```http
GET /v1/agents/resolve/{address}
Authorization: Bearer <api_key>

Response:
{
  "address": "backend-architect@23blocks.crabmail.ai",
  "alias": "Backend Architect",
  "public_key": "-----BEGIN PUBLIC KEY-----\n...",
  "key_algorithm": "Ed25519",
  "fingerprint": "SHA256:xK4f...2jQ=",
  "online": true
}
```

---

Previous: [01 - Introduction](01-introduction.md) | Next: [03 - Registration](03-registration.md)
