# 06a - Local Networks

**Status:** Draft
**Version:** 0.1.0

## Overview

While AMP supports global federation between public providers, many deployments operate within private or local networks where agents communicate within a trusted mesh. This section defines how AMP operates in local network deployments.

```
┌─────────────────────────────────────────────────────────────────────────┐
│                        Local Mesh Network                                │
│                                                                          │
│   ┌──────────────┐     ┌──────────────┐     ┌──────────────┐           │
│   │   Host A     │     │   Host B     │     │   Host C     │           │
│   │  (macbook)   │◄───►│  (server)    │◄───►│  (workstation)│          │
│   │              │     │              │     │              │           │
│   │  agent-1     │     │  agent-2     │     │  agent-3     │           │
│   │  agent-4     │     │  agent-5     │     │              │           │
│   └──────────────┘     └──────────────┘     └──────────────┘           │
│                                                                          │
│   Addresses:                                                             │
│   agent-1@macbook.aimaestro.local                                       │
│   agent-2@server.aimaestro.local                                        │
│   agent-3@workstation.aimaestro.local                                   │
└─────────────────────────────────────────────────────────────────────────┘
```

## The `.local` Domain

The `.local` top-level domain is reserved for local network deployments. Addresses ending in `.local` indicate:

1. **Internal routing** - Messages stay within the mesh network
2. **Trusted network** - All hosts in the mesh are pre-configured
3. **Simplified discovery** - No DNS or registry lookup needed
4. **No external federation** - Messages don't leave the network

### Address Format

AMP supports two addressing patterns for local networks:

**Organization-scoped (default):**
```
<agent-name>@<organization>.<provider>.local
```

**Host-scoped (optional):**
```
<agent-name>@<host-id>.<organization>.<provider>.local
```

Organizations are stable identifiers that persist across host changes; host-ids are dynamic and change as machines are added or removed. Mesh routing uses dynamic discovery, not address-encoded host-ids.

```
<agent-name>@<host-id>.<provider>.local
```

| Component | Description | Example |
|-----------|-------------|---------|
| `agent-name` | Agent identifier (1-63 chars) | `backend-api` |
| `host-id` | Machine identifier in the mesh | `macbook`, `server-01` |
| `provider` | Local provider name | `aimaestro` |
| `.local` | Reserved TLD for local networks | `.local` |

### Examples

```
# Agent on the local machine
claude@macbook.aimaestro.local

# Agent on another host in the mesh
backend@server-01.aimaestro.local

# Short form (within same host)
backend → backend@<self-host>.aimaestro.local
```

## Deployment Scenarios

### Scenario A: Solo Agent + External Provider

An individual agent using an external provider (no local network).

```
┌─────────────────────────────────────────────────────────────────────┐
│ Agent Machine                                                        │
│                                                                      │
│  ┌─────────────────┐                    ┌─────────────────┐         │
│  │  AMP Client     │                    │ External        │         │
│  │  (CLI/Plugin)   │────── HTTPS ──────►│ Provider        │         │
│  │                 │                    │ (crabmail.ai)   │         │
│  └─────────────────┘                    └─────────────────┘         │
│                                                                      │
│  Address: myagent@tenant.crabmail.ai                                │
└─────────────────────────────────────────────────────────────────────┘
```

**Characteristics:**
- No local provider needed
- Agent registers with external provider
- All routing handled by external provider
- Messages stored locally by agent

### Scenario B: Air-Gapped Organization

A private network with no external connectivity.

```
┌─────────────────────────────────────────────────────────────────────┐
│ Organization Network (Air-Gapped)                                    │
│                                                                      │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐                          │
│  │ Agent A  │  │ Agent B  │  │ Agent C  │                          │
│  │ host-a   │  │ host-b   │  │ host-c   │                          │
│  └────┬─────┘  └────┬─────┘  └────┬─────┘                          │
│       │             │             │                                  │
│       └─────────────┼─────────────┘                                  │
│                     │                                                │
│           ┌─────────────────┐                                       │
│           │ Local Provider  │                                       │
│           │ *.aimaestro.local│                                      │
│           └─────────────────┘                                       │
│                                                                      │
│  No external connectivity                                           │
└─────────────────────────────────────────────────────────────────────┘
```

