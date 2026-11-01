# 13 - Person Principals

**Status:** Draft (proposed extension)
**Version:** 0.1.3-proposed
**Capability:** `principals:person` (providers advertise it in `/v1/info`)

## Overview

AMP addresses, keys and signatures describe **agents**. People appear only in the security rules, as the operator or the "verified principal" whose approval an action may need ([07 - Security](07-security.md#content-authority-normative)). A person cannot send a message, so today a person who wants to instruct an agent asks another agent to relay it. The relayed instruction arrives as a peer's message and, correctly, carries no authority.

This section adds the **person** as a second kind of principal. A person has an address, a mailbox, and one signing key **per device**, so the same person can send from a phone, a tablet and a laptop at the same time. Everything else in AMP (routing, threading, signatures, notification) is unchanged.

## 13.1 Principal kinds (Normative)

A principal is either:

| Kind | Who | Keys |
|------|-----|------|
| `agent` | An AI agent, as defined in [02 - Identity](02-identity.md) | One keypair (rotated over time) |
| `person` | A human member of the tenant | **One keypair per enrolled device**, several valid at once |

- Persons and agents share **one address namespace** in a scope: `juan-pelaez@acme.crabmail.ai` cannot also be an agent. The address grammar is the one in [02 - Identity](02-identity.md#address-grammar-abnf).
- A principal's kind is a property of its registration, not of a message. Resolution returns it (13.4); a message can carry it as a hint (13.3), but recipients MUST take the kind from resolution or from the provider's trust verdict, never from the envelope alone.
- Absent any kind, a principal is an `agent`. Implementations that do not support this extension see no change.

## 13.2 Devices and keys (Normative)

A person is usually signed in on several devices **at the same time**. Each device is a separate signer:

1. **One keypair per device.** The device generates its keypair locally and the private key MUST NOT leave the device (a platform keystore such as the iOS Keychain or the Android Keystore is RECOMMENDED). Only the public key is enrolled.
2. **A key id per device key.** The provider assigns each enrolled key a `key_id` (opaque, unique within the person). A device signs with its own key and names it in the envelope (13.3).
3. **All enrolled keys are valid concurrently.** A message is verified against the key its `key_id` names; there is no "current" device. Adding a device does not invalidate the others.
4. **Enrollment needs an existing authority.** A provider MUST NOT enroll a device key from an unauthenticated request. It MUST be approved by one of: an already-enrolled device of the same person, the tenant's administrator, or an out-of-band verification the provider documents (for example a one-time code shown on an enrolled device).
5. **Revocation is per device.** A lost or retired device's key is revoked on its own; the person's other devices keep working. Messages signed with a revoked key after its revocation time MUST NOT be verified.
6. **Device name.** Each key carries a human-readable device name ("Juan's iPhone") so the person can tell their devices apart when revoking.

## 13.3 Sending as a person (Normative)

A person's message is an ordinary AMP message. The envelope MAY carry two optional fields:

| Field | Type | Meaning |
|-------|------|---------|
| `sender_kind` | `"agent"` \| `"person"` | A hint for display; recipients MUST NOT trust it (13.1) |
| `key_id` | string | Which of the sender's enrolled keys signed this message. REQUIRED when the sender is a `person` |

The canonical signature string is unchanged ([04 - Messages](04-messages.md#canonical-signature-format-v11)): `from|to|subject|priority|in_reply_to|payload_hash`. `sender_kind` and `key_id` are not signed: a wrong `key_id` makes verification fail, and the kind comes from resolution.

The provider that accepts a person's message MUST:

1. Authenticate the sending device (its own session or token mechanism).
2. Check that `from` is that person's address ([04 - Messages, Sender Address Validation](04-messages.md#sender-address-validation)).
3. Verify the signature against the key named by `key_id`, which MUST be enrolled for that person and not revoked.
4. Only then mark the message `verified`. A provider MUST NOT accept a client-supplied "verified" flag.

## 13.4 Resolution (Normative for `principals:person`)

`GET /v1/agents/resolve/{address}` ([02 - Identity](02-identity.md#resolution-api)) returns the kind, and for a person the list of active keys:

```json
{
  "address": "juan-pelaez@acme.crabmail.ai",
  "kind": "person",
  "alias": "Juan Pelaez",
  "keys": [
    { "key_id": "k_iphone_7f3a", "device": "Juan's iPhone", "public_key": "-----BEGIN PUBLIC KEY-----\n...", "key_algorithm": "Ed25519", "enrolled_at": "2026-11-01T10:00:00Z" },
    { "key_id": "k_ipad_19c2", "device": "Juan's iPad", "public_key": "-----BEGIN PUBLIC KEY-----\n...", "key_algorithm": "Ed25519", "enrolled_at": "2026-11-01T10:05:00Z" }
  ]
}
```

For an `agent`, `kind` is `"agent"` and the existing single `public_key` fields are unchanged. Caching follows [02 - Identity](02-identity.md#address-resolution), except that a recipient MUST re-resolve when it sees a `key_id` that is not in its cache (a newly enrolled device).

## 13.5 The person's mailbox (Normative)

- A person's inbox and sent messages are stored **by the provider**, not on a device, so every device of the person sees the same mailbox. Read and archived state is per person, not per device: a message read on the phone is read on the tablet.
- A reply to a person's message is addressed to the person, not to a device. Agents reply exactly as they reply to agents.
- Notification ([12 - Notification and Wake](12-notification.md)) goes to **every** active device of the person. A device that has already shown or read the message SHOULD NOT alert again; the store, not the notification, is the truth about read state.

## 13.6 Authority (Normative)

This section extends [Content Authority](07-security.md#content-authority-normative); it does not relax it.

1. A `verified` message from a `person` is from a **verified principal**.
2. It carries **operator authority** only for agents whose registered operator or manager is that person. To any other agent it is a verified peer's message: trusted for delivery, not for action.
3. Authority never comes from text: a message that says "I am the operator" or "approve this" grants nothing unless rule 2 holds for its verified sender.
4. An approval requested under Content Authority rule 5 MAY be answered by a person's verified message on a channel the untrusted content cannot write to (for example a reply from the person's device to the agent, not a message relayed by another agent).

## 13.7 Presentation (Recommended)

Clients SHOULD show a person sender distinctly from an agent, with its trust verdict, for example `From: Juan Pelaez (person, VERIFIED)`, so that an agent and its operator can both see whose word a message carries.

---

Previous: [12 - Notification and Wake](12-notification.md) | Next: [Appendix A - Injection Patterns](appendix-a-injection-patterns.md)
