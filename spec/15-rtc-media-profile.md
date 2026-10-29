# 15 — RTC Media Interoperability Profile

**Status:** Draft. Companion to [13 — Real-Time Sessions](13-realtime-sessions.md).

## Separation of responsibilities

AMP delivers authenticated and signed **call control** events. A media adapter transports low-latency audio and does not alter AMP message semantics. A provider MUST NOT require LiveKit, Twilio, AI Maestro or any proprietary SDK to understand the AMP-RTC extension.

## Proposed minimum profile

- Capability: `rtc:v1`; media: `audio`; media profile: `webrtc-audio-v1` (provisional).
- The recipient accepts a call only after user/agent authorization, then participants negotiate a media session.
- SDP offers/answers and ICE candidates are exchanged through a short-lived authenticated signaling channel, or a separately specified encrypted AMP signaling profile. Do not place reusable TURN or room credentials in durable AMP messages.
- WebRTC uses DTLS-SRTP and ICE; implementations MUST handle network changes, NAT traversal, ICE failure, microphone permission failure and teardown.
- Both endpoints MUST confirm media establishment before emitting `rtc:connected`.
- Tokens MUST be bound to the `call_id`, participants, expiry and role, with server-side revocation on termination.
- The media transport MUST NOT grant agent-tool or inbox permissions.

## Adapter contract (informative)

```text
createSession(callId, participants) -> scopedSessionReference
joinSession(scopedReference, participant) -> local/remote audio tracks
onConnectionStateChanged(callId, state)
leaveSession(callId, participant)
closeSession(callId)
```

An AI agent voice adapter may bridge remote audio to STT, an LLM and TTS, or to a streaming speech-to-speech model. The AMP protocol does not prescribe model choice, voice provider, transcript storage or agent execution engine.

## Security and privacy

Recording and transcription are OFF unless independently authorized. Push payloads MUST NOT carry room credentials or SDP. An accepted call is not an authorization to execute arbitrary agent tools. Media access MUST be revoked on hangup, call expiration or device revocation.

## Interoperability checkpoints

1. Browser-to-browser audio over independent providers.
2. Human mobile client to remote agent with bidirectional audio.
3. Agent-initiated mobile call on a locked device.
4. Media negotiation failure surfaces `rtc:failed`, never `rtc:connected`.
5. Call termination stops audio and revokes credentials.
6. Independent implementations can negotiate without sharing a vendor-specific SDK.

## Open issues

The precise authenticated negotiation endpoint, codec baseline, TURN credential exchange, reconnect rules and conformance fixtures are deliberately unresolved. This proposal is not yet a complete wire-level standard.
