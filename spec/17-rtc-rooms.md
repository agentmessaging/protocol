# 17 — Optional RTC Rooms Extension

**Status: Draft.** Companion to RTC v1, human endpoints and media profile.

## Scope
An optional `rtc-room:v1` capability enables multiple human principals and agent principals to join a real-time voice room. Room membership and permissions are distinct from one-to-one `rtc:v1` call state.

## Entities
- `room_id`: opaque, globally unique room reference scoped to an authoritative hosting provider.
- `participant_id`: authenticated human or AMP agent principal.
- `session_id`: revocable, per-device or per-agent runtime join session.
- `invitation_id`: expiring, targeted invitation with inviter and intended role.
- `room_version`: monotonically increasing authoritative membership revision.

## Provisional events
`rtc-room:invite`, `rtc-room:join-request`, `rtc-room:joined`, `rtc-room:left`, `rtc-room:invite-revoked`, `rtc-room:role-changed`, `rtc-room:ended`, `rtc-room:failed`.

Every event MUST bind `room_id`, `event_id`, authenticated actor, timestamp, and relevant participant or invitation. All cross-provider events use normal signed AMP envelopes. Event IDs MUST be replay-safe. A signed invitation does not grant media access without a separate scoped join authorization.

## State and policy
A room is `open`, `closing` or `closed`. A participant is `invited`, `joining`, `joined`, `left` or `removed`. Room host policy determines admission, maximum participants, who can invite, mute or remove participants, and whether guests may invite agents. The room coordinator serializes membership changes and publishes revisioned events.

A human joining a room MUST NOT inherit tool, memory, message, or administrative permissions of any agent. Agent-directed tool actions remain subject to their own authorization policy. Participants MUST see accurate membership and recording indicators.

## Media
A room's media adapter MAY use an SFU; AMP does not mandate a vendor. Room join credentials MUST be short-lived, scoped to room and participant, and never embedded in push payloads or durable signed invitations. Audio track access is revoked on removal and room closure.

## Call escalation
A one-to-one call MAY propose `move-to-room` through a future explicit negotiation. A private call MUST NOT silently add participants. The original call ends or transitions under a documented policy after consent.

## Federation
The authoritative room provider validates remote AMP principals and issues scoped session grants; remote providers MUST NOT forge room membership or independently mint host permissions. Versioned membership events permit convergence after reconnect. Media federation topology remains an open question.

## Interop acceptance
Two humans and two agents join; join/leave/mute and revocation work; remote participant joins through a second provider; late/replayed join rejected; all media ends on room closure. Recording and transcription require separate explicit policy.

## Open wire decisions
Canonical identity encoding, event schemas, role vocabulary, room discovery, federation bridge topology and codec baseline require consensus before normative publication.
