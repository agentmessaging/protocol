# 10 - Local Bus

**Status:** Draft
**Version:** 0.1.2

## Overview

AMP defines **inter-entity** messaging — agents communicating across networks via providers. But when multiple components operate as a single entity (e.g., a "brain" composed of `cortex`, `cerebellum`, and `gateway` processes), they need fast, ordered **intra-entity** communication.

The Local Bus is a filesystem-based mailbox system that enables components within a single entity to exchange messages through shared folders. Each component has its own mailbox directory, processes messages in order, and deletes them after handling. No central bus process is required — components operate autonomously using only file I/O.

A dedicated gateway component bridges the internal bus to the external AMP network, presenting a single AMP identity for the entire entity.

### What Local Bus Is

- A same-machine IPC mechanism for components of a single AMP entity
- A filesystem-based mailbox system — write to send, watch to receive
- Zero dependencies beyond file I/O — works with bash, Python, Node.js, or any language
- A complement to AMP, not a replacement

### What Local Bus Is NOT

- NOT a replacement for AMP inter-entity messaging
- NOT a network protocol — it operates exclusively on a single machine
- NOT a real-time streaming system — polling-based with configurable intervals

## Architecture

The Local Bus uses a **mailbox-per-component** model. Each component owns a directory under `mailbox/`. To send a message, a component writes a JSON file to the recipient's mailbox directory. To receive, a component watches its own mailbox and processes files in order.

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
│     watches its          watches its           watches its      │
│     own mailbox          own mailbox           own mailbox      │
│           │                   │                     │            │
│           ▼                   ▼                     ▼            │
│   mailbox/cortex/    mailbox/cerebellum/    mailbox/gateway/    │
│                                                                 │
│  Bus dir: ~/.agent-messaging/bus/brain/                         │
└─────────────────────────────────────────────────────────────────┘
```

### Why Filesystem Mailboxes

| Property | Filesystem (chosen) | Unix Domain Sockets | TCP/HTTP |
|----------|-------------------|-------------------|----------|
| OS support | All (macOS, Linux, Windows) | macOS/Linux only | All |
| Dependencies | None (file I/O only) | Socket libraries | HTTP libraries |
| Crash recovery | Messages survive on disk | Messages lost in memory | Messages lost |
| Ordering | Filename sort | Must implement in-memory | Must implement |
| Central process | Not needed | Bus process required | Server required |
| Debuggability | `ls` and `cat` | Requires tools | Requires tools |
| AI agent can implement | Yes — basic file read/write | Needs socket programming | Needs HTTP stack |
| Backpressure | Natural — files queue up | Must implement | Must implement |

## Directory Structure

All Local Bus state lives under `~/.agent-messaging/bus/<entity>/`:

```
~/.agent-messaging/bus/brain/
├── bus.json                              # Bus configuration
├── components/                           # Component registry
│   ├── cortex.json                       # Registration + heartbeat
│   ├── cerebellum.json
│   └── gateway.json
├── mailbox/                              # Per-component message queues
│   ├── cortex/
│   │   ├── 1706648400123_a1b2c3d4.json   # Message files (sorted = processing order)
│   │   └── 1706648400456_e5f6a7b8.json
│   ├── cerebellum/
│   │   └── 1706648400789_c9d0e1f2.json
│   └── gateway/
└── topics/                               # Pub/sub subscriber lists
    ├── sensor.updates.json
    └── amp.inbound.json
