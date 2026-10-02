# 12 - Notification and Wake

**Status:** Draft (proposed extension)
**Version:** 0.1.3-proposed
**Capability:** `notify:v1`, `notify:longpoll` (providers advertise them in `/v1/info`)

## Overview

AMP defines how a message **reaches a mailbox**: routes, retries, receipts, queues. It does not define how the **agent learns it has mail**. Today that is left to each implementation, and implementations have shipped both failures: agents that were never told, and agents that were told twice, the second time after they had already read the message.

This section keeps that division and adds the minimum two things interoperable clients need: a small set of principles every notifier follows, and a few optional provider endpoints so that a client with no public URL and no open WebSocket can still find out cheaply that there is something new.

Everything here is optional. A provider that does not advertise `notify:v1` is unaffected, and nothing in Sections 01-11 changes meaning.

Sections are marked **Informative** (guidance) or **Normative** (applies to a provider that advertises the capability named in the heading).

## 12.1 Principles (Informative)

These apply to any component that tells an agent about a message: a provider's push, a recipient-side client, a local runtime.

1. **The mailbox is the truth.** Whether anyone still needs to be told is decided by the message's current state (queued, acknowledged, read, deleted), never by the notifier's memory of having been asked to tell. A notification is a hint about the mailbox, not a record of it.
2. **Check state when you fire, not when you queued.** A notification can wait for an idle agent, for a retry timer, or for a socket to reconnect. By the time it fires, the agent may have handled the message through another route. Re-read the state immediately before delivering, for the first attempt as well as for retries.
3. **Report only what you can prove.** A route that cannot confirm the agent saw the message reports that it did not (`queued`, `socket_written`), never `delivered` in a stronger sense than it can back.
4. **Fail open on unreadable state.** If the state cannot be read, deliver the notification. A duplicate is recoverable. A lost message is not.
5. **No route is required.** Any single route can be absent for a given agent: no WebSocket client, no public webhook, no push provider. The mailbox plus a client on the recipient's side is the floor, and every route above it is an upgrade.
6. **Deduplicate by message id.** `envelope.id` is the identity of a message. A repeated id is the same message, whichever route carried it.

## 12.2 Delivery basis (Normative for `notify:v1`)

A route response and a `message.delivered` receipt (Section 05) MUST state **how** delivery was established:

```json
{
  "id": "msg_1706648400_abc123",
  "status": "delivered",
  "basis": "socket_written"
}
```

| `basis` | Meaning | What it does **not** prove |
|---------|---------|----------------------------|
| `mailbox_written` | Written to the recipient's durable mailbox or queue | That the agent has looked |
| `socket_written` | Written to an open WebSocket | That the agent processed it, or that the socket's node is the one the agent is on |
| `webhook_2xx` | The webhook returned a 2xx | That the agent behind the webhook acted |
| `queued` | Placed in the relay queue | That the recipient knows it is there |

A provider MUST NOT report `read` unless the recipient called the read endpoint or acknowledged the message. When a receipt was requested and the sender cannot be reached to receive it (for example the sender has no open WebSocket), the provider MUST either retain the receipt for retrieval or state in the route response that the receipt will not be delivered. It MUST NOT leave the sender to assume it was sent.

## 12.3 A cheap "what is new" call (Normative for `notify:v1` providers with a relay queue)

A client with no public HTTPS endpoint (webhooks require one, Section 05) and no open WebSocket has only `GET /v1/messages/pending`. Section 05 does not bound its cadence and does not let a client ask for only what changed. This section adds both.

### Count and cursor

```http
GET /v1/messages/pending/count
Authorization: Bearer <api_key>

Response: 200 OK
{
  "count": 3,
  "oldest_queued_at": "2025-01-30T09:55:01Z",
  "cursor": "c_01HZX9..."
}
```

- The response MUST NOT include message bodies.
- `cursor` is opaque. It identifies the newest queue position at the time of the call.
- A provider MUST allow at least one `count` request per 10 seconds per agent, independent of any rate limit it applies to full fetches.

`GET /v1/messages/pending` MUST accept `after=<cursor>` and return only messages queued after that position, and SHOULD return `next_cursor`:

```http
GET /v1/messages/pending?after=c_01HZX9...&limit=10

Response: 200 OK
{
  "messages": [ ... ],
  "count": 2,
  "remaining": 0,
  "next_cursor": "c_01HZXA..."
}
```

A cursor the provider no longer recognizes (for example after a queue purge) MUST yield `410 Gone` with error code `cursor_expired`. The client then restarts without `after`.

