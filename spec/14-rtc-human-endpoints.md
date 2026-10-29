# 14 — Human Endpoints and Bidirectional Calls (AMP-RTC)

**Status:** Draft companion proposal to [13 — Real-Time Sessions](13-realtime-sessions.md).

## Goal

A human using a registered AI Maestro mobile app can **call an agent**, and an agent can **call that human**. Neither direction requires a telephone number, PSTN, Twilio, WhatsApp or a special AI Maestro protocol fork. A client installation is an implementation choice, not a requirement of AMP itself.

## Principals and devices

- **Agent**: existing AMP agent identity and address.
- **Human principal**: a distinct authenticated user identity governed by a provider; not an agent masquerading as a person. The exact interoperable identifier format remains an open question.
- **Device**: an enrolled client endpoint bound to a human principal with a revocable public key and scoped push registration.
- **Call coordinator**: the recipient provider's authoritative call-state owner.

A human principal MAY have multiple devices. Devices MUST NOT be treated as independent humans. An app installation is not automatically authorized to answer a call.

## Both initiation directions

**Human → agent**: the authenticated mobile client asks its provider to create an `rtc:invite` attributed to the human principal and addressed to an AMP agent. The provider checks authorization, recipient capabilities and policy before delivery. The agent MUST explicitly accept or reject; delivery or agent wake does not imply acceptance.

**Agent → human**: the agent sends an `rtc:invite` to the human's provider-resolved RTC endpoint. The provider checks the human's allowlist, do-not-disturb and notification policy, then alerts enrolled devices. A user accepts or declines using the mobile client. The provider signs/attests the decision without falsely attributing a human signature to an agent.

## Endpoint and device routing

Providers MUST distinguish **principal identity** from **device installation**. A callee's provider chooses eligible devices, handles push, and atomically arbitrates the first authorized acceptance. The caller never receives APNs tokens. Rejected, expired or canceled calls MUST cease ringing on all devices.

Agent recipients may be busy, unavailable, or require explicit wake; the provider MUST report an accurate outcome rather than synthesizing a connected state.

## Call permissions

- Human-initiated calls to agents require the user's authentication and the agent's call policy.
- Agent-initiated calls to humans require an explicit permission grant or allowlist, with revocation and quiet hours.
- Each device requires enrollment, revocation and short-lived call-scoped credentials.
- Caller display names and avatars are hints; identity verification is derived from AMP trust and the provider's human auth.
- Never equate `rtc:accept` with an established media path; `rtc:connected` follows verified transport establishment.
- Consent for recording and transcription is separate from consent to answer.

## Capability negotiation

Advertise `rtc:v1` as an **optional** capability. An agent or human endpoint may advertise incoming/outgoing audio separately. No shared vendor media service is implied. A client without `rtc:v1` still sends/receives ordinary AMP messages.

## Mobile implementation note

An iOS app can use PushKit + CallKit to present a native incoming call on a locked device, subject to Apple's rules. This is an implementation profile, not a mandatory AMP transport.

## Acceptance criteria

1. Human calls agent; agent accepts, connects, and ends.
2. Agent calls human; registered iPhone rings; human accepts, connects, and ends.
3. Either side can reject or cancel; neither side sees a false connection.
4. Multiple devices ring; only one wins acceptance.
5. Revoked device and blocked caller cannot join.
6. Human and agent endpoints work across compatible providers without telephone numbers.

## Open questions

Human identifier format, provider trust assertions, device attestation, push policy, media negotiation profile, and cross-provider call-coordinator semantics must be resolved before this draft becomes normative.
