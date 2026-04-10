# OpenShift Security Context Constraints (SCC)

## Overview

SCCs control what permissions pods can have in OpenShift, providing more granular security control than standard Kubernetes PodSecurityStandards. This is critical for banking compliance.

## SCC Fundamentals

```yaml
# Restrictive SCC for GenAI applications
apiVersion: security.openshift.io/v1
kind: SecurityContextConstraints
metadata:
  name: genai-restricted
allowPrivilegedContainer: false
runAsUser:
  type: MustRunAsNonRoot
seLinuxContext:
  type: MustRunAs
fsGroup:
  type: MustRunAs
  ranges:
    - min: 1000
      max: 2000
supplementalGroups:
  type: MustRunAs
  ranges:
    - min: 1000
      max: 2000
volumes:
  - configMap
  - downwardAPI
  - emptyDir
  - persistentVolumeClaim
  - projected
  - secret
allowedCapabilities: null
requiredDropCapabilities:
  - ALL
defaultAddCapabilities: null
readOnlyRootFilesystem: true
allowHostDirVolumePlugin: false
allowHostIPC: false
allowHostNetwork: false
allowHostPID: false
allowHostPorts: false
allowPrivilegeEscalation: false
```

## Common SCCs

| SCC | Privileged | RunAs | Capabilities | Use Case |
|-----|-----------|-------|-------------|----------|
| privileged | Yes | Any | Any | System daemons |
| anyuid | No | Any | Default | Legacy apps |
| nonroot | No | Non-root | Default | Modern apps |
| restricted | No | Non-root | Drop all | Default (most secure) |

```bash
# View SCCs
oc get scc

# View which SCC applies to a pod
oc describe pod <pod-name> | grep SCC

# Grant SCC to service account
oc adm policy add-scc-to-user restricted -z genai-api-sa -n banking-genai

# Check SCC priority
oc get scc restricted -o yaml | grep priority
```

## Cross-References

- **RBAC**: See [rbac.md](rbac.md) for access control
- **Secure Deployments**: See [secure-deployment-patterns.md](secure-deployment-patterns.md)

## Interview Questions

1. **What are SCCs in OpenShift? How do they differ from Kubernetes PodSecurityStandards?**
2. **What is the most restrictive SCC? What does it prevent?**
3. **How do you grant a specific SCC to a service account?**
4. **Why is readOnlyRootFilesystem important for security?**
5. **What capabilities should be dropped in a banking application?**
6. **How do you debug SCC-related pod startup failures?**

## Checklist: SCC Configuration

- [ ] Default SCC is 'restricted' for all namespaces
- [ ] No application pods use 'privileged' or 'anyuid' SCC
- [ ] readOnlyRootFilesystem enabled where possible
- [ ] All capabilities dropped
- [ ] Host namespaces (hostNetwork, hostPID) disabled
- [ ] Volume types restricted to necessary types
- [ ] SCC assignments reviewed regularly
- [ ] Custom SCCs documented and approved
