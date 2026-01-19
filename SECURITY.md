# Security Policy

## Table of Contents

- [Reporting a Vulnerability](#reporting-a-vulnerability)
- [Supported Versions](#supported-versions)
- [Security Best Practices](#security-best-practices)
- [Security Advisories](#security-advisories)

---

## Reporting a Vulnerability

The Butler team takes security seriously. We appreciate your efforts to responsibly disclose your findings.

### How to Report

**DO NOT** create a public GitHub issue for security vulnerabilities.

Instead, please report security vulnerabilities via email:

Email: **security@butlerlabs.dev**

### What to Include

Please include as much of the following information as possible:

- Type of issue (e.g., buffer overflow, SQL injection, cross-site scripting, etc.)
- Full paths of source file(s) related to the issue
- Location of the affected source code (tag/branch/commit or direct URL)
- Step-by-step instructions to reproduce the issue
- Proof-of-concept or exploit code (if possible)
- Impact of the issue, including how an attacker might exploit it

### Response Timeline

| Action | Timeline |
|--------|----------|
| Initial acknowledgment | Within 48 hours |
| Initial assessment | Within 1 week |
| Fix development | Depends on severity |
| Public disclosure | Coordinated with reporter |

### What to Expect

1. **Acknowledgment**: We will acknowledge receipt of your report within 48 hours
2. **Assessment**: We will investigate and assess the vulnerability
3. **Updates**: We will keep you informed of our progress
4. **Fix**: We will develop and test a fix
5. **Disclosure**: We will coordinate public disclosure with you

### Safe Harbor

We consider security research conducted in accordance with this policy to be:

- Authorized concerning any applicable anti-hacking laws
- Authorized concerning any relevant anti-circumvention laws
- Exempt from restrictions in our Terms of Service that would interfere with security research

We will not pursue legal action against researchers who:

- Act in good faith
- Avoid privacy violations and data destruction
- Do not exploit vulnerabilities beyond proof-of-concept
- Report vulnerabilities promptly

---

## Supported Versions

| Version | Supported |
|---------|-----------|
| Latest release | Yes |
| Previous minor release | Security fixes only |
| Older versions | No |

We recommend always running the latest version of Butler components.

---

## Security Best Practices

When deploying Butler, follow these security best practices:

### Infrastructure Security

- Run Butler in a private network, not exposed to the internet
- Use network policies to restrict communication between components
- Enable TLS for all communications
- Regularly rotate credentials and certificates

### Authentication & Authorization

- Enable OIDC/SSO for user authentication
- Use strong passwords for local accounts
- Follow the principle of least privilege for RBAC
- Regularly audit team memberships and permissions

### Secrets Management

- Use Kubernetes Secrets or external secret management (e.g., HashiCorp Vault)
- Never commit credentials to Git repositories
- Rotate provider credentials regularly
- Use service accounts with minimal permissions

### Monitoring & Auditing

- Enable Kubernetes audit logging
- Monitor Butler controller logs for anomalies
- Set up alerts for failed authentication attempts
- Regularly review access logs

---

## Security Advisories

Security advisories will be published via:

- GitHub Security Advisories
- Release notes
- Mailing list (if established)

Subscribe to GitHub releases to receive notifications about security updates.
