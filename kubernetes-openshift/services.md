# Kubernetes Services: Types, DNS, and Discovery

## Overview

Services provide stable network endpoints for pods in Kubernetes. They enable service discovery, load balancing, and network abstraction between microservices. This guide covers service types, DNS resolution, and banking-specific networking patterns.

## Service Types

```mermaid
graph LR
    subgraph ClusterIP (Internal)
    C1[Client Pod] --> C2[Service:80]
    C2 --> P1[Pod 1:8080]
    C2 --> P2[Pod 2:8080]
    end
    
    subgraph NodePort (External via Node)
    N1[External] --> N2[Node:30080]
    N2 --> N3[Service:80]
    N3 --> P3[Pod 1]
    end
    
    subgraph LoadBalancer (Cloud LB)
    L1[External] --> L2[Cloud LB]
    L2 --> L3[Service:80]
    L3 --> P4[Pod 1]
    end
```

| Type | Accessibility | Use Case | Banking Fit |
|------|--------------|----------|-------------|
| ClusterIP | Internal only | Microservice communication | Primary (default) |
| NodePort | Node IP + Port | Debugging, direct access | Rare (security) |
| LoadBalancer | External IP | Public APIs | With WAF/ingress |
| ExternalName | DNS alias | External service proxy | Database connections |

## ClusterIP Service (Internal)

```yaml
# Default service type: accessible only within the cluster
apiVersion: v1
kind: Service
metadata:
  name: genai-api
  namespace: banking-genai
  labels:
    app: genai-api
spec:
  type: ClusterIP
  selector:
    app: genai-api
  ports:
    - name: http
      port: 80
      targetPort: 8080
      protocol: TCP
    - name: metrics
      port: 9090
      targetPort: 9090
      protocol: TCP
  # Session affinity (optional)
  sessionAffinity: None  # or ClientIP
---
# Headless service (no load balancing, returns all pod IPs)
apiVersion: v1
kind: Service
metadata:
  name: genai-api-headless
  namespace: banking-genai
spec:
  clusterIP: None  # Headless
  selector:
    app: genai-api
  ports:
    - name: http
      port: 80
      targetPort: 8080
```

## Service Discovery

```
DNS Resolution in Kubernetes:

Service: genai-api in namespace: banking-genai

Full DNS: genai-api.banking-genai.svc.cluster.local
Short DNS: genai-api.banking-genai
Same namespace: genai-api

Resolution flow:
1. Pod queries CoreDNS
2. CoreDNS returns Service ClusterIP
3. kube-proxy routes to one of the backing pods

# Pod DNS configuration
# /etc/resolv.conf inside pod:
nameserver 10.96.0.10  # CoreDNS
search banking-genai.svc.cluster.local svc.cluster.local cluster.local
options ndots:5
```

```python
# Python service discovery example
import os
import requests

# Access service via DNS
GENAI_API_URL = os.getenv('GENAI_API_URL', 'http://genai-api.banking-genai:80')

def call_genai_api(prompt: str) -> dict:
    """Call GenAI API using Kubernetes service discovery."""
    response = requests.post(
        f"{GENAI_API_URL}/generate",
        json={"prompt": prompt},
        timeout=30,
    )
    response.raise_for_status()
    return response.json()

# With headless service (direct pod access)
import dns.resolver

def get_genai_api_pods():
    """Get all pod IPs via headless service DNS."""
    answers = dns.resolver.resolve('genai-api-headless.banking-genai.svc.cluster.local', 'A')
    return [str(rdata) for rdata in answers]
```

## ExternalName Service

```yaml
# Proxy to external service (e.g., managed database)
apiVersion: v1
kind: Service
metadata:
  name: external-postgres
  namespace: banking-data
spec:
  type: ExternalName
  externalName: postgres-prod.managed-db.internal
  ports:
    - port: 5432
---
# Application uses internal DNS name
# DATABASE_URL=postgresql://user:pass@external-postgres:5432/banking
```

## Endpoint Slices

```bash
# View service endpoints
kubectl get endpoints genai-api -n banking-genai
kubectl get endpointslices -n banking-genai

# Manually add endpoint (rare)
kubectl create endpoint manual-svc --from-ip=10.0.1.5 -n banking-genai
```

## Cross-References

- **Ingress**: See [ingress.md](ingress.md) for external routing
- **Network Policies**: See [network-policies.md](network-policies.md) for security
- **OpenShift Routes**: See [openshift-routes.md](openshift-routes.md) for OpenShift-specific routing

## Interview Questions

1. **What are the different Kubernetes service types? When do you use each?**
2. **How does Kubernetes service discovery work? What is the DNS format?**
3. **What is a headless service? When would you use it?**
4. **How does kube-proxy route traffic from a service to pods?**
5. **What is the difference between ClusterIP and ExternalName services?**
6. **How do you debug service connectivity issues in Kubernetes?**

## Checklist: Service Configuration

- [ ] ClusterIP used for internal microservices
- [ ] Service ports named for clarity (http, metrics, grpc)
- [ ] Selector matches pod labels
- [ ] Headless service used for stateful sets (direct pod access)
- [ ] ExternalName used for external service proxying
- [ ] DNS names documented in runbooks
- [ ] Endpoint health verified
- [ ] Service mesh (Istio) evaluated for advanced routing
