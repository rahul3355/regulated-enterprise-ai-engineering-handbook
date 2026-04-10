# Security Scanning in CI/CD: SAST, DAST, and SCA

## Overview

Security scanning identifies vulnerabilities in code, dependencies, and running applications. In banking, security scanning is mandatory for compliance and risk management. This guide covers SAST, DAST, SCA, and secret detection in CI/CD pipelines.

## Scanning Types

| Type | What It Checks | When | Tools |
|------|---------------|------|-------|
| SAST | Source code vulnerabilities | CI (pre-merge) | CodeQL, SonarQube, Semgrep |
| SCA | Third-party dependency vulnerabilities | CI (build) | Trivy, Dependabot, Snyk |
| DAST | Running application vulnerabilities | Staging | OWASP ZAP, Burp Suite |
| Secret Detection | Leaked credentials | CI (pre-merge) | gitleaks, TruffleHog |
| IaC Scanning | Infrastructure misconfiguration | CI (pre-merge) | Checkov, tfsec |
| Container Scanning | Image vulnerabilities | CI (build) | Trivy, Grype |

## SAST with CodeQL

```yaml
# .github/workflows/codeql.yml
name: CodeQL Analysis
on:
  push:
    branches: [main]
  pull_request:
    branches: [main]
  schedule:
    - cron: '0 6 * * 1'  # Weekly

jobs:
  analyze:
    runs-on: ubuntu-latest
    permissions:
      security-events: write
    strategy:
      matrix:
        language: ['python', 'javascript']
    steps:
      - uses: actions/checkout@v4
      - uses: github/codeql-action/init@v3
        with:
          languages: ${{ matrix.language }}
          queries: security-extended
      - uses: github/codeql-action/analyze@v3
        with:
          category: "/language:${{ matrix.language }}"
```

## SCA with Trivy

```yaml
# .github/workflows/dependency-scan.yml
name: Dependency Scan
on:
  pull_request:
    branches: [main]

jobs:
  sca:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Run Trivy filesystem scan
        uses: aquasecurity/trivy-action@master
        with:
          scan-type: 'fs'
          scan-ref: '.'
          format: 'sarif'
          output: 'trivy-results.sarif'
          severity: 'CRITICAL,HIGH'
          exit-code: '1'
      
      - name: Upload results
        uses: github/codeql-action/upload-sarif@v3
        with:
          sarif_file: 'trivy-results.sarif'
```

## Secret Detection

```yaml
# .github/workflows/secret-detection.yml
name: Secret Detection
on:
  pull_request:
    branches: [main]

jobs:
  secrets:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0
      - name: Run gitleaks
        uses: gitleaks/gitleaks-action@v2
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
          GITLEAKS_LICENSE: ${{ secrets.GITLEAKS_LICENSE }}
```

## DAST with OWASP ZAP

```yaml
# .github/workflows/dast.yml
name: DAST Scan
on:
  workflow_dispatch:
  schedule:
    - cron: '0 2 * * *'  # Nightly

jobs:
  dast:
    runs-on: ubuntu-latest
    steps:
      - name: OWASP ZAP Scan
        uses: zaproxy/action-full-scan@v0.8.0
        with:
          target: 'https://genai-api-staging.bank.com'
          rules_file_name: 'zap-rules.tsv'
          cmd_options: '-a'
          allow_issue_writing: true
          fail_action: true
```

## Cross-References

- **Container Scanning**: See [container-scanning.md](container-scanning.md) for image scanning
- **SBOM**: See [sbom-generation.md](sbom-generation.md) for software bill of materials

## Interview Questions

1. **What is the difference between SAST, DAST, and SCA?**
2. **How do you integrate security scanning into CI/CD without slowing down development?**
3. **What should happen when a CRITICAL vulnerability is found?**
4. **How do you handle false positives from security scanners?**
5. **How often should DAST scans run?**
6. **What is secret detection and how do you prevent credential leaks?**

## Checklist: Security Scanning

- [ ] SAST runs on every PR
- [ ] SCA runs on every build
- [ ] DAST runs nightly against staging
- [ ] Secret detection runs on every PR
- [ ] IaC scanning runs on infrastructure changes
- [ ] Container scanning runs on every image build
- [ ] CRITICAL vulnerabilities block merges
- [ ] HIGH vulnerabilities tracked and fixed within SLA
- [ ] Scan results uploaded to security dashboard
- [ ] False positive process documented
