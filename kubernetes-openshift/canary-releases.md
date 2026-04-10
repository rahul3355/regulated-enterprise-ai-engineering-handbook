# Canary Releases in Kubernetes

## Overview

Canary releases gradually shift traffic to a new version, allowing validation with real traffic before full rollout. This guide covers canary patterns for banking GenAI applications.

## Canary with NGINX Ingress

```yaml
# Production service (stable)
apiVersion: v1
kind: Service
metadata:
  name: genai-api
spec:
  selector:
    app: genai-api
    track: stable
  ports:
    - port: 80
      targetPort: 8080
---
# Canary service
apiVersion: v1
kind: Service
metadata:
  name: genai-api-canary
spec:
  selector:
    app: genai-api
    track: canary
  ports:
    - port: 80
      targetPort: 8080
---
# Canary ingress (10% traffic)
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: genai-api-canary
  annotations:
    nginx.ingress.kubernetes.io/canary: "true"
    nginx.ingress.kubernetes.io/canary-weight: "10"
spec:
  rules:
    - host: genai-api.bank.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: genai-api-canary
                port:
                  number: 80
---
# Canary deployment
apiVersion: apps/v1
kind: Deployment
metadata:
  name: genai-api-canary
spec:
  replicas: 1
  selector:
    matchLabels:
      app: genai-api
      track: canary
  template:
    metadata:
      labels:
        app: genai-api
        track: canary
    spec:
      containers:
        - name: api
          image: quay.io/banking/genai-api:1.3.0
```

## Automated Canary Promotion

```yaml
# Argo Rollouts for automated canary
apiVersion: argoproj.io/v1alpha1
kind: Rollout
metadata:
  name: genai-api
spec:
  replicas: 3
  strategy:
    canary:
      steps:
        - setWeight: 10
        - pause: {duration: 5m}
        - analysis:
            templates:
              - templateName: error-rate-check
        - setWeight: 25
        - pause: {duration: 5m}
        - setWeight: 50
        - pause: {duration: 10m}
        - setWeight: 75
        - pause: {duration: 10m}
        - setWeight: 100
---
apiVersion: argoproj.io/v1alpha1
kind: AnalysisTemplate
metadata:
  name: error-rate-check
spec:
  metrics:
    - name: error-rate
      interval: 1m
      successCondition: result[0] < 0.01
      failureLimit: 3
      provider:
        prometheus:
          address: http://prometheus.monitoring:9090
          query: |
            sum(rate(http_requests_total{service="genai-api",status=~"5.."}[1m]))
            /
            sum(rate(http_requests_total{service="genai-api"}[1m]))
```

## Cross-References

- **Blue/Green**: See [blue-green-deployments.md](blue-green-deployments.md) for instant switch
- **Progressive Delivery**: See [progressive-delivery.md](../cicd-devops/progressive-delivery.md) for broader patterns

## Interview Questions

1. **What is canary deployment? How does it differ from blue/green?**
2. **How do you implement automated canary promotion?**
3. **What metrics should you monitor during a canary release?**
4. **How do you rollback a failed canary?**
5. **What percentage of traffic should you start with for a canary?**
6. **How does Argo Rollouts automate canary deployments?**

## Checklist: Canary Releases

- [ ] Canary starts with small traffic percentage (5-10%)
- [ ] Metrics monitored during each stage
- [ ] Automated rollback on error threshold breach
- [ ] Gradual traffic increase (10% -> 25% -> 50% -> 100%)
- [ ] Each stage has sufficient observation time
- [ ] Error rate, latency, and business metrics tracked
- [ ] Rollback procedure automated
- [ ] Team notified of canary status
