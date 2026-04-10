# Secure Deployment Patterns in Kubernetes

## Overview

Security in Kubernetes requires defense in depth: secure images, least-privilege access, network isolation, and runtime protection. This guide covers security patterns for banking GenAI deployments.

## Pod Security Standards

```yaml
# Enforce restricted pod security standard
apiVersion: v1
kind: Namespace
metadata:
  name: banking-genai
  labels:
    pod-security.kubernetes.io/enforce: restricted
    pod-security.kubernetes.io/audit: restricted
    pod-security.kubernetes.io/warn: restricted
---
# Secure pod template
apiVersion: apps/v1
kind: Deployment
metadata:
  name: genai-api
spec:
  template:
    spec:
      securityContext:
        runAsNonRoot: true
        runAsUser: 1000
        runAsGroup: 3000
        fsGroup: 2000
        seccompProfile:
          type: RuntimeDefault
      containers:
        - name: api
          image: quay.io/banking/genai-api@sha256:abc123...  # SHA digest
          securityContext:
            allowPrivilegeEscalation: false
            readOnlyRootFilesystem: true
            capabilities:
              drop:
                - ALL
          volumeMounts:
            - name: tmp
              mountPath: /tmp
      volumes:
        - name: tmp
          emptyDir:
            medium: Memory
```

## Image Security

```yaml
# ImagePullPolicy and SHA digests
containers:
  - name: api
    image: quay.io/banking/genai-api@sha256:abc123def456...
    imagePullPolicy: IfNotPresent

# Image scanning in CI/CD
# Trivy scan before deployment
# OPA/Gatekeeper policies enforce scanning
```

## Supply Chain Security

```yaml
# OPA/Gatekeeper: Require image from approved registry
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: K8sAllowedRepos
metadata:
  name: restrict-image-repos
spec:
  match:
    kinds:
      - apiGroups: [""]
        kinds: ["Pod"]
  parameters:
    repos:
      - "quay.io/banking/"
      - "registry.access.redhat.com/"
```

## Cross-References

- **RBAC**: See [rbac.md](rbac.md) for access control
- **Network Policies**: See [network-policies.md](network-policies.md) for network security
- **Secrets**: See [secrets.md](secrets.md) for credential management

## Interview Questions

1. **What are Kubernetes Pod Security Standards? What does 'restricted' enforce?**
2. **How do you ensure only approved container images run in your cluster?**
3. **Why is readOnlyRootFilesystem important for security?**
4. **How do you prevent privilege escalation in Kubernetes pods?**
5. **What is supply chain security in Kubernetes?**
6. **How do you scan container images for vulnerabilities?**

## Checklist: Secure Deployments

- [ ] Pod Security Standards enforced (restricted)
- [ ] Images use SHA digests (not tags)
- [ ] Images scanned for vulnerabilities before deployment
- [ ] readOnlyRootFilesystem enabled
- [ ] All capabilities dropped
- [ ] runAsNonRoot: true
- [ ] Seccomp profile set to RuntimeDefault
- [ ] Network policies default-deny
- [ ] Secrets not in environment variable defaults
- [ ] Service account token automount disabled
- [ ] Admission controllers configured (OPA/Gatekeeper)
- [ ] Audit logging enabled for security events
