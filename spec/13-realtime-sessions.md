# 13 — Real-Time Sessions (AMP-RTC)

**Status:** Draft proposal  
**Extension:** `rtc:v1` (optional)  
**Base:** AMP 0.1.x

## Purpose and scope

AMP-RTC defines **signed call signaling** between AMP principals. It enables an agent to invite another agent or an explicitly authorized human endpoint to a real-time voice session without requiring a telephone number. AMP handles discovery, authentication, authorization context and call lifecycle; **media flows outside AMP** over an independently negotiated transport such as WebRTC.

This extension MUST NOT require AI Maestro, LiveKit, a specific AI model, or a PSTN/SIP provider. Existing AMP implementations that do not advertise `rtc:v1` remain fully compatible.

## Existing AMP primitives

- Section 04 already supports namespaced custom payload types, `context`, `thread_id`, `in_reply_to`, `expires_at`, and `idempotency_key`.
- Section 02 defines agent addresses and signed Agent Cards with capabilities.
- Section 12 defines optional `notify:v1` mailbox notifications. Notification delivery MUST NOT be confused with an answered call.
- Section 07 signing and replay protections apply unchanged to all signaling messages.

## Capability discovery

An RTC-capable endpoint advertises `rtc:v1` in its signed Agent Card or a resolved endpoint capability response, and MAY declare `media: ["audio"]` and transport adapters. Provider capability advertisement alone is insufficient: **both the caller and the addressed recipient endpoint** must support RTC. How human/device endpoint identity is represented requires a companion proposal; human identities MUST NOT be silently treated as ordinary agent identities.

## Signaling payloads

The following custom `payload.type` values are proposed:

| Type | Direction | Meaning |
|---|---|---|
| `rtc:invite` | Caller → callee | Offer a time-limited session |
| `rtc:ringing` | Callee → caller | One or more eligible devices are alerting |
| `rtc:accept` | Callee → caller | Authorized endpoint accepts the invitation |
| `rtc:reject` | Callee → caller | Invitation declined |
| `rtc:cancel` | Caller → callee | Caller withdraws before acceptance |
| `rtc:connected` | Either → other | Media path established |
| `rtc:end` | Either → other | End an accepted session |
| `rtc:failed` | Either → other | Terminal setup or transport failure |
| `rtc:summary` | Either → other | Optional post-call application message; consent rules apply |

Every RTC message MUST contain `context.call_id` (an unpredictable opaque identifier), `context.rtc_version: "1"`, and `context.event_id` (unique per transition). Replies SHOULD set `envelope.in_reply_to` to the preceding signaling message and retain the invitation's `thread_id`. All messages use the standard AMP signature. `payload.message` remains a required human-readable description under Section 04.

Example invitation **payload** (inside an ordinary signed AMP envelope):

```json
{
  "type": "rtc:invite",
  "message": "Lola requests a voice call",
  "context": {
    "rtc_version": "1",
    "call_id": "call_opaque_random",
    "event_id": "evt_opaque_random",
    "media": ["audio"],
    "transport_options": ["webrtc"],
    "expires_at": "2026-10-28T20:00:30Z",
    "display_name": "Lola"
  }
}
```

The AMP envelope's `expires_at` MUST be no later than the invitation's expiry. All endpoint display names are **untrusted presentation hints**, not identity proof.

## Call state machine

```text
idle → inviting → ringing → accepted → connected → ended
            ↘ rejected / canceled / timed_out / failed
                              accepted → failed / ended
```

- The callee MUST reject an expired invitation, even if a delayed push is delivered.
- Duplicate `event_id` or `envelope.id` MUST NOT repeat transitions.
- An `accept` MUST bind to the invitation's `call_id` and authorized callee identity.
- When multiple devices ring, the recipient's authoritative call coordinator MUST select **one winning acceptance atomically**; losing devices receive cancellation/termination.
- Concurrent `cancel` and `accept` require a deterministic coordinator decision, not last-message-wins ordering across providers.
- A timeout or terminal state MUST release reserved media resources.
- Delivery or `rtc:ringing` does not prove a human saw the invitation; `rtc:accept` does not prove media connectivity.

## Media negotiation and security

AMP carries signaling and **opaque references** to short-lived media-session negotiation. Media session access MUST be scoped to `call_id`, the authorized participants and a short expiry. Long-lived room tokens, reusable credentials, raw media, and STT transcripts MUST NOT be placed in durable AMP signaling messages. An adapter MAY use WebRTC directly, LiveKit, or another implementation with equivalent identity and transport security. Transport-specific offers, ICE candidates, and TURN details need a separate interoperability profile; implementations MUST NOT assume that advertising `webrtc` alone guarantees cross-provider compatibility.

A recipient MUST apply caller allowlists, consent, rate limits, spam prevention, and device-level permissions before alerting a human. A verified AMP agent identity is **not** permission to call a person. Recordings and summaries require explicit consent and applicable retention policy. Push payloads SHOULD contain minimal data and MUST NOT contain media credentials.

## Mobile notification bridge (informative)

An iOS implementation may use APNs VoIP PushKit and CallKit to present a native incoming-call UI on a locked device, subject to Apple's platform requirements. Android implementations may use their platform's push and calling APIs. Push is a **wake hint**, not the call transport, source of truth, or a guarantee of immediate delivery. Device tokens remain private to the recipient's push gateway. An iOS app and push infrastructure are required; AMP alone cannot ring a locked phone.

## Federation and failure modes

A cross-provider call must preserve AMP signature verification and correlation, tolerate delayed/reordered events, handle unreachable devices and rejected capabilities, and expire abandoned invitations. A provider MUST NOT forge an `rtc:accept` on behalf of a human. Media reachability, network changes, permission revocation, and rejoin behavior require explicit implementation tests.

## Interoperability acceptance tests

1. Existing non-RTC AMP client continues to receive ordinary messages unchanged.
2. Two independent RTC endpoints exchange a signed invite, accept, connect, and end sequence.
3. Expired, duplicate, replayed, unauthorized, and out-of-order events are safely rejected or reconciled.
4. Two devices receiving one invite result in at most one accepted media session.
5. A provider-to-provider call works without shared internal state or vendor-specific AMP fields.
6. Mobile push failure does not leave a phantom answered call; cancellation and timeout converge.
7. No PSTN number is required for an app-to-app call.

## Open design questions (review checkpoint)

- How should **human and device identities** relate to agent addresses, AID and Agent Cards?
- Which endpoint/provider is authoritative for multi-device arbitration across federation?
- Should SDP/ICE signaling use ephemeral authenticated channels rather than durable AMP messages?
- What is the minimal transport profile for interoperability beyond capability names?
- Which call metadata can providers retain without exposing private information?

**Checkpoint:** Resolve identity, arbitration and transport-profile questions before declaring `rtc:v1` normative. Keep this document draft until an independent implementation validates interoperability.
