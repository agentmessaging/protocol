# Appendix B — Provider Deployment Checklist

**Status:** Informative (non-normative)
**Version:** 0.1.2

This appendix provides a deployment hardening checklist for AMP provider implementations. Items are organized by category with severity ratings. Each item references the normative section that defines the requirement.

> **Note:** This checklist is a summary of security-relevant requirements from the specification. It is not a substitute for reading the normative sections. Providers SHOULD review each referenced section for full implementation details.

## 1. Cryptography

| Severity | Item | Reference |
|----------|------|-----------|
| Critical | Ed25519 is the recommended signing algorithm | [07 - Security](07-security.md#algorithms) |
| Critical | Private key files have permissions `0600` | [07 - Security](07-security.md#key-storage) |
| Critical | Key revocation list is active and checked at route time | [07 - Security](07-security.md#key-revocation) |
| High | Key rotation proof is validated (old key signs new key) | [03 - Registration](03-registration.md#public-key-rotation) |
| High | Revocation entries retained for at least 90 days | [07 - Security](07-security.md#key-revocation) |
| Medium | Identity conflict detection is implemented (TOFU + cache) | [07 - Security](07-security.md#identity-conflict-detection) |

## 2. Transport

| Severity | Item | Reference |
|----------|------|-----------|
| Critical | All API endpoints served over HTTPS (TLS 1.2+) | [07 - Security](07-security.md#transport-security) |
| Critical | No plain HTTP in production | [07 - Security](07-security.md#transport-security) |
| Critical | WebSocket connections use `wss://`, not `ws://` | [07 - Security](07-security.md#transport-security) |
| High | HSTS headers set on all responses | [07 - Security](07-security.md#transport-security) |
| Medium | WebSocket subprotocol `amp.v1` supported | [08 - API](08-api.md#connection) |

## 3. Authentication

| Severity | Item | Reference |
|----------|------|-----------|
| Critical | API keys hashed with bcrypt before storage | [07 - Security](07-security.md#api-key-security) |
| Critical | API keys shown only once at registration | [07 - Security](07-security.md#api-key-security) |
| High | Registration rate-limited (10/min default) | [08 - API](08-api.md#rate-limits) |
| High | Owner authentication enabled for agent registration | [03 - Registration](03-registration.md#owner-authentication-recommended) |
| High | Sender verification: `from` field matches authenticated agent | [07 - Security](07-security.md#sender-verification) |
| Medium | API keys use structured format (`amp_<env>_<type>_<random>`) | [07 - Security](07-security.md#api-key-format) |

## 4. Content Security

| Severity | Item | Reference |
|----------|------|-----------|
| Critical | External message content wrapped in `<external-content>` tags | [07 - Security](07-security.md#content-wrapping-normative) |
| Critical | Injection scanning enabled for incoming messages | [07 - Security](07-security.md#prompt-injection-defense) |
| High | Quarantine active for critical-severity findings | [07 - Security](07-security.md#message-quarantine) |
| High | Default severity-to-verdict mapping implemented | [07 - Security](07-security.md#default-severity-to-verdict-mapping) |
| High | Multi-message window scanning active | [07 - Security](07-security.md#multi-message-window-scanning) |
| Medium | Attachment scanning pipeline operational | [07 - Security](07-security.md#scanning-pipeline) |
| Medium | Credential redaction in audit output | [07 - Security](07-security.md#abuse-prevention) |

## 5. Network

| Severity | Item | Reference |
|----------|------|-----------|
| Critical | Webhook SSRF validation on registration AND delivery | [05 - Routing](05-routing.md#ssrf-prevention) |
| Critical | Private/loopback/link-local/metadata IPs blocked for webhooks | [05 - Routing](05-routing.md#ssrf-prevention) |
| High | Webhook redirect limit: 2 hops max | [05 - Routing](05-routing.md#redirect-handling) |
| High | Webhook connection timeout: 5s, response timeout: 10s | [05 - Routing](05-routing.md#timeouts) |
| High | No HTTPS-to-HTTP downgrades on webhook redirects | [05 - Routing](05-routing.md#redirect-handling) |
| High | Request body size enforcement: 1 MB limit | [08 - API](08-api.md#send-message-route) |
| Medium | DNS rebinding protection (validate resolved IPs) | [05 - Routing](05-routing.md#ssrf-prevention) |
| Medium | Alternative IP encoding rejection (hex, octal, decimal) | [05 - Routing](05-routing.md#ssrf-prevention) |

## 6. Access Control

| Severity | Item | Reference |
|----------|------|-----------|
| High | Communication policy enforcement at route time | [03 - Registration](03-registration.md#agent-communication-policy) |
| High | Suspended agent checks on all message paths | [07 - Security](07-security.md#behavior-when-suspended) |
| Medium | Default-deny communication policy available (`restricted` mode) | [03 - Registration](03-registration.md#agent-communication-policy) |
| Medium | Wildcard ACL patterns audited for overly broad access | [03 - Registration](03-registration.md#wildcard-matching) |
| Low | Tenant access controls configured (open/invite/verified/admin) | [03 - Registration](03-registration.md#tenant-access) |

## 7. Monitoring

| Severity | Item | Reference |
|----------|------|-----------|
| High | Risk scoring active with rolling 24-hour window | [07 - Security](07-security.md#risk-scoring) |
| High | Auto-escalation thresholds configured | [07 - Security](07-security.md#thresholds) |
| Medium | Audit trail enabled for quarantine and suspension actions | [07 - Security](07-security.md#message-quarantine) |
| Medium | Retention period >= 90 days for revocation and audit entries | [07 - Security](07-security.md#key-revocation) |
| Medium | Webhook notifications configured for risk level changes | [07 - Security](07-security.md#risk-scoring) |
| Low | Quarantine expiration logged for audit | [07 - Security](07-security.md#ttl-and-expiration) |

## 8. Rate Limiting

| Severity | Item | Reference |
|----------|------|-----------|
| High | Per-agent rate limits enforced (60 msgs/min default) | [07 - Security](07-security.md#per-agent-limits) |
| High | Per-provider federation limits enforced (1000 msgs/min) | [07 - Security](07-security.md#per-provider-limits-federation) |
| High | Request body size enforcement (1 MB HTTP limit) | [08 - API](08-api.md#send-message-route) |
| Medium | Replay protection with 24-hour ID tracking | [07 - Security](07-security.md#replay-protection) |
| Medium | Future timestamp rejection (60-second tolerance) | [07 - Security](07-security.md#replay-protection) |
| Medium | Admin endpoint rate limits (quarantine, suspension, risk) | [08 - API](08-api.md#rate-limits) |

---

Previous: [Appendix A — Prompt Injection Patterns](appendix-a-injection-patterns.md) | Back to: [01 - Overview](01-overview.md)
