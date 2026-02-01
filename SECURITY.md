# Security Policy

## Reporting a Vulnerability

We take the security of the Agent Messaging Protocol seriously. If you discover a security vulnerability, please report it responsibly.

### How to Report

**Email:** security@agentmessaging.org

**GitHub:** [Open a security advisory](https://github.com/agentmessaging/protocol/security/advisories/new)

### What to Include

- Description of the vulnerability
- Steps to reproduce
- Potential impact
- Suggested fix (if any)

### What to Expect

| Timeframe | Action |
|-----------|--------|
| 24 hours | Acknowledgment of your report |
| 72 hours | Initial assessment and severity classification |
| 7 days | Status update on fix timeline |
| 90 days | Public disclosure (coordinated with you) |

### Scope

This policy covers:
- The Agent Messaging Protocol specification
- Reference implementations maintained by this organization
- The agentmessaging.org website

Out of scope:
- Third-party implementations
- Provider-specific issues (contact the provider directly)

## Security Considerations

The protocol includes several security mechanisms:

1. **Cryptographic Signatures** - All messages are signed with Ed25519 keys
2. **Key Verification** - Public keys are registered with providers and verified on receipt
3. **Transport Security** - TLS 1.3+ required for all provider communication
4. **Rate Limiting** - Providers must implement rate limiting to prevent abuse

For detailed security architecture, see [spec/07-security.md](./spec/07-security.md).

## Supported Versions

| Version | Supported |
|---------|-----------|
| 0.1.x (draft) | :white_check_mark: |

## Recognition

We appreciate responsible disclosure. With your permission, we'll acknowledge your contribution in our release notes and security advisories.

## Contact

- Security issues: security@agentmessaging.org
- General questions: hello@agentmessaging.org
- Protocol discussions: [GitHub Discussions](https://github.com/agentmessaging/protocol/discussions)
