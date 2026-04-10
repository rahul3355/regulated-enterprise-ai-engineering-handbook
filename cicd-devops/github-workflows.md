# GitHub Actions for CI/CD

## Overview

GitHub Actions provides cloud-native CI/CD directly in your repository. This guide covers reusable workflows, self-hosted runners for banking security, and production-grade CI/CD patterns.

## Reusable Workflow

```yaml
# .github/workflows/reusable-build.yml
name: Reusable Build
on:
  workflow_call:
    inputs:
      image_name:
        required: true
        type: string
      dockerfile:
        required: false
        type: string
        default: Dockerfile
    outputs:
      image_digest:
        value: ${{ jobs.build.outputs.image_digest }}

jobs:
  build:
    runs-on: ubuntu-latest
    outputs:
      image_digest: ${{ steps.build.outputs.digest }}
    steps:
      - uses: actions/checkout@v4
      
      - name: Build container image
        uses: docker/build-push-action@v5
        with:
          context: .
          file: ${{ inputs.dockerfile }}
          push: true
          tags: ${{ inputs.image_name }}:${{ github.sha }}
          labels: |
            org.opencontainers.image.source=${{ github.server_url }}/${{ github.repository }}
            org.opencontainers.image.revision=${{ github.sha }}
      
      - name: Generate SBOM
        uses: anchore/sbom-action@v0
        with:
          image: ${{ inputs.image_name }}:${{ github.sha }}
      
      - name: Scan image
        uses: aquasecurity/trivy-action@master
        with:
          image-ref: ${{ inputs.image_name }}:${{ github.sha }}
          format: sarif
          exit-code: '1'
          severity: CRITICAL,HIGH
```

## Main CI Pipeline

```yaml
# .github/workflows/ci.yml
name: CI Pipeline
on:
  pull_request:
    branches: [main]
  push:
    branches: [main]

permissions:
  contents: read
  packages: write
  security-events: write

jobs:
  build:
    uses: ./.github/workflows/reusable-build.yml
    with:
      image_name: quay.io/banking/genai-api
  
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with:
          python-version: '3.11'
      - run: pip install -r requirements.txt
      - run: pytest --junitxml=test-results.xml --cov=src/
      - uses: actions/upload-artifact@v4
        with:
          name: test-results
          path: test-results.xml
  
  security:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Run SAST
        uses: github/codeql-action/analyze@v3
      - name: Run SCA
        uses: aquasecurity/trivy-action@master
        with:
          scan-type: 'fs'
          severity: CRITICAL,HIGH
  
  deploy-staging:
    needs: [build, test, security]
    if: github.ref == 'refs/heads/main'
    uses: ./.github/workflows/reusable-deploy.yml
    with:
      environment: staging
      image_digest: ${{ needs.build.outputs.image_digest }}
```

## Cross-References

- **Branching Strategies**: See [branching-strategies.md](branching-strategies.md) for branching models
- **PR Process**: See [pull-request-process.md](pull-request-process.md) for review requirements

## Interview Questions

1. **How do you create reusable workflows in GitHub Actions?**
2. **What are the security implications of GitHub Actions for banking code?**
3. **How do you use self-hosted runners for compliance?**
4. **How do you pass outputs between jobs in GitHub Actions?**
5. **What permissions should a GitHub Actions workflow have?**
6. **How do you handle secrets in GitHub Actions securely?**

## Checklist: GitHub Actions CI/CD

- [ ] Reusable workflows for common patterns
- [ ] Branch protection rules enforced
- [ ] Self-hosted runners for sensitive operations
- [ ] Secrets stored in GitHub (not in code)
- [ ] Minimum permissions set (principle of least privilege)
- [ ] Artifact retention policy configured
- [ ] Workflow execution logs retained
- [ ] SBOM generated for every build
- [ ] Image scanning before deployment
- [ ] Environment protection rules for production
