# 10 - Local Bus

**Status:** Draft
**Version:** 0.1.2

## Overview

AMP defines **inter-entity** messaging — agents communicating across networks via providers. But when multiple components operate as a single entity (e.g., a "brain" composed of `cortex`, `cerebellum`, and `gateway` processes), they need fast, low-latency **intra-entity** communication that HTTP REST cannot provide.

The Local Bus is a lightweight JSON-RPC 2.0 transport over Unix Domain Sockets (UDS) that enables components within a single entity to communicate with sub-millisecond latency. A dedicated gateway component bridges the internal bus to the external AMP network, presenting a single AMP identity for the entire entity.

### What Local Bus Is

- A same-machine IPC mechanism for components of a single AMP entity
- A star-topology bus using JSON-RPC 2.0 over Unix Domain Sockets
- A complement to AMP, not a replacement

### What Local Bus Is NOT

- NOT a replacement for AMP inter-entity messaging
- NOT a network protocol — it operates exclusively on a single machine
- NOT a general-purpose message broker (no persistence, no durability guarantees)

## Architecture

The Local Bus uses a **star topology** with a central bus process. All components connect to the bus via a shared Unix Domain Socket. The bus handles routing, discovery, and lifecycle management.

```
                         ┌─────────────────────────┐
                         │       AMP Network        │
                         │  (Providers, Agents)     │
                         └────────────┬────────────┘
                                      │
                                      │ AMP HTTP/WS
                                      │
┌─────────────────────────────────────────────────────────────────┐
│  Entity: "brain"                                                │
│                                                                 │
│     ┌───────────┐       ┌───────────┐       ┌───────────┐      │
│     │  cortex   │       │ cerebellum│       │  gateway  │      │
│     │           │       │           │       │ (AMP ↔ Bus)│      │
│     └─────┬─────┘       └─────┬─────┘       └─────┬─────┘      │
│           │                   │                     │            │
│           │    UDS            │    UDS               │    UDS    │
│           │                   │                     │            │
│           └───────────┬───────┴─────────────────────┘            │
│                       │                                          │
│                ┌──────┴──────┐                                   │
│                │  Bus Process │                                   │
│                │  (Router)    │                                   │
│                └─────────────┘                                   │
│                                                                 │
│  Socket: /tmp/amp-bus-brain/bus.sock                             │
└─────────────────────────────────────────────────────────────────┘
```

### Why Star Topology

| Property | Star (chosen) | Mesh |
|----------|---------------|------|
| Routing complexity | O(1) — bus routes all | O(n) — every component routes |
| Discovery | Centralized, deterministic | Distributed, eventually consistent |
| New component joins | Connects to bus only | Must discover all peers |
| Single point of failure | Bus process | None (but higher complexity) |

The star topology trades redundancy for simplicity. Since all components run on the same machine, the bus process can be supervised and restarted quickly by the OS or a process manager.

## Wire Protocol

### Transport

- **Socket type:** Unix Domain Socket (stream mode, `SOCK_STREAM`)
- **Framing:** Newline-delimited JSON — one JSON-RPC message per line (`\n` terminated)
- **Encoding:** UTF-8
- **Max message size:** 1 MB (configurable via `max_message_bytes` in bus config)

### JSON-RPC 2.0

