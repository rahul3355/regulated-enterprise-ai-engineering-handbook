# Kubernetes Deployments: Strategies, Rolling Updates, and Rollbacks

## Overview

Deployments manage the lifecycle of stateless applications in Kubernetes, handling replica scaling, rolling updates, and automatic rollbacks. This guide covers deployment strategies for banking GenAI applications with zero-downtime requirements.

## Deployment Fundamentals

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: genai-api
  namespace: banking-genai
  labels:
    app: genai-api
    team: genai-platform
spec:
  replicas: 3
  selector:
    matchLabels:
      app: genai-api
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1        # Max pods above desired during update
      maxUnavailable: 0  # No pods unavailable (zero-downtime)
  minReadySeconds: 10    # Wait before marking pod as available
  revisionHistoryLimit: 5
  progressDeadlineSeconds: 300
  template:
    metadata:
      labels:
        app: genai-api
        version: "1.2.0"
    spec:
      containers:
        - name: api
          image: quay.io/banking/genai-api:1.2.0
          ports:
            - containerPort: 8080
          resources:
            requests:
              cpu: 250m
              memory: 512Mi
            limits:
              cpu: "1"
              memory: 1Gi
          readinessProbe:
            httpGet:
              path: /health/ready
              port: 8080
            initialDelaySeconds: 10
            periodSeconds: 5
          livenessProbe:
            httpGet:
              path: /health/live
              port: 8080
            initialDelaySeconds: 30
            periodSeconds: 10
```

## Rolling Update Strategy

```
Rolling Update Flow (replicas=3, maxSurge=1, maxUnavailable=0):

Initial:    [v1-1] [v1-2] [v1-3]

Step 1:     [v1-1] [v1-2] [v1-3] [v2-1]  (surge: +1)
            Scale up v2 pod

Step 2:     [v1-1] [v1-2] [v2-1]          (wait for v2-1 ready)
            Terminate v1-3

Step 3:     [v1-1] [v1-2] [v2-1] [v2-2]  (surge: +1)
            Scale up v2 pod

Step 4:     [v1-1] [v2-1] [v2-2]          (terminate v1-2)

Step 5:     [v1-1] [v2-1] [v2-2] [v2-3]  (surge: +1)

Step 6:     [v2-1] [v2-2] [v2-3]          (terminate v1-1)
            Update complete
```

## Deployment Operations

```bash
# Deploy new version
kubectl set image deployment/genai-api api=quay.io/banking/genai-api:1.3.0 -n banking-genai

# Watch rollout status
kubectl rollout status deployment/genai-api -n banking-genai --timeout=300s

# View rollout history
kubectl rollout history deployment/genai-api -n banking-genai
kubectl rollout history deployment/genai-api -n banking-genai --revision=3

# Pause rollout (for manual verification)
kubectl rollout pause deployment/genai-api -n banking-genai

# Resume rollout
kubectl rollout resume deployment/genai-api -n banking-genai

# Rollback to previous version
kubectl rollout undo deployment/genai-api -n banking-genai

# Rollback to specific revision
kubectl rollout undo deployment/genai-api -n banking-genai --to-revision=2

# Scale replicas
kubectl scale deployment/genai-api --replicas=5 -n banking-genai

# Check deployment status
kubectl get deployment genai-api -n banking-genai
kubectl describe deployment genai-api -n banking-genai

# View rollout status details
kubectl get replicaset -n banking-genai -l app=genai-api
```

## Blue-Green Deployment

```yaml
# Blue-Green: Two identical environments, switch traffic
apiVersion: v1
kind: Service
metadata:
  name: genai-api-active
spec:
  selector:
    app: genai-api
    version: "1.2.0"  # Switch this to change active version
  ports:
    - port: 80
      targetPort: 8080
---
# Blue deployment (current)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: genai-api-blue
spec:
  replicas: 3
  selector:
    matchLabels:
      app: genai-api
      version: "1.2.0"
  template:
    metadata:
      labels:
        app: genai-api
        version: "1.2.0"
    spec:
      containers:
        - name: api
          image: quay.io/banking/genai-api:1.2.0
---
# Green deployment (new version)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: genai-api-green
spec:
  replicas: 3
  selector:
    matchLabels:
      app: genai-api
      version: "1.3.0"
  template:
    metadata:
      labels:
        app: genai-api
        version: "1.3.0"
    spec:
      containers:
        - name: api
          image: quay.io/banking/genai-api:1.3.0

# Switch traffic: Change service selector from "1.2.0" to "1.3.0"
kubectl patch service genai-api-active -p '{"spec":{"selector":{"version":"1.3.0"}}}'
```

## Canary Deployment

```yaml
# Canary: Gradually shift traffic to new version
# Requires Istio or NGINX Ingress for traffic splitting
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: genai-api-canary
  annotations:
    nginx.ingress.kubernetes.io/canary: "true"
    nginx.ingress.kubernetes.io/canary-weight: "10"  # 10% traffic to canary
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
apiVersion: apps/v1
kind: Deployment
metadata:
  name: genai-api-canary
spec:
  replicas: 1  # Start with 1 replica
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

# Gradually increase canary traffic: 10% -> 25% -> 50% -> 100%
kubectl patch ingress genai-api-canary \
  -p '{"metadata":{"annotations":{"nginx.ingress.kubernetes.io/canary-weight":"25"}}}'
```

## Cross-References

- **Blue-Green Deployments**: See [blue-green-deployments.md](blue-green-deployments.md) for detailed patterns
- **Canary Releases**: See [canary-releases.md](canary-releases.md) for gradual rollout
- **Autoscaling**: See [autoscaling.md](autoscaling.md) for replica management

## Interview Questions

1. **How does a rolling update work in Kubernetes? What do maxSurge and maxUnavailable control?**
2. **How do you rollback a failed deployment in Kubernetes?**
3. **Compare rolling updates, blue-green, and canary deployments. When do you use each?**
4. **How do you ensure zero-downtime deployments for a banking API?**
5. **What happens if a new pod fails its readiness probe during a rolling update?**
6. **How do you automate canary promotion based on metrics?**

## Checklist: Deployment Best Practices

- [ ] Rolling update strategy with maxUnavailable=0 for zero-downtime
- [ ] minReadySeconds set to verify pod stability
- [ ] progressDeadlineSeconds set to detect stalled rollouts
- [ ] revisionHistoryLimit configured (keep last 3-5)
- [ ] Resource requests and limits defined
- [ ] Readiness and liveness probes configured
- [ ] Image tags are immutable (use SHA or version tags, not :latest)
- [ ] Pod disruption budgets for minimum availability
- [ ] Rollback procedure documented and tested
- [ ] Deployment automated via CI/CD pipeline
