# Secrets Handling in CI/CD

## Overview

Managing secrets in CI/CD pipelines requires balancing security with automation. This guide covers secret storage, Vault integration, and best practices for banking environments.

## Secret Storage Options

| Method | Security | Automation | Best For |
|--------|----------|------------|----------|
| GitHub Secrets | Medium | High | CI/CD variables |
| AWS Secrets Manager | High | High | AWS infrastructure |
| HashiCorp Vault | Highest | High | Enterprise secrets |
| SOPS + Git | High | Medium | Encrypted files in git |
| Kubernetes Secrets | Medium | High | In-cluster secrets |

## GitHub Actions Secrets

```yaml
# Using secrets in workflows
jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Deploy
        env:
          AWS_ACCESS_KEY_ID: ${{ secrets.AWS_ACCESS_KEY_ID }}
          AWS_SECRET_ACCESS_KEY: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          DATABASE_URL: ${{ secrets.DATABASE_URL }}
        run: ./scripts/deploy.sh
```

## Vault Integration

```yaml
# Authenticate with Vault in CI/CD
- name: Authenticate with Vault
  uses: hashicorp/vault-action@v2
  with:
    url: https://vault.bank.com:8200
    method: kubernetes
    role: genai-ci-role
    secrets: |
      secret/data/banking/genai/database url | DATABASE_URL ;
      secret/data/banking/genai/openai api_key | OPENAI_API_KEY ;
```

## Cross-References

- **Secrets (K8s)**: See [secrets.md](../kubernetes-openshift/secrets.md) for Kubernetes secrets
- **Security Scanning**: See [security-scanning.md](security-scanning.md) for secret detection

## Interview Questions

1. **How do you store and access secrets in CI/CD pipelines?**
2. **Compare GitHub Secrets, Vault, and AWS Secrets Manager.**
3. **How do you rotate secrets without downtime?**
4. **How do you prevent secrets from leaking in CI/CD logs?**
5. **What is SOPS? How does it work with GitOps?**
6. **How do you audit secret access in CI/CD?**

## Checklist: Secrets in CI/CD

- [ ] No secrets in source code
- [ ] Secrets stored in dedicated secret manager
- [ ] Secret access logged and audited
- [ ] Secret rotation automated
- [ ] CI/CD pipeline uses short-lived credentials
- [ ] Secrets masked in pipeline logs
- [ ] Secret access scoped to minimum required
- [ ] Emergency secret access procedure documented
- [ ] Secret leakage detection enabled
- [ ] Vault or equivalent integrated for production