All communication uses [JSON-RPC 2.0](https://www.jsonrpc.org/specification). Each message is a single JSON object on one line.

**Request:**

```json
{"jsonrpc":"2.0","method":"bus.send","params":{"to":"cerebellum","payload":{"task":"analyze"}},"id":1}
```

**Response:**

```json
{"jsonrpc":"2.0","result":{"status":"delivered"},"id":1}
```

**Notification (no `id`, no response expected):**

```json
{"jsonrpc":"2.0","method":"bus.notify","params":{"event":"tick","data":{"cycle":42}}}
```

**Error:**

```json
{"jsonrpc":"2.0","error":{"code":-32001,"message":"component_not_found","data":{"name":"unknown"}},"id":1}
```

### Error Codes

Standard JSON-RPC 2.0 error codes apply. Additional bus-specific codes:

| Code | Name | Description |
|------|------|-------------|
| -32001 | `component_not_found` | Target component is not registered |
| -32002 | `already_registered` | Component name is already taken |
| -32003 | `not_registered` | Caller has not registered with the bus |
| -32004 | `topic_not_found` | Subscription topic does not exist |
| -32005 | `bus_full` | Bus has reached max component capacity |
| -32006 | `message_too_large` | Message exceeds `max_message_bytes` |
| -32007 | `rate_limited` | Component is sending too fast |

## Component Model

### Addressing

Each component has a unique **name** within the entity. Names are simple identifiers:

| Rule | Example |
|------|---------|
| Lowercase alphanumeric + hyphens | `cortex`, `cerebellum`, `amp-gateway` |
| 1–63 characters | — |
| Unique within the entity | No two components share a name |
| No dots or `@` signs | Prevents confusion with AMP addresses |

The bus process itself is implicitly addressed as `bus`.

### Roles

Components declare a role during registration:

| Role | Description |
|------|-------------|
| `worker` | General-purpose component (default) |
| `gateway` | Bridges internal bus to external AMP network |
| `coordinator` | Orchestrates other components |
| `monitor` | Observes bus traffic for logging/metrics |

Roles are informational — the bus does not enforce role-based access control. The exception is `gateway`: only one component per entity SHOULD register with the `gateway` role.

### Capabilities

Components declare capabilities during registration to advertise what they can do:

```json
{
  "name": "cortex",
  "role": "coordinator",
  "capabilities": ["reasoning", "planning", "delegation"]
}
```

Capabilities are free-form strings. They are discoverable via `bus.discover` so components can find peers by capability rather than by name.

## Bus Methods

### `bus.register`

Register a component with the bus. MUST be the first message sent after connecting.

**Request:**

```json
{
  "jsonrpc": "2.0",
  "method": "bus.register",
  "params": {
    "name": "cortex",
    "role": "coordinator",
    "capabilities": ["reasoning", "planning"],
    "version": "1.0.0"
  },
  "id": 1
}
```

| Param | Required | Description |
|-------|----------|-------------|
| `name` | Yes | Unique component name |
| `role` | No | Component role (default: `worker`) |
| `capabilities` | No | Array of capability strings |
| `version` | No | Component version string |

**Response:**

```json
{
  "jsonrpc": "2.0",
  "result": {
    "status": "registered",
    "heartbeat_interval_ms": 10000,
    "heartbeat_timeout_ms": 30000
  },
  "id": 1
}
```

**Errors:** `-32002` (already registered), `-32005` (bus full)

### `bus.deregister`

Gracefully remove a component from the bus.

**Request:**

```json
{
  "jsonrpc": "2.0",
  "method": "bus.deregister",
  "params": {},
  "id": 2
}
```

**Response:**

```json
{
  "jsonrpc": "2.0",
  "result": {
    "status": "deregistered"
  },
  "id": 2
}
```

The bus MUST notify other components via a `bus.notify` event with `"event": "component.left"`.

### `bus.heartbeat`

Liveness check. Components MUST send heartbeats at the interval specified during registration.

**Request:**

```json
{
  "jsonrpc": "2.0",
  "method": "bus.heartbeat",
  "params": {},
  "id": 3
}
```

**Response:**

```json
{
  "jsonrpc": "2.0",
  "result": {
    "status": "ok",
    "uptime_ms": 123456
  },
  "id": 3
}
```

If a component misses heartbeats for `heartbeat_timeout_ms` (default: 30000ms), the bus MUST treat it as disconnected and deregister it.

### `bus.discover`

List all connected components, optionally filtered by role or capability.

**Request:**

```json
{
  "jsonrpc": "2.0",
  "method": "bus.discover",
  "params": {
    "role": "worker",
    "capability": "reasoning"
  },
  "id": 4
}
```

| Param | Required | Description |
|-------|----------|-------------|
| `role` | No | Filter by role |
| `capability` | No | Filter by capability |

**Response:**

```json
{
  "jsonrpc": "2.0",
  "result": {
    "components": [
      {
        "name": "cortex",
        "role": "coordinator",
        "capabilities": ["reasoning", "planning"],
        "version": "1.0.0",
        "connected_at": "2025-01-30T10:00:00Z",
        "last_heartbeat": "2025-01-30T10:05:30Z"
      }
    ]
  },
  "id": 4
}
```

### `bus.send`

Send a point-to-point message to a specific component.

**Request:**

```json
{
  "jsonrpc": "2.0",
  "method": "bus.send",
  "params": {
    "to": "cerebellum",
    "payload": {
      "type": "task",
      "action": "analyze_pattern",
      "data": {"input": "sensor readings..."}
    }
  },
  "id": 5
}
```

| Param | Required | Description |
|-------|----------|-------------|
| `to` | Yes | Target component name |
| `payload` | Yes | Arbitrary JSON payload |

**Response:**

```json
{
  "jsonrpc": "2.0",
  "result": {
    "status": "delivered"
  },
  "id": 5
}
```

The recipient receives the message as a JSON-RPC notification:

```json
{
  "jsonrpc": "2.0",
  "method": "bus.message",
  "params": {
    "from": "cortex",
    "payload": {
      "type": "task",
      "action": "analyze_pattern",
      "data": {"input": "sensor readings..."}
    },
    "timestamp": "2025-01-30T10:05:31Z"
  }
}
```

**Errors:** `-32001` (component not found), `-32006` (message too large)

### `bus.broadcast`

Send a message to all connected components (except the sender).

**Request:**

```json
{
  "jsonrpc": "2.0",
  "method": "bus.broadcast",
  "params": {
    "payload": {
      "type": "announcement",
      "message": "shutting down in 10 seconds"
    }
  },
  "id": 6
}
```

**Response:**

```json
{
  "jsonrpc": "2.0",
  "result": {
    "delivered_to": ["cerebellum", "gateway", "memory"],
    "count": 3
  },
  "id": 6
}
```

### `bus.subscribe`

Subscribe to a named topic for pub/sub messaging.

**Request:**

```json
{
  "jsonrpc": "2.0",
  "method": "bus.subscribe",
  "params": {
    "topic": "sensor.updates"
  },
  "id": 7
}
```

**Response:**

```json
{
  "jsonrpc": "2.0",
  "result": {
    "status": "subscribed",
    "topic": "sensor.updates"
  },
  "id": 7
}
```

Topics are created on first subscription and removed when the last subscriber leaves.

### `bus.unsubscribe`

Unsubscribe from a topic.

**Request:**

```json
{
  "jsonrpc": "2.0",
  "method": "bus.unsubscribe",
  "params": {
    "topic": "sensor.updates"
  },
  "id": 8
}
```

**Response:**

```json
{
  "jsonrpc": "2.0",
  "result": {
    "status": "unsubscribed",
    "topic": "sensor.updates"
  },
  "id": 8
}
```

**Errors:** `-32004` (topic not found / not subscribed)

### `bus.publish`

Publish a message to all subscribers of a topic.

**Request:**

```json
{
  "jsonrpc": "2.0",
  "method": "bus.publish",
  "params": {
    "topic": "sensor.updates",
    "payload": {
      "sensor": "temperature",
      "value": 72.5,
      "unit": "F"
    }
  },
  "id": 9
}
```

**Response:**

```json
{
  "jsonrpc": "2.0",
  "result": {
    "topic": "sensor.updates",
    "delivered_to": 3
  },
  "id": 9
}
```

Subscribers receive the message as a notification:

```json
{
  "jsonrpc": "2.0",
  "method": "bus.topic",
  "params": {
    "topic": "sensor.updates",
    "from": "cortex",
    "payload": {
      "sensor": "temperature",
      "value": 72.5,
      "unit": "F"
    },
    "timestamp": "2025-01-30T10:05:32Z"
  }
}
```

### `bus.notify`

Fire-and-forget notification. No response is expected. Used by the bus itself to announce lifecycle events and by components for one-way signals.

**Bus lifecycle events:**

| Event | When | Data |
|-------|------|------|
| `component.joined` | A component registers | `{"name": "cortex", "role": "coordinator"}` |
| `component.left` | A component deregisters or times out | `{"name": "cortex", "reason": "deregistered"}` |
| `bus.shutdown` | Bus is shutting down | `{"timeout_ms": 5000}` |

**Example (bus → components):**

```json
{
  "jsonrpc": "2.0",
  "method": "bus.notify",
  "params": {
    "event": "component.joined",
    "data": {
      "name": "cerebellum",
      "role": "worker"
    },
    "timestamp": "2025-01-30T10:05:33Z"
  }
}
```

## AMP Gateway

The **gateway** is a special component that bridges the internal Local Bus to the external AMP network. It is the only component that holds the entity's AMP identity and communicates with AMP providers.

```
                    AMP Network
                        │
                        │  AMP REST/WS
                        ▼
┌─────────────────────────────────────────────┐
│  Gateway Component                           │
│                                              │
│  AMP Address: brain@acme.provider.ai         │
│  Private Key: ~/.agent-messaging/keys/       │
│                                              │
│  Responsibilities:                           │
│  - Send AMP messages on behalf of entity     │
│  - Receive AMP messages and route to bus     │
│  - Maintain provider connection (WS/polling) │
│  - Manage AMP identity and registration      │
└──────────────────────┬──────────────────────┘
                       │
                       │  Local Bus (UDS)
                       ▼
                    Bus Process
```

### Gateway Methods

These methods are exposed by the gateway component on the bus. Other components call them to interact with the AMP network.

#### `amp.send`

Send an AMP message to an external agent.

**Request:**

```json
{
  "jsonrpc": "2.0",
  "method": "amp.send",
  "params": {
    "to": "alice@acme.crabmail.ai",
    "subject": "Analysis complete",
    "priority": "normal",
    "payload": {
      "type": "response",
      "message": "Pattern analysis results are ready.",
      "context": {"task_id": "12345"}
    }
  },
  "id": 10
}
```

**Response:**

```json
{
  "jsonrpc": "2.0",
  "result": {
    "id": "msg_1706648400_abc123",
    "status": "delivered",
    "method": "websocket"
  },
  "id": 10
}
```

#### `amp.inbox`

Fetch the entity's AMP inbox.

**Request:**

```json
{
  "jsonrpc": "2.0",
  "method": "amp.inbox",
  "params": {
    "limit": 10,
    "status": "unread"
  },
  "id": 11
}
```

**Response:**

```json
{
  "jsonrpc": "2.0",
  "result": {
    "messages": [
      {
        "id": "msg_1706648400_def456",
        "from": "alice@acme.crabmail.ai",
        "subject": "New task",
        "priority": "high",
        "timestamp": "2025-01-30T10:00:00Z"
      }
    ],
    "count": 1
  },
  "id": 11
}
```

#### `amp.identity`

Get the entity's AMP identity.

**Request:**

```json
{
  "jsonrpc": "2.0",
  "method": "amp.identity",
  "params": {},
  "id": 12
}
```

**Response:**

```json
{
  "jsonrpc": "2.0",
  "result": {
    "address": "brain@acme.provider.ai",
    "fingerprint": "SHA256:...",
    "provider": "provider.ai",
    "tenant": "acme"
  },
  "id": 12
}
```

### Inbound Message Routing

When the gateway receives an AMP message from the external network, it publishes it on the `amp.inbound` topic so any interested component can process it:

```json
{
  "jsonrpc": "2.0",
  "method": "bus.topic",
  "params": {
    "topic": "amp.inbound",
    "from": "gateway",
    "payload": {
      "id": "msg_1706648400_def456",
      "envelope": {
        "from": "alice@acme.crabmail.ai",
        "to": "brain@acme.provider.ai",
        "subject": "New task",
        "priority": "high",
        "timestamp": "2025-01-30T10:00:00Z",
        "signature": "base64..."
      },
      "payload": {
        "type": "request",
        "message": "Please analyze these patterns."
      },
      "trust_level": "external"
    },
    "timestamp": "2025-01-30T10:00:01Z"
  }
}
```

The gateway MUST apply the same content security rules as standard AMP (see [07 - Security](07-security.md)): external messages are wrapped in `<external-content>` tags and injection patterns are flagged.

## Lifecycle

### Bus Startup

1. Bus process creates the socket directory (e.g., `/tmp/amp-bus-brain/`)
2. Bus process creates the UDS at the configured path (e.g., `/tmp/amp-bus-brain/bus.sock`)
3. Bus process sets directory permissions to `0700` (owner only)
4. Bus process writes a PID file at `/tmp/amp-bus-brain/bus.pid`
5. Bus begins accepting connections

### Component Registration

1. Component connects to the bus socket
2. Component sends `bus.register` as its first message
3. Bus validates the name is unique and responds with heartbeat parameters
4. Bus broadcasts `component.joined` to all other components
5. Component begins sending heartbeats at the specified interval

If a component sends any method other than `bus.register` before registering, the bus MUST respond with error `-32003` (not registered).

### Graceful Shutdown

**Component shutdown:**

1. Component sends `bus.deregister`
2. Bus removes the component from its registry
3. Bus broadcasts `component.left` with `"reason": "deregistered"`
4. Component closes its socket connection

**Bus shutdown:**

1. Bus broadcasts `bus.notify` with `"event": "bus.shutdown"` and a timeout
2. Bus waits for components to deregister (up to the timeout)
3. Bus closes all connections
4. Bus removes the socket file and PID file

### Crash Recovery

**Component crash:**

- The bus detects a closed socket or missed heartbeats
- The bus deregisters the component and broadcasts `component.left` with `"reason": "timeout"` or `"reason": "disconnected"`
- The component MAY reconnect and re-register with the same name

**Bus crash:**

- All component connections are broken (OS closes the socket)
- Components detect the broken connection and enter a reconnect loop
- Components SHOULD use exponential backoff: 100ms, 200ms, 400ms, ..., up to 10s
- When the bus restarts, components reconnect and re-register
- No message recovery — the Local Bus does not persist messages

## Security

The Local Bus security model relies on **OS-level process isolation** rather than cryptographic authentication. Since all components run on the same machine under the same user, the trust boundary is the operating system.

### Socket Permissions

| Path | Permissions | Purpose |
|------|-------------|---------|
| `/tmp/amp-bus-<entity>/` | `0700` | Socket directory — owner only |
| `/tmp/amp-bus-<entity>/bus.sock` | `0700` | Socket file — owner only |
| `/tmp/amp-bus-<entity>/bus.pid` | `0644` | PID file — readable for monitoring |

Only processes running as the same OS user can connect to the socket. This provides the same level of isolation as SSH agent sockets.

### No Internal Authentication

Components do NOT authenticate to the bus. The trust model:

- **Same user = trusted.** If a process can connect to the socket, it is part of the entity.
- **No signatures on bus messages.** Cryptographic signing is unnecessary for same-machine IPC under a single user.
- **AMP signatures are gateway-only.** The gateway signs outbound AMP messages with the entity's private key.

### Content Security on Inbound AMP

When the gateway bridges an external AMP message onto the Local Bus, it MUST:

1. Apply content security wrapping per [07 - Security](07-security.md)
2. Include the `trust_level` field (`verified`, `external`, or `untrusted`)
3. Flag any detected injection patterns in metadata

Internal bus messages (component-to-component) are not subject to content security wrapping.

## Configuration

### Bus Configuration File

Location: `~/.agent-messaging/bus.json` (or specified via `AMP_BUS_CONFIG` environment variable)

```json
{
  "entity": "brain",
  "socket_dir": "/tmp/amp-bus-brain",
  "max_components": 32,
  "max_message_bytes": 1048576,
  "heartbeat_interval_ms": 10000,
  "heartbeat_timeout_ms": 30000,
  "shutdown_timeout_ms": 5000,
  "gateway": {
    "amp_dir": "~/.agent-messaging",
    "auto_fetch_interval_ms": 30000
  }
}
```

| Field | Default | Description |
|-------|---------|-------------|
| `entity` | Required | Entity name (used in socket path) |
| `socket_dir` | `/tmp/amp-bus-<entity>` | Directory for socket and PID files |
| `max_components` | 32 | Maximum simultaneous components |
| `max_message_bytes` | 1048576 (1 MB) | Maximum JSON-RPC message size |
| `heartbeat_interval_ms` | 10000 | How often components must heartbeat |
| `heartbeat_timeout_ms` | 30000 | How long before a silent component is dropped |
| `shutdown_timeout_ms` | 5000 | Grace period during bus shutdown |
| `gateway.amp_dir` | `~/.agent-messaging` | AMP identity directory for the gateway |
| `gateway.auto_fetch_interval_ms` | 30000 | How often the gateway polls for new AMP messages |

### Socket Path Conventions

```
/tmp/amp-bus-<entity>/bus.sock
```

The `<entity>` segment MUST match the `entity` field in the bus config and MUST follow the same naming rules as component names (lowercase alphanumeric + hyphens, 1–63 chars).

### Environment Variables

| Variable | Description |
|----------|-------------|
| `AMP_BUS_CONFIG` | Path to bus configuration file |
| `AMP_BUS_SOCKET` | Override socket path (bypasses config) |
| `AMP_BUS_ENTITY` | Override entity name |

## Example Flows

### Brain Coordination

A coordinator delegates work to specialized components:

```
  cortex              Bus              cerebellum         memory
    │                  │                    │                │
    │  bus.discover    │                    │                │
    │  (cap=analysis)  │                    │                │
    │─────────────────>│                    │                │
    │                  │                    │                │
    │  [cerebellum]    │                    │                │
    │<─────────────────│                    │                │
    │                  │                    │                │
    │  bus.send        │                    │                │
    │  to=cerebellum   │   bus.message      │                │
    │  {analyze:data}  │   from=cortex      │                │
    │─────────────────>│───────────────────>│                │
    │                  │                    │                │
    │                  │   bus.send         │                │
    │   bus.message    │   to=memory        │   bus.message  │
    │   from=cerebellum│   {store:results}  │   from=cerebellum
    │<─────────────────│<───────────────────│───────────────>│
    │                  │                    │                │
```

### External Message Handling

An external AMP message arrives and is processed internally:

```
  AMP Network        gateway            Bus              cortex
      │                 │                 │                  │
      │  AMP message    │                 │                  │
      │  from: alice@.. │                 │                  │
      │────────────────>│                 │                  │
      │                 │                 │                  │
      │                 │  bus.publish     │                  │
      │                 │  topic=amp.inbound                 │
      │                 │  {envelope,payload}                │
      │                 │────────────────>│   bus.topic      │
      │                 │                 │   amp.inbound    │
      │                 │                 │─────────────────>│
      │                 │                 │                  │
      │                 │                 │   bus.send       │
      │                 │   bus.message   │   to=gateway     │
      │                 │   {amp.send:..} │   {amp.send      │
      │  AMP reply      │<───────────────│<──reply to alice} │
      │  to: alice@..   │                 │                  │
      │<────────────────│                 │                  │
      │                 │                 │                  │
```

### Pub/Sub Sensor Updates

Components subscribe to a shared data stream:

```
  sensor             Bus              cortex          cerebellum
    │                  │                  │                │
    │                  │   bus.subscribe  │                │
    │                  │   sensor.updates │                │
    │                  │<─────────────────│                │
    │                  │                  │                │
    │                  │   bus.subscribe  │                │
    │                  │   sensor.updates │   bus.subscribe│
    │                  │<─────────────────────────────────│
    │                  │                  │                │
    │  bus.publish     │                  │                │
    │  sensor.updates  │   bus.topic      │   bus.topic    │
    │  {temp: 72.5}    │   sensor.updates │   sensor.updates
    │─────────────────>│─────────────────>│───────────────>│
    │                  │                  │                │
```

## Relationship to AMP

| Aspect | AMP (Sections 01–09) | Local Bus (Section 10) |
|--------|----------------------|------------------------|
| Scope | Inter-entity (across machines/providers) | Intra-entity (single machine) |
| Transport | HTTP REST, WebSocket | Unix Domain Socket |
| Protocol | Custom envelope + payload | JSON-RPC 2.0 |
| Identity | `name@tenant.provider` | Simple name (`cortex`) |
| Security | Ed25519 signatures, TLS | OS file permissions |
| Storage | Local filesystem (persistent) | None (ephemeral) |
| Discovery | Provider registry, DNS | Bus `discover` method |

---

Previous: [09 - External Agents](09-external-agents.md) | Next: [Appendix A - Injection Patterns](appendix-a-injection-patterns.md)