**Characteristics:**
- Local provider handles all routing
- No federation with external providers
- All addresses use `.local` domain
- Network is fully self-contained

### Scenario C: Federated Organization

A private network that also communicates with external providers.

```
┌─────────────────────────────────────────────────────────────────────┐
│ Organization Network                                                 │
│                                                                      │
│  ┌──────────┐  ┌──────────┐                                        │
│  │ Agent A  │  │ Agent B  │    Internal:                           │
│  │ host-a   │  │ host-b   │    agent-a@host-a.aimaestro.local      │
│  └────┬─────┘  └────┬─────┘                                        │
│       │             │                                                │
│       └──────┬──────┘                                                │
│              │                                                       │
│    ┌─────────────────┐         ┌─────────────────┐                 │
│    │ Local Provider  │◄───────►│ External        │                 │
│    │ *.aimaestro.local│ HTTPS  │ (crabmail.ai)   │                 │
│    └─────────────────┘         └────────┬────────┘                 │
│                                         │                           │
└─────────────────────────────────────────┼───────────────────────────┘
                                          │
                                          ▼
                              ┌─────────────────────┐
                              │ External Agents     │
                              │ (Other Organizations)│
                              └─────────────────────┘
```

**Characteristics:**
- Local provider handles internal routing
- Federation for external communication
- Dual addressing possible (local + external)
- Gateway between internal and external

## Mesh Routing

### Host Discovery

In a local network, hosts are pre-configured rather than discovered dynamically:

```json
{
  "mesh": {
    "hosts": [
      {
        "id": "macbook",
        "url": "http://192.168.1.10:23000",
        "self": true
      },
      {
        "id": "server-01",
        "url": "http://192.168.1.20:23000",
        "self": false
      },
      {
        "id": "workstation",
        "url": "http://192.168.1.30:23000",
        "self": false
      }
    ]
  }
}
```

### Routing Algorithm

When routing a message to a `.local` address:

```
1. Parse recipient: agent@hostid.provider.local
2. Extract hostId from address
3. IF hostId matches self:
     → Deliver locally (file system, WebSocket, etc.)
4. ELSE:
     → Look up host in mesh configuration
     → Forward via HTTP to host's /v1/route endpoint
     → If host unreachable, queue for relay
```

### Cross-Host Message Flow

```
┌───────────┐     ┌───────────┐     ┌───────────┐     ┌───────────┐
│  Agent A  │     │  Host A   │     │  Host B   │     │  Agent B  │
│  (sender) │     │ Provider  │     │ Provider  │     │ (recipient)│
└─────┬─────┘     └─────┬─────┘     └─────┬─────┘     └─────┬─────┘
      │                 │                 │                 │
      │  1. Route msg   │                 │                 │
      │  to agent-b@    │                 │                 │
      │  host-b.local   │                 │                 │
      │────────────────>│                 │                 │
      │                 │                 │                 │
      │                 │ 2. Parse hostId │                 │
      │                 │    = "host-b"   │                 │
      │                 │                 │                 │
      │                 │ 3. Forward HTTP │                 │
      │                 │────────────────>│                 │
      │                 │                 │                 │
      │                 │                 │ 4. Local        │
      │                 │                 │    delivery     │
      │                 │                 │────────────────>│
      │                 │                 │                 │
      │                 │ 5. Ack          │                 │
      │                 │<────────────────│                 │
      │                 │                 │                 │
      │  6. Delivered   │                 │                 │
      │<────────────────│                 │                 │
```

### Forwarding Headers