```

### Directory Permissions

| Path | Permissions | Purpose |
|------|-------------|---------|
| `~/.agent-messaging/bus/<entity>/` | `0700` | Bus root — owner only |
| `components/` | `0700` | Component registry |
| `mailbox/` | `0700` | All mailboxes |
| `mailbox/<component>/` | `0700` | Individual mailbox |
| `topics/` | `0700` | Topic subscriber lists |

All directories and files MUST be owned by the same OS user. This provides the trust boundary — if a process can write to the mailbox, it is part of the entity.

## Message Format

### Filename Convention

```
<unix_timestamp_ms>_<random_hex>.json
```

- **`unix_timestamp_ms`**: Unix timestamp in milliseconds (e.g., `1706648400123`)
- **`random_hex`**: 8 hex characters for collision avoidance (e.g., `a1b2c3d4`)

Sorting filenames lexicographically produces **chronological processing order**. This is the only ordering guarantee the Local Bus provides.

Examples:

```
1706648400123_a1b2c3d4.json   ← processed first
1706648400456_e5f6a7b8.json   ← processed second
1706648401001_c9d0e1f2.json   ← processed third
```

### Message Schema

Every message file contains a single JSON object:

```json
{
  "id": "bus_1706648400123_a1b2c3d4",
  "from": "cortex",
  "method": "bus.send",
  "payload": {
    "type": "task",
    "action": "analyze_pattern",
    "data": {"input": "sensor readings..."}
  },
  "timestamp": "2025-01-30T10:05:31.123Z",
  "topic": null
}
```

| Field | Required | Description |
|-------|----------|-------------|
| `id` | Yes | Unique message ID: `bus_<timestamp_ms>_<random_hex>` |
| `from` | Yes | Sender component name |
| `method` | Yes | Operation type (see [Operations](#operations)) |
| `payload` | Yes | Arbitrary JSON content |
| `timestamp` | Yes | ISO 8601 timestamp with milliseconds |
| `topic` | No | Topic name if this is a pub/sub message, `null` otherwise |

### Atomic Writes

To prevent partial reads, messages MUST be written atomically:

1. Write the JSON to a temporary file in the same directory (e.g., `.tmp_<random>.json`)
2. Rename (move) the temporary file to the final filename

Rename is atomic on all POSIX systems and on Windows (NTFS). This guarantees that a reader never sees a half-written file.

```bash
# Example: atomic write in bash
tmp_file="mailbox/cerebellum/.tmp_$$_$(openssl rand -hex 4).json"
final_file="mailbox/cerebellum/1706648400123_a1b2c3d4.json"
echo '{"id":"bus_1706648400123_a1b2c3d4",...}' > "$tmp_file"
mv "$tmp_file" "$final_file"
```

## Component Model

### Addressing

Each component has a unique **name** within the entity. Names are simple identifiers:

| Rule | Example |
|------|---------|
| Lowercase alphanumeric + hyphens | `cortex`, `cerebellum`, `amp-gateway` |
| 1–63 characters | — |
| Unique within the entity | No two components share a name |
| No dots or `@` signs | Prevents confusion with AMP addresses |

### Component Registration File

Each component creates a JSON file at `components/<name>.json` when it joins the bus:

```json
{
  "name": "cortex",
  "role": "coordinator",
  "capabilities": ["reasoning", "planning", "delegation"],
  "version": "1.0.0",
  "pid": 12345,
  "registered_at": "2025-01-30T10:00:00Z",
  "last_seen": "2025-01-30T10:05:30Z"
}
```

| Field | Required | Description |
|-------|----------|-------------|
| `name` | Yes | Component name (matches filename) |
| `role` | No | One of: `worker` (default), `gateway`, `coordinator`, `monitor` |
| `capabilities` | No | Array of free-form capability strings |
| `version` | No | Component version string |
| `pid` | Yes | OS process ID (for liveness checks) |
| `registered_at` | Yes | When the component registered |
| `last_seen` | Yes | Updated on each heartbeat cycle |

### Roles

| Role | Description |
|------|-------------|
| `worker` | General-purpose component (default) |
| `gateway` | Bridges internal bus to external AMP network |
| `coordinator` | Orchestrates other components |
| `monitor` | Observes bus traffic for logging/metrics |

Roles are informational. The exception is `gateway`: only one component per entity SHOULD register with the `gateway` role.

### Discovery

To discover other components, read the `components/` directory:

```bash
# List all components
ls ~/.agent-messaging/bus/brain/components/

# Find components with a specific capability
cat ~/.agent-messaging/bus/brain/components/*.json | \
  jq 'select(.capabilities[] == "reasoning") | .name'
```

A component is considered **alive** if:
1. Its `components/<name>.json` file exists, AND
2. Its `last_seen` timestamp is within the heartbeat timeout (default: 30s), OR
3. Its `pid` corresponds to a running process (`kill -0 <pid>`)

## Operations

### Send (Point-to-Point)

To send a message to a specific component, write a JSON file to the recipient's mailbox:

```bash
# cortex sends to cerebellum
cat > "mailbox/cerebellum/1706648400123_a1b2c3d4.json" <<'EOF'
{
  "id": "bus_1706648400123_a1b2c3d4",
  "from": "cortex",
  "method": "bus.send",
  "payload": {
    "type": "task",
    "action": "analyze_pattern",
    "data": {"input": "sensor readings..."}
  },
  "timestamp": "2025-01-30T10:05:31.123Z",
  "topic": null
}
EOF
```

The sender MUST use atomic writes (write to temp file, then rename).

If the recipient's mailbox directory does not exist, the message is **undeliverable**. The sender SHOULD check that `mailbox/<target>/` exists before writing. If it does not exist, the component is not registered.

### Receive (Processing Loop)

Each component runs a processing loop on its own mailbox:

1. List all `.json` files in `mailbox/<self>/`, sorted by filename (ascending)
2. Read the oldest file
3. Process the message
4. Delete the file
5. Repeat

```bash
# cerebellum processing loop
while true; do
  for msg_file in $(ls mailbox/cerebellum/*.json 2>/dev/null | sort); do
    # Process
    payload=$(jq '.payload' "$msg_file")
    handle_message "$payload"
    # ACK by deleting
    rm "$msg_file"
  done
  sleep 0.1  # poll interval
done
```

**Processing guarantees:**

- Messages are processed in **filename order** (chronological)
- A message is processed **at most once** (deleted after handling)
- If a component crashes mid-processing, the current message file may still exist. On restart, the component re-processes it (at-least-once for that message). Implementations MAY track processed message IDs to achieve exactly-once semantics.

### Broadcast

To broadcast to all components, write the message to every component's mailbox (except your own):

```bash
# cortex broadcasts to all
for dir in mailbox/*/; do
  component=$(basename "$dir")
  [ "$component" = "cortex" ] && continue  # skip self
  cp_atomic "$message_file" "$dir/$(timestamp_filename)"
