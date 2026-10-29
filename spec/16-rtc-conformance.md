# 16 — AMP-RTC Conformance Matrix

**Status:** Draft test plan for optional `rtc:v1`.

## Core scenarios

| ID | Scenario | Expected |
|---|---|---|
| RTC-01 | Human invites agent | Agent may accept/reject; no automatic connection |
| RTC-02 | Agent invites human | Authorized device can ring and accept |
| RTC-03 | Caller cancels before answer | All devices stop ringing |
| RTC-04 | Invite expires | No late answer accepted |
| RTC-05 | Duplicate signed event | Exactly one state transition |
| RTC-06 | Reordered connect before accept | Rejected without connection |
| RTC-07 | Two devices answer | Exactly one wins atomically |
| RTC-08 | Blocked caller | No device alerted |
| RTC-09 | Revoked device | Cannot accept or join media |
| RTC-10 | Non-RTC peer | Existing AMP messaging unaffected |
| RTC-11 | Cross-provider invitation | Signatures, call_id and state preserved |
| RTC-12 | Media negotiation fails | Terminal failed state, no phantom connection |
| RTC-13 | Either participant hangs up | Media and credentials revoked |
| RTC-14 | Push arrives late | No stale CallKit incoming call |
| RTC-15 | Reconnect after network loss | Explicit policy; no duplicate sessions |
| RTC-16 | Recording not authorized | No recording or persisted transcript |

## Required evidence

Each scenario should provide independent sender and recipient logs, AMP envelope IDs, call ID, ordered event IDs, transition outcomes and redacted media adapter events. Never log push tokens, SDP credentials, raw audio or secret keys.

## Release gates

1. **Protocol gate:** two independent endpoints exchange signed bidirectional signaling.
2. **Coordinator gate:** durable, atomic first-answer-wins and timeout behavior.
3. **Media gate:** verified two-way audio and cleanup.
4. **Mobile gate:** native locked-screen incoming call and outgoing call.
5. **Security gate:** revocation, rate limiting, consent and replay tests.

The implementation MUST remain feature-flagged until all gates relevant to its advertised capabilities pass.