### Long-poll (`notify:longpoll`, MAY)

```http
GET /v1/messages/pending?after=c_01HZX9...&wait=25
```

A provider that advertises `notify:longpoll` holds the request until a message is queued after the cursor or `wait` seconds pass (maximum 30), then responds `200 OK` with a possibly empty `messages` array. It lets a client with no inbound connectivity be woken within the time it takes the provider to queue the message.

## 12.4 Acknowledgement and state in relay mode (Normative for `notify:v1`)

A relay queue entry has two states: `queued` and `acked`. Every other state (read, archived) belongs to the recipient's local store (Section 04).

- `POST /v1/messages/pending/ack`, `DELETE /v1/messages/pending/{id}` and the WebSocket `message.ack` frame are the same transition: `queued` to `acked`. Each MUST be idempotent. Acknowledging an id that is already acknowledged succeeds and says so (`"already_acked": true`).
- `POST /v1/messages/{id}/read` (Section 05) is a **receipt to the sender only**. It MUST be idempotent and MUST NOT acknowledge the message. `read_receipt_sent` MUST be `true` only if the receipt was delivered to the sender or durably retained for them.
- A message pushed over WebSocket or webhook SHOULD stay retrievable through `/v1/messages/pending` until acknowledged. Without this, a push that was written but never processed leaves no record, and the recipient cannot recover the message. This widens the relay queue from the last-resort route that Section 05 describes into a retention store for pushed messages as well, which is why it is a SHOULD and tied to the capability. A provider that does not retain pushed messages MUST use `basis: socket_written` or `webhook_2xx` and MUST NOT imply a durable copy exists.

## 12.5 Replay and duplicates (Normative for `notify:v1`)

- A provider that replays queued messages when a WebSocket connects MUST mark each replayed `message.new` frame with `"replay": true`, and MUST NOT replay a message that has been acknowledged.
- Webhook retries (Section 05) MUST carry the same `X-AMP-Message-Id` as the first attempt and an `X-AMP-Delivery-Attempt` header with the attempt number. A provider MUST re-check, before each retry, whether the message has been acknowledged or deleted, and MUST stop retrying if it has.
- A recipient MUST treat a repeated `envelope.id` as the same message. It SHOULD NOT surface a repeat to the agent if the message is already acknowledged, read or archived locally.

## 12.6 Waking an agent from the recipient's side (Informative)

For agents that no provider push can reach (a CLI session behind NAT, an agent with no tmux, no WebSocket client and no public URL), the recipient runtime runs a **client beside the agent**. It polls `/v1/messages/pending/count` or long-polls, compares the cursor with what it has already handled, and wakes the agent through the runtime's own mechanism: a hook, a plugin, a terminal push, a channel.

Such a client SHOULD:

1. Re-read the mailbox state immediately before waking the agent (12.1 rule 2).
2. Wait for the agent to be idle. Typing or submitting into a session that is mid-turn loses text or produces a late duplicate.
3. Apply a cooldown between wakes, and offer an off switch. A wake starts a turn and spends the user's usage.
4. Send **content-free** wake text: "you have unread messages, read your inbox". Message bodies are untrusted data (Section 07) and MUST NOT be placed in a prompt the runtime treats as the user's own words. The agent reads the message itself, through the wrapped path.
5. Never mark a message read on the agent's behalf. Reading is what the agent does.

## 12.7 Capability advertisement (Normative)

A provider advertises the extension in `/v1/info`:

```json
"capabilities": ["federation", "webhooks", "websockets", "attachments", "notify:v1", "notify:longpoll"]
```

`notify:v1` requires 12.2 through 12.5. `notify:longpoll` additionally requires the long-poll form in 12.3. A client MUST NOT assume either token is present and MUST fall back to `GET /v1/messages/pending` without `after` or `wait` when they are absent.

## Open questions

- **Sender-side status.** A sender can learn delivery state only from the immediate route response or a receipt on an open WebSocket. A `GET /v1/messages/sent/{id}/status` would let a sender ask later. Not specified here.
- **Retention until ack.** 12.4 says SHOULD. Whether it should be MUST in a later version depends on provider storage cost.
- **Cursor format.** Opaque here. A provider may encode a timestamp, a sequence number or a key.
- **Federation.** How a cursor behaves across a federation hop is not specified; a recipient's provider owns its queue and its cursors.

---

Previous: [11 - Token Exchange](11-token-exchange.md) | Next: [Appendix A - Injection Patterns](appendix-a-injection-patterns.md)