done
```

### Heartbeat

Components update their `last_seen` timestamp in `components/<name>.json` at a regular interval (default: every 10 seconds):

```bash
# Update heartbeat
jq '.last_seen = now | todate' components/cortex.json > components/.tmp_cortex.json
mv components/.tmp_cortex.json components/cortex.json
```

A component MAY check other components' `last_seen` timestamps to detect failures. A component whose `last_seen` exceeds the heartbeat timeout (default: 30s) SHOULD be considered dead. Its mailbox directory is preserved (messages persist), and a replacement component can re-register with the same name to resume processing.

### Pub/Sub

#### Subscribe

Add your component name to the topic's subscriber list at `topics/<topic>.json`:

```json
{
  "topic": "sensor.updates",
  "subscribers": ["cortex", "cerebellum"],
  "created_at": "2025-01-30T10:00:00Z"
}
```

If the topic file doesn't exist, create it. Use atomic writes and file locking when updating the subscriber list to avoid concurrent modification.

#### Publish

Read the subscriber list and write the message to each subscriber's mailbox with the `topic` field set:

```json
{
  "id": "bus_1706648400789_f1e2d3c4",
  "from": "sensor",
  "method": "bus.publish",
  "payload": {
    "sensor": "temperature",
    "value": 72.5,
    "unit": "F"
  },
  "timestamp": "2025-01-30T10:05:32.789Z",
  "topic": "sensor.updates"
}
```

The publisher fans out the message: one file is written to each subscriber's mailbox. Subscribers process topic messages the same way as point-to-point messages — they arrive in the same mailbox and are ordered by filename.

#### Unsubscribe

Remove your component name from `topics/<topic>.json`. If the subscriber list becomes empty, the topic file MAY be deleted.

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
│  - Receive AMP messages, deliver to bus      │
│  - Maintain provider connection (WS/polling) │
│  - Manage AMP identity and registration      │
└──────────────────────┬──────────────────────┘
                       │
                       │  Filesystem (mailbox)
                       ▼
              mailbox/gateway/
```

### Sending AMP Messages

Any component can send an AMP message by writing to the gateway's mailbox with `method: "amp.send"`:

```json
{
  "id": "bus_1706648400123_a1b2c3d4",
  "from": "cortex",
  "method": "amp.send",
  "payload": {
    "to": "alice@acme.crabmail.ai",
    "subject": "Analysis complete",
    "priority": "normal",
    "payload": {
      "type": "response",
      "message": "Pattern analysis results are ready.",
      "context": {"task_id": "12345"}
    }
  },
  "timestamp": "2025-01-30T10:05:31.123Z",
  "topic": null
}
```

The gateway processes this by:

1. Reading the message from its mailbox
2. Constructing an AMP envelope using the entity's identity
3. Signing the message with the entity's Ed25519 private key
4. Sending via the AMP provider's `/route` endpoint
5. Optionally writing a confirmation back to the sender's mailbox

### Receiving AMP Messages

When the gateway receives an AMP message from the external network (via WebSocket, webhook, or polling), it publishes to the `amp.inbound` topic by writing to each subscriber's mailbox:

```json
{
  "id": "bus_1706648400456_e5f6a7b8",
  "from": "gateway",
  "method": "amp.inbound",
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
  "timestamp": "2025-01-30T10:00:01.000Z",
  "topic": "amp.inbound"
}
```

