# Container Scanning: Image Security in CI/CD

## Overview

Container scanning identifies vulnerabilities in container images before they reach production. This guide covers Trivy, policy enforcement, and container security best practices for banking GenAI platforms.

## Trivy Scanning

```yaml
# CI/CD container scanning with Trivy
name: Container Security Scan
on:
  pull_request:
    paths:
      - 'Dockerfile'
      - 'requirements.txt'
      - 'package.json'

jobs:
  scan:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - name: Build image
        run: docker build -t genai-api:${{ github.sha }} .
      
      - name: Run Trivy scan
        uses: aquasecurity/trivy-action@master
        with:
          image-ref: 'genai-api:${{ github.sha }}'
          format: 'table'
          exit-code: '1'
          severity: 'CRITICAL,HIGH'
          ignore-unfixed: true
      
      - name: Generate SARIF report
        uses: aquasecurity/trivy-action@master
        with:
          image-ref: 'genai-api:${{ github.sha }}'
          format: 'sarif'
          output: 'trivy-results.sarif'
      
      - name: Upload to GitHub Security
        uses: github/codeql-action/upload-sarif@v3
        with:
          sarif_file: 'trivy-results.sarif'
```

## Policy Enforcement

```yaml
# OPA policy: Block images with CRITICAL vulnerabilities
# policy.rego
package container_security

deny[msg] {
  input.vulnerabilities[_].severity == "CRITICAL"
  msg := sprintf("CRITICAL vulnerability found: %s in %s", 
    [input.vulnerabilities[_].vulnerabilityID, 
     input.vulnerabilities[_].pkgName])
}

deny[msg] {
  count([v | v = input.vulnerabilities[_]; v.severity == "HIGH"]) > 5
  msg := "More than 5 HIGH vulnerabilities found"
}

deny[msg] {
  input.image.user == "root"
  msg := "Container runs as root user"
}
```

## Cross-References

- **Security Scanning**: See [security-scanning.md](security-scanning.md) for SAST/DAST/SCA
- **SBOM**: See [sbom-generation.md](sbom-generation.md) for bill of materials

## Interview Questions

1. **How do you scan container images for vulnerabilities?**
2. **What severity levels should block a deployment?**
3. **How do you handle false positives from container scanners?**
4. **What is the difference between scanning at build time vs runtime?**
5. **How do you enforce container security policies?**
6. **What base image practices reduce container vulnerabilities?**

## Checklist: Container Scanning

- [ ] Trivy or equivalent scanner integrated
- [ ] CRITICAL vulnerabilities block merges
- [ ] HIGH vulnerabilities tracked with SLA
- [ ] Scan results uploaded to security dashboard
- [ ] Base images regularly updated
- [ ] Minimal base images used (distroless, Alpine)
- [ ] Non-root user in containers
- [ ] Image signing and verification
- [ ] Vulnerability exceptions process defined
- [ ] Runtime scanning enabled
