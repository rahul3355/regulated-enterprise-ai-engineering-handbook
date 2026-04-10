# Security Policy

## Supported Versions

This handbook contains security guidance and code examples for educational purposes. While we strive for accuracy, this is not a substitute for professional security consulting.

| Version | Supported |
|---------|-----------|
| Latest `main` branch | Yes |
| Released versions (tags) | Yes |
| Historical versions | No — content may be outdated |

## Reporting a Security Concern

**If you find a security vulnerability in the code examples or guidance within this handbook**, we take it seriously.

### What We Consider a Security Issue

- **Insecure code examples** that could mislead readers into introducing vulnerabilities
- **Outdated security guidance** that no longer reflects best practices
- **Missing security controls** in architecture or design examples
- **Exposed secrets** (API keys, passwords, connection strings) in code samples
- **Incorrect cryptographic recommendations**

### How to Report

**For sensitive or critical security findings:**

1. **Do not file a public issue.**
2. Email the maintainers directly at: **[SECURITY EMAIL — SET THIS BEFORE PUBLISHING]**
3. Include:
   - The file(s) and line(s) affected
   - Description of the vulnerability
   - Potential impact if followed as-is
   - Suggested remediation

We will acknowledge receipt within **48 hours** and aim to publish a fix within **7 days**.

**For non-sensitive findings** (outdated recommendations, missing controls):

- File a public issue using the [Security Issue template](.github/ISSUE_TEMPLATE/security_issue.md)

### Response Process

1. **Triage (within 48 hours):** Assess severity and scope
2. **Investigation (within 7 days):** Research and validate the finding
3. **Remediation (within 14 days):** Publish corrected content
4. **Disclosure:** Acknowledge the reporter (with their permission)

### Responsible Disclosure

We ask researchers to:

- Give us reasonable time to respond before public disclosure
- Avoid accessing, modifying, or destroying others' data
- Avoid disruption of production services (this is a documentation repository)

## Security Review of Content

All code examples and security guidance in this handbook undergo review before merging:

- **Code examples** must follow secure coding practices
- **Security guidance** must reference current standards and frameworks
- **Cryptographic recommendations** must align with current NIST guidance
- **Architecture examples** must include security controls

## Security-Related Content

This handbook covers security extensively. Please note:

- **All security content is educational** — intended to improve security posture
- **Attack descriptions are defensive** — we describe attacks only to explain prevention
- **Code examples are hardened** — all examples include security controls
- **Guidance is current** — we aim to keep security content updated as threats evolve

## Contact

- **Security email:** [SET BEFORE PUBLISHING]
- **General inquiries:** [SUPPORT.md](SUPPORT.md)
- **Community discussions:** [GitHub Discussions](https://github.com/USERNAME/regulated-enterprise-ai-engineering-handbook/discussions)
