# 18 — RTC Conformance Fixtures and Harness

**Status: Draft.** This document is a test plan, not a claim that interoperable wire fixtures already exist.

## Fixture format
A conformance fixture should contain `case_id`, capabilities, authenticated principals, signed AMP envelope inputs, expected call/room state transitions, and normalized outcomes. Private signing keys and media credentials must never be committed. Use test-only ephemeral keys.

## Required scenarios
1. Human calls agent; authenticated agent accepts; media connects; either side ends.
2. Agent calls human; registered device receives invitation; human accepts.
3. Invite timeout and caller cancellation.
4. Duplicate signed message and reordered connect before accept.
5. Two devices accept concurrently: exactly one wins atomically.
6. Revoked device, blocked caller and expired join grant denied.
7. Two humans and two agents join a room; participant permissions enforced.
8. Unauthorized guest cannot access agent tools or private memory.
9. Two independent AMP providers exchange signed call and room invitations.
10. Media failure never results in a connected call.
11. Old AMP clients ignore optional RTC capabilities without breaking messaging.
12. Recording and transcription disabled unless consent policy permits them.

## Harness contract
An adapter implements `sendEnvelope`, `receiveEnvelope`, `resolveCapabilities`, `observeState`, `advanceClock` and `reset`. The harness drives deterministic timestamps and event IDs and asserts both provider outcomes. Concurrent-answer tests MUST execute against a real durable atomic coordinator, not only a pure reducer.

## Release gates
Schema validity; signature verification; idempotency; permission checks; provider federation; media failure; mobile wake/cancel; backward compatibility. A green spec-only test does not prove working audio or CallKit.

## Deliverables still needed
Normative JSON Schemas, sample signed envelopes with test keys, reference harness runner and CI integration. These are explicit follow-on implementation items.
