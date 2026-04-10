# Multi-Cluster Kubernetes Deployments

## Overview

Multi-cluster deployments enable banking GenAI platforms to achieve high availability across regions, meet data residency requirements, and isolate environments. This guide covers cluster federation, GitOps-based multi-cluster management, and disaster recovery patterns.

## Multi-Cluster Architecture

```mermaid
graph TB
    subgraph Region 1 (us-east)
    C1[Cluster 1 - Active]
    E1[GenAI API x3]
    D1[(PostgreSQL Primary)]
    C1 --> E1
    E1 --> D1
    end
    
    subgraph Region 2 (us-west)
    C2[Cluster 2 - Standby]
    E2[GenAI API x3]
    D2[(PostgreSQL Standby)]
    C2 --> E2
    E2 --> D2
    end
    
    D1 -->|Replication| D2
    LB[Global Load Balancer] --> C1
    LB --> C2
```

## GitOps for Multi-Cluster

```yaml
# ArgoCD Application for multi-cluster deployment
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: genai-api
  namespace: argocd
spec:
  project: banking-genai
  source:
    repoURL: https://github.com/bank/genai-deployments.git
    targetRevision: main
    path: clusters/us-east/genai-api
  destination:
    server: https://api.us-east.cluster.bank.com:6443
    namespace: banking-genai
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
---
# Second cluster
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: genai-api-west
  namespace: argocd
spec:
  project: banking-genai
  source:
    repoURL: https://github.com/bank/genai-deployments.git
    targetRevision: main
    path: clusters/us-west/genai-api
  destination:
    server: https://api.us-west.cluster.bank.com:6443
    namespace: banking-genai
```

## Cross-References

- **Disaster Recovery**: See [disaster-recovery.md](../databases/disaster-recovery.md) for DB-level DR
- **Blue-Green**: See [blue-green-deployments.md](blue-green-deployments.md) for deployment patterns

## Interview Questions

1. **Why would you run multiple Kubernetes clusters for a banking platform?**
2. **How do you manage deployments across multiple clusters?**
3. **What is GitOps and how does it help with multi-cluster management?**
4. **How do you handle data synchronization between clusters?**
5. **What are the challenges of multi-cluster service discovery?**
6. **How do you perform failover between clusters?**

## Checklist: Multi-Cluster

- [ ] Cluster naming convention established
- [ ] GitOps tool configured for all clusters
- [ ] Global load balancer configured
- [ ] Data replication between regions
- [ ] Failover procedure documented and tested
- [ ] Monitoring aggregated across clusters
- [ ] Cross-cluster networking secure
- [ ] Cluster provisioning automated