The gateway MUST apply the same content security rules as standard AMP (see [07 - Security](07-security.md)): external messages are wrapped in `<external-content>` tags and injection patterns are flagged.

### Getting AMP Identity

Components can read the entity's AMP identity directly from the AMP configuration:

```bash
cat ~/.agent-messaging/config.json | jq '{address, fingerprint, provider, tenant}'
```

Or write a `method: "amp.identity"` message to the gateway's mailbox and receive a response.

### Standalone Mode (No Provider)

The gateway is **optional**. An entity can operate without any AMP provider connection — components communicate entirely via the Local Bus. This is useful for:

- Local development and testing
- Air-gapped environments
- Single-machine multi-agent systems that don't need external messaging

When no gateway is registered, `amp.*` methods are simply not available.

## Lifecycle

### Initialization

When the first component starts for an entity:

1. Create the bus directory tree:
   ```bash
   mkdir -p ~/.agent-messaging/bus/brain/{components,mailbox,topics}
   chmod 700 ~/.agent-messaging/bus/brain
   ```
2. Create `bus.json` if it doesn't exist (see [Configuration](#configuration))

If the directories already exist (another component is running or previously ran), skip creation.

### Component Registration

1. Create `mailbox/<name>/` directory for your inbox
2. Write `components/<name>.json` with role, capabilities, PID, and timestamps
3. Begin heartbeat loop (update `last_seen` every 10s)
4. Begin mailbox processing loop

```bash
# Register "cortex"
mkdir -p "mailbox/cortex"
cat > "components/cortex.json" <<'EOF'
{
  "name": "cortex",
  "role": "coordinator",
  "capabilities": ["reasoning", "planning"],
  "version": "1.0.0",
  "pid": 12345,
  "registered_at": "2025-01-30T10:00:00Z",
  "last_seen": "2025-01-30T10:00:00Z"
}
EOF
```

### Graceful Shutdown

1. Stop the mailbox processing loop
2. Process any remaining messages in the mailbox (drain)
3. Remove `components/<name>.json`
4. Optionally remove `mailbox/<name>/` if empty, or leave it for crash recovery

### Crash Recovery

Since messages are files on disk, crash recovery is straightforward:

- **Component crash:** The mailbox directory and any unprocessed messages survive. When the component restarts, it re-registers (overwrites `components/<name>.json`) and resumes processing its mailbox from where it left off.
- **Machine crash:** All bus state survives on disk. Components restart and resume processing. No messages are lost.

A component that restarts SHOULD:

1. Check if `mailbox/<name>/` has pending messages
2. Process them in filename order before accepting new work
3. Re-register in `components/<name>.json` with a new PID

### Stale Component Cleanup

A component is **stale** if:
- Its `last_seen` exceeds the heartbeat timeout, AND
- Its `pid` does not correspond to a running process

Any component MAY clean up stale registrations by removing the stale component's `components/<name>.json`. The stale component's mailbox SHOULD be preserved — it may contain unprocessed messages that the component will handle on restart.

## Security

The Local Bus security model relies on **OS-level file permissions** rather than cryptographic authentication.

### Trust Model

- **Same user = trusted.** If a process can read/write the bus directory, it is part of the entity.
- **No signatures on bus messages.** Cryptographic signing is unnecessary for same-machine IPC under a single user.
- **AMP signatures are gateway-only.** The gateway signs outbound AMP messages with the entity's private key.

### Content Security on Inbound AMP

When the gateway bridges an external AMP message onto the Local Bus, it MUST:

1. Apply content security wrapping per [07 - Security](07-security.md)
2. Include the `trust_level` field (`verified`, `external`, or `untrusted`)
3. Flag any detected injection patterns in metadata

Internal bus messages (component-to-component) are not subject to content security wrapping.

### Temporary Files

Temporary files (used during atomic writes) MUST be created in the same directory as the target file and SHOULD use a dot-prefix (e.g., `.tmp_<random>.json`) so they are easily distinguished from real messages. Components MUST ignore dot-prefixed files when listing their mailbox.

## Configuration

### Bus Configuration File

Location: `~/.agent-messaging/bus/<entity>/bus.json`

```json
{
  "entity": "brain",
  "version": "1.0",
  "heartbeat_interval_ms": 10000,
  "heartbeat_timeout_ms": 30000,
  "poll_interval_ms": 100,
  "max_message_bytes": 1048576,
  "max_components": 32,
  "gateway": {
    "amp_dir": "~/.agent-messaging",
    "auto_fetch_interval_ms": 30000
  }
}
```

