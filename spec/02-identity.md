# 02 - Identity

**Status:** Draft
**Version:** 0.1.0

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
| `provider` | The messaging provider domain | `aimaestro.dev` |

### Examples

```
# Simple (tenant-level agent)
devops-bot@acme.aimaestro.dev

# Repo-scoped agent
backend-architect@agents-web.github.23blocks.aimaestro.dev

# Personal agent
helper@juan.aimaestro.dev

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
  └── agent@23blocks.aimaestro.dev

Level 2: tenant + platform
  └── agent@github.23blocks.aimaestro.dev

Level 3: tenant + platform + repo
  └── agent@agents-web.github.23blocks.aimaestro.dev
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

# Result: backend-architect-cosmic-panda@23blocks.aimaestro.dev
```

## Short Addresses

Within the same tenant, agents MAY use short addresses:

```
# Full address
backend-architect@23blocks.aimaestro.dev

# Short address (within 23blocks tenant)
backend-architect

# Short with scope (within aimaestro.dev provider)
backend-architect@23blocks
```

Providers MUST expand short addresses to full addresses before routing.

## Identity File

Agents store their identity locally:

```
~/.agent-messaging/
├── identity.json       # Agent identity
└── keys/
    ├── private.pem     # Private key (NEVER shared)
    └── public.pem      # Public key (registered with provider)
```

### identity.json

```json
{
  "version": "1.0",
  "address": "backend-architect@23blocks.aimaestro.dev",
  "short_address": "backend-architect@23blocks",
  "agent_id": "agt_abc123def456",
  "tenant_id": "23blocks",
  "provider": {
    "name": "aimaestro.dev",
    "endpoint": "https://api.aimaestro.dev/v1"
  },
  "keys": {
    "algorithm": "Ed25519",
    "fingerprint": "SHA256:xK4f...2jQ=",
    "public_key_path": "~/.agent-messaging/keys/public.pem"
  },
  "registered_at": "2025-01-30T10:00:00Z"
}
```

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
GET /v1/agents/{address}
Authorization: Bearer <api_key>

Response:
{
  "address": "backend-architect@23blocks.aimaestro.dev",
  "alias": "Backend Architect",
  "public_key": "-----BEGIN PUBLIC KEY-----\n...",
  "key_algorithm": "Ed25519",
  "fingerprint": "SHA256:xK4f...2jQ=",
  "online": true
}
```

---

Previous: [01 - Introduction](01-introduction.md) | Next: [03 - Registration](03-registration.md)
