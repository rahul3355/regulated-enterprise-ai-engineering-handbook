# Drift Detection and Remediation

## Overview

Drift occurs when actual infrastructure differs from the desired state defined in code. This guide covers detecting and remediating drift for Terraform, Kubernetes, and configuration management.

## Terraform Drift Detection

```bash
# Manual drift check
terraform plan -detailed-exitcode
# Exit code 0: No drift
# Exit code 1: Error
# Exit code 2: Drift detected

# Automated drift detection (CI scheduled)
# .github/workflows/terraform-drift.yml
name: Terraform Drift Detection
on:
  schedule:
    - cron: '0 6 * * *'  # Daily
  workflow_dispatch:

jobs:
  drift-check:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: hashicorp/setup-terraform@v3
      - run: terraform init
      - run: terraform plan -detailed-exitcode
        id: plan
        continue-on-error: true
      
      - name: Alert on drift
        if: steps.plan.outcome == 'failure'
        run: |
          echo "DRIFT DETECTED!"
          terraform plan -no-color > drift-report.txt
          # Send to Slack
          curl -X POST $SLACK_WEBHOOK -d "{\"text\": \"Terraform drift detected\"}"
```

## Kubernetes Drift Detection

```yaml
# Drift detection with ArgoCD
apiVersion: argoproj.io/v1alpha1
kind: AppProject
metadata:
  name: banking-genai
spec:
  # Auto-sync on drift
  syncWindows:
    - kind: allow
      schedule: '0 * * * *'  # Every hour
      duration: 5m
      applications: ['*']
      manualSync: true

# Or manual drift check
argocd app diff genai-api
argocd app sync genai-api --dry-run
```

## Remediation Strategies

```
1. Automatic Remediation:
   - ArgoCD auto-sync for Kubernetes
   - Scheduled terraform apply (with caution)
   - Configuration management (Ansible) re-run

2. Manual Remediation:
   - Review drift report
   - Determine cause (manual change, bug, emergency fix)
   - Update code to match desired state OR
   - Update desired state to match reality
   - Apply and verify

3. Prevention:
   - Restrict manual changes (RBAC)
   - Audit all manual changes
   - Regular drift checks scheduled
   - Alert on drift detection
```

## Cross-References

- **Terraform**: See [terraform.md](terraform.md) for state management
- **IaC**: See [infrastructure-as-code.md](infrastructure-as-code.md) for principles

## Interview Questions

1. **What is infrastructure drift? How does it happen?**
2. **How do you detect drift in Terraform-managed infrastructure?**
3. **Should drift remediation be automated or manual?**
4. **How do you prevent drift in Kubernetes clusters?**
5. **What causes drift between CI/CD runs?**
6. **How do you handle emergency manual changes without causing drift?**

## Checklist: Drift Detection

- [ ] Drift detection scheduled regularly (daily/weekly)
- [ ] Alerts configured for drift
- [ ] Drift report reviewed by team
- [ ] Root cause identified for each drift
- [ ] Remediation procedure documented
- [ ] Manual changes restricted and audited
- [ ] Code updated to match necessary manual changes
- [ ] Drift metrics tracked over time