| Field | Default | Description |
|-------|---------|-------------|
| `entity` | Required | Entity name (used in directory path) |
| `version` | `"1.0"` | Bus protocol version |
| `heartbeat_interval_ms` | 10000 | How often components update `last_seen` |
| `heartbeat_timeout_ms` | 30000 | When a component is considered stale |
| `poll_interval_ms` | 100 | How often to check mailbox for new messages |
| `max_message_bytes` | 1048576 (1 MB) | Maximum message file size |
| `max_components` | 32 | Maximum registered components |
| `gateway.amp_dir` | `~/.agent-messaging` | AMP identity directory for the gateway |
| `gateway.auto_fetch_interval_ms` | 30000 | How often the gateway polls for AMP messages |

### Environment Variables

| Variable | Description |
|----------|-------------|
| `AMP_BUS_DIR` | Override bus root directory (default: `~/.agent-messaging/bus`) |
| `AMP_BUS_ENTITY` | Override entity name |

## Example Flows

### Brain Coordination

A coordinator delegates work to specialized components via mailbox files:

```
  cortex                                cerebellum          memory
    │                                       │                 │
    │  ls components/*.json                 │                 │
    │  → finds cerebellum (cap=analysis)    │                 │
    │                                       │                 │
    │  write to mailbox/cerebellum/         │                 │
    │  {action: "analyze", data: ...}       │                 │
    │──────────────────────────────────────>│                 │
    │                                       │                 │
    │                                       │  processes msg  │
    │                                       │  writes result  │
    │                                       │  to mailbox/    │
    │  reads from                           │  memory/        │
    │  mailbox/cortex/                      │  {store: ...}   │
    │  {status: "done", result: ...}        │────────────────>│
    │<──────────────────────────────────────│                 │
    │                                       │                 │
```

### External Message Handling

An external AMP message arrives and is processed by internal components:

```
  AMP Network        gateway                            cortex
      │                 │                                  │
      │  AMP message    │                                  │
      │  from: alice@.. │                                  │
      │────────────────>│                                  │
      │                 │                                  │
      │                 │  reads amp.inbound subscribers   │
      │                 │  writes to mailbox/cortex/       │
      │                 │  {amp.inbound, envelope, payload}│
      │                 │─────────────────────────────────>│
      │                 │                                  │
      │                 │  reads from mailbox/gateway/     │
      │                 │  {method: "amp.send",            │
      │  AMP reply      │   to: "alice@...",               │
      │  to: alice@..   │   payload: reply}                │
      │<────────────────│<─────────────────────────────────│
      │                 │                                  │
```

### Ordered Processing (Queuing)

Cortex sends two tasks while cerebellum is busy — messages queue naturally:

```
  cortex                                     cerebellum
    │                                            │
    │  write mailbox/cerebellum/                 │
    │  1706648400100_a1b2.json  {task: "speak A"}│  ← processing this
    │───────────────────────────────────────────>│
    │                                            │
    │  write mailbox/cerebellum/                 │
    │  1706648400200_c3d4.json  {task: "speak B"}│  ← queued on disk
    │───────────────────────────────────────────>│
    │                                            │
    │                                            │  finishes "speak A"
    │                                            │  deletes ...100_a1b2.json
    │                                            │
    │                                            │  picks up ...200_c3d4.json
    │                                            │  starts "speak B"
    │                                            │
```

This is the key advantage of filesystem mailboxes: **backpressure and ordering are free**. Messages accumulate as files, processed one at a time in chronological order. No messages are lost, even if the receiver is slow or temporarily down.

## Relationship to AMP

| Aspect | AMP (Sections 01–09) | Local Bus (Section 10) |
|--------|----------------------|------------------------|
| Scope | Inter-entity (across machines/providers) | Intra-entity (single machine) |
| Transport | HTTP REST, WebSocket | Filesystem (read/write JSON files) |
| Protocol | AMP envelope + payload | JSON message files in mailbox dirs |
| Identity | `name@tenant.provider` | Simple name (`cortex`) |
| Security | Ed25519 signatures, TLS | OS file permissions (0700) |
| Storage | Local filesystem (persistent) | Filesystem (persistent until processed) |
| Discovery | Provider registry, DNS | Read `components/` directory |
| Ordering | Per-sender via `in_reply_to` | Filename sort (global chronological) |
| Crash recovery | Messages persist in inbox | Messages persist in mailbox |
| Dependencies | curl, openssl, jq | File I/O only |

---

Previous: [09 - External Agents](09-external-agents.md) | Next: [Appendix A - Injection Patterns](appendix-a-injection-patterns.md)