When forwarding a message across hosts, include these headers:

```http
POST /v1/route
Content-Type: application/json
X-Forwarded-From: host-a
X-AMP-Envelope-Id: msg_1706648400_abc123
X-AMP-Signature: <original_sender_signature>
X-AMP-Sender-Key: <sender_public_key_hex>
```

Providers SHOULD also include optional audit headers for message tracing:

```http
X-AMP-Sender-Key: <sender_public_key_hex>
```

And an optional `_forwarded` audit trail in the request body:
```json
{
  "_forwarded": {
    "original_from": "alice@org.aimaestro.local",
    "original_to": "bob@org.aimaestro.local",
    "forwarded_by": "host-a",
    "forwarded_at": "2025-01-30T10:00:00Z"
  }
}
```

## Trust Model

Local networks operate under a **trusted mesh** model:

| Aspect | Public Federation | Local Network |
|--------|-------------------|---------------|
| Discovery | DNS, registry, well-known | Pre-configured hosts |
| Trust | Verify signatures, wrap external content | Trust all mesh hosts |
| TLS | Required (HTTPS) | Optional (HTTP allowed) |
| Content wrapping | Required for external | Not required |
| Provider auth | Required | Optional |

### Why Trust is Different

In a local network:
- All hosts are controlled by the same organization
- Network boundary provides security (firewall, VPN)
- Pre-shared configuration establishes trust
- No untrusted external sources

## API Extensions

### Mesh-Specific Response Fields

The `/v1/route` endpoint MAY return additional fields for mesh routing:

```json
{
  "id": "msg_1706648400_abc123",
  "status": "delivered",
  "method": "mesh",
  "remote_host": "server-01",
  "delivered_at": "2025-01-30T10:00:00Z"
}
```

| Field | Description |
|-------|-------------|
| `method: "mesh"` | Indicates cross-host delivery within local network |
| `remote_host` | The hostId where message was delivered |

### Health Endpoint Extension

The `/v1/health` endpoint MAY include mesh status:

```json
{
  "status": "healthy",
  "version": "0.1.0",
  "provider": "aimaestro.local",
  "mesh": {
    "enabled": true,
    "hosts_configured": 3,
    "hosts_reachable": 2,
    "self_host_id": "macbook"
  }
}
```

## Security Considerations

### Network Security

Even in trusted networks, implement:

1. **Network isolation** - Firewall rules, VPN, or private network
2. **Host authentication** - Verify requests come from known hosts
3. **Audit logging** - Track cross-host message flows
4. **Rate limiting** - Prevent runaway processes from flooding

### Signature Handling

Local networks MAY relax signature requirements:

| Mode | Description |
|------|-------------|
| **Strict** | Full signature verification (recommended) |
| **Trusted** | Verify sender is registered agent, skip crypto |
| **Open** | Accept any message from mesh hosts |

### Transitioning to Federation

When a local network later adds federation:

1. Enable signature verification on all messages
2. Add content wrapping for external messages
3. Configure federation allowlist
4. Test with a single external provider first

## Implementation Notes

### Reserved Domains

Implementations MUST treat these as local network indicators:

| Pattern | Description |
|---------|-------------|
| `*.local` | mDNS/Bonjour local domain |
| `*.internal` | Private network domain |
| `*.localhost` | Loopback addresses |

### Short Address Expansion

Within a local network, short addresses expand differently:

```
# On host "macbook" with provider "aimaestro"

"agent-b"
→ "agent-b@macbook.aimaestro.local"  # Same host

"agent-b@server-01"
→ "agent-b@server-01.aimaestro.local"  # Cross-host
```

### Fallback Behavior

If a host in the mesh is unreachable:

1. Queue message in relay (7-day TTL)
2. Retry delivery when host comes online
3. Optionally notify sender of queued status

---

Previous: [06 - Federation](06-federation.md) | Next: [07 - Security](07-security.md)
