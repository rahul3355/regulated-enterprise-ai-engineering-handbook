# Skill: OpenShift Deployments

## Core Principles

1. **Everything Is a Resource** — In OpenShift, deployments, routes, configs, and policies are all first-class API objects managed declaratively.
2. **Security Is Default** — OpenShift runs containers as non-root, enforces SCCs, and requires TLS. Work within these constraints, not against them.
3. **GitOps Is the Way** — Use ArgoCD or OpenShift GitOps for declarative, version-controlled deployments. Manual `oc apply` is for debugging only.
4. **Rolling Updates Are Default but Not Always Safe** — For GenAI services with large models, rolling updates can cause memory pressure. Consider recreate or blue-green strategies.
5. **Observability Is Built-In** — Use OpenShift's integrated monitoring (Prometheus/Grafana) and logging (Loki/Elasticsearch) before adding external tools.

## Mental Models

### OpenShift Deployment Architecture
```
┌─────────────────────────────────────────────────────────────┐
│                    OpenShift Cluster                        │
│                                                             │
│  ┌─────────────┐   ┌─────────────┐   ┌─────────────────┐   │
│  │   Route     │──▶│  Service    │──▶│  Deployment     │   │
│  │ (external)  │   │ (internal)  │   │  (pods)         │   │
│  └─────────────┘   └─────────────┘   └────────┬────────┘   │
│                                                │            │
│                                         ┌──────┴──────┐     │
│                                         │             │     │
│                                  ┌──────▼──┐   ┌─────▼──┐   │
│                                  │ Pod 1   │   │ Pod 2  │   │
│                                  │ :8080   │   │ :8080  │   │
│                                  └─────────┘   └────────┘   │
│                                                             │
│  ┌─────────────┐   ┌─────────────┐   ┌─────────────────┐   │
│  │ ConfigMap   │   │  Secret     │   │ ServiceAccount  │   │
│  │ (app config)│   │ (creds)     │   │ (RBAC)          │   │
│  └─────────────┘   └─────────────┘   └─────────────────┘   │
│                                                             │
│  ┌─────────────┐   ┌─────────────┐   ┌─────────────────┐   │
│  │ HPA         │   │  PDB        │   │ NetworkPolicy   │   │
│  │ (scaling)   │   │ (avail.)    │   │ (network)       │   │
│  └─────────────┘   └─────────────┘   └─────────────────┘   │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### Deployment Strategy Decision Matrix
```
                         Blast Radius of Change
                         Small          Large
Update ──────────────────────────────────────────────
Type     Simple        Rolling        Blue-Green
         config        update         or Canary
         change
         │
         Recreate      Rolling        Blue-Green
         (dev only)    with PDB       (recommended)
```

### OpenShift-Specific Objects Checklist
```
□ ImageStream (internal image management)
□ BuildConfig (source-to-image builds)
□ Route (external HTTPS access)
□ DeploymentConfig (legacy) or Deployment (preferred)
□ ServiceAccount + RoleBinding (RBAC)
□ SecurityContextConstraints (SCCs)
□ NetworkPolicy (namespace isolation)
□ ResourceQuota (namespace resource limits)
□ LimitRange (default container limits)
□ HorizontalPodAutoscaler (auto-scaling)
□ PodDisruptionBudget (availability during updates)
□ ConfigMap + Secret (configuration)
□ PersistentVolumeClaim (storage)
□ HorizontalPodAutoscaler (scaling)
```

## Step-by-Step Approach

### 1. Deploy a GenAI Service Using Declarative YAML

```yaml
# rag-service-deployment.yaml
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: rag-service
  namespace: genai-platform
  labels:
    app: rag-service
    version: v2.3.1
    team: genai-platform
spec:
  replicas: 3
  selector:
    matchLabels:
      app: rag-service
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1        # One extra pod during update
      maxUnavailable: 0  # Zero downtime
  template:
    metadata:
      labels:
        app: rag-service
        version: v2.3.1
      annotations:
        prometheus.io/scrape: "true"
        prometheus.io/port: "8080"
        prometheus.io/path: "/metrics"
    spec:
      # Banking requirement: use a specific service account
      serviceAccountName: rag-service-sa
      # Banking requirement: node selection for GPU nodes if needed
      affinity:
        nodeAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
            - weight: 100
              preference:
                matchExpressions:
                  - key: node-type
                    operator: In
                    values: ["compute-optimized"]
      containers:
        - name: rag-service
          image: registry.internal/banking/genai/rag-service:v2.3.1
          ports:
            - containerPort: 8080
              name: http
              protocol: TCP
          envFrom:
            - configMapRef:
                name: rag-service-config
            - secretRef:
                name: rag-service-secrets
          env:
            - name: POD_NAME
              valueFrom:
                fieldRef:
                  fieldPath: metadata.name
            - name: POD_NAMESPACE
              valueFrom:
                fieldRef:
                  fieldPath: metadata.namespace
          resources:
            requests:
              cpu: 500m
              memory: 512Mi
            limits:
              cpu: "2"
              memory: 2Gi
          readinessProbe:
            httpGet:
              path: /health
              port: 8080
            initialDelaySeconds: 30
            periodSeconds: 10
            timeoutSeconds: 5
            failureThreshold: 3
          livenessProbe:
            httpGet:
              path: /healthz
              port: 8080
            initialDelaySeconds: 60
            periodSeconds: 15
            timeoutSeconds: 5
            failureThreshold: 3
          # Banking requirement: never run as root
          securityContext:
            runAsNonRoot: true
            readOnlyRootFilesystem: true
            allowPrivilegeEscalation: false
            capabilities:
              drop: ["ALL"]
          volumeMounts:
            - name: tmp
              mountPath: /tmp
            - name: config
              mountPath: /app/config
              readOnly: true
      volumes:
        - name: tmp
          emptyDir:
            sizeLimit: 100Mi
        - name: config
          configMap:
            name: rag-service-config
      # Banking requirement: tolerations for dedicated nodes
      tolerations:
        - key: "dedicated"
          operator: "Equal"
          value: "genai"
          effect: "NoSchedule"

---
# Service for internal communication
apiVersion: v1
kind: Service
metadata:
  name: rag-service
  namespace: genai-platform
spec:
  selector:
    app: rag-service
  ports:
    - name: http
      port: 80
      targetPort: 8080
      protocol: TCP
  type: ClusterIP

---
# Route for external access (banking intranet)
apiVersion: route.openshift.io/v1
kind: Route
metadata:
  name: rag-service
  namespace: genai-platform
  annotations:
    haproxy.router.openshift.io/timeout: 60s
    haproxy.router.openshift.io/rate-limit-connections: "true"
    haproxy.router.openshift.io/rate-limit-connections.rate-tcp: "100"
spec:
  host: rag-service.genai-platform.apps.bank.internal
  to:
    kind: Service
    name: rag-service
  port:
    targetPort: http
  tls:
    termination: reencrypt
    insecureEdgeTerminationPolicy: Redirect
```

### 2. Apply with ArgoCD (GitOps Workflow)

```yaml
# argocd-application.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: rag-service
  namespace: openshift-gitops
spec:
  project: genai-platform
  source:
    repoURL: https://github.internal/banking/genai-deployments.git
    targetRevision: main
    path: overlays/production
  destination:
    server: https://kubernetes.default.svc
    namespace: genai-platform
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
      - PruneLast=true
    retry:
      limit: 5
      backoff:
        duration: 5s
        factor: 2
        maxDuration: 3m
  healthCheck:
    - group: apps
      kind: Deployment
      check: |
        hs = {}
        if obj.status ~= nil and obj.status.conditions ~= nil then
          for _, condition in ipairs(obj.status.conditions) do
            if condition.type == "Available" and condition.status == "True" then
              hs.status = "Healthy"
              hs.message = condition.message
              return hs
            end
          end
        end
        hs.status = "Progressing"
        hs.message = "Waiting for deployment to become available"
        return hs
```

### 3. Blue-Green Deployment for Zero-Downtime GenAI Updates

```yaml
# blue-green-deployment.yaml
---
# Blue deployment (currently active)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: rag-service-blue
  namespace: genai-platform
spec:
  replicas: 3
  selector:
    matchLabels:
      app: rag-service
      track: blue
  template:
    metadata:
      labels:
        app: rag-service
        track: blue
        version: v2.3.0
    spec:
      containers:
        - name: rag-service
          image: registry.internal/banking/genai/rag-service:v2.3.0
          # ... same config as above

---
# Green deployment (new version, not yet receiving traffic)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: rag-service-green
  namespace: genai-platform
spec:
  replicas: 3
  selector:
    matchLabels:
      app: rag-service
      track: green
  template:
    metadata:
      labels:
        app: rag-service
        track: green
        version: v2.3.1
    spec:
      containers:
        - name: rag-service
          image: registry.internal/banking/genai/rag-service:v2.3.1
          # ... same config as above

---
# Service points to blue (active)
apiVersion: v1
kind: Service
metadata:
  name: rag-service-active
  namespace: genai-platform
spec:
  selector:
    app: rag-service
    track: blue  # Switch to 'green' to flip traffic
  ports:
    - port: 80
      targetPort: 8080
  type: ClusterIP

---
# Script to switch traffic: oc patch service rag-service-active -n genai-platform \
#   -p '{"spec":{"selector":{"track":"green"}}}'
```

### 4. Configure SecurityContextConstraints (SCCs)

```yaml
# Custom SCC for GenAI services that need more than 'restricted'
apiVersion: security.openshift.io/v1
kind: SecurityContextConstraints
metadata:
  name: genai-service-scc
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
  - emptyDir
  - projected
  - secret
  - downwardAPI
  - persistentVolumeClaim
readOnlyRootFilesystem: true
allowHostDirVolumePlugin: false
allowHostNetwork: false
allowHostPID: false
allowHostPorts: false
defaultAddCapabilities: []
requiredDropCapabilities:
  - ALL
allowedCapabilities: []
priority: 10

# Bind SCC to service account
# oc adm policy add-scc-to-user genai-service-scc -z rag-service-sa -n genai-platform
```

### 5. Configure HorizontalPodAutoscaler

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: rag-service-hpa
  namespace: genai-platform
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: rag-service
  minReplicas: 3
  maxReplicas: 15
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70
    - type: Resource
      resource:
        name: memory
        target:
          type: Utilization
          averageUtilization: 80
    # Custom metric: RAG queue depth
    - type: Pods
      pods:
        metric:
          name: rag_queue_depth
        target:
          type: AverageValue
          averageValue: "100"
  behavior:
    scaleUp:
      stabilizationWindowSeconds: 60
      policies:
        - type: Pods
          value: 2
          periodSeconds: 60
    scaleDown:
      stabilizationWindowSeconds: 300
      policies:
        - type: Pods
          value: 1
          periodSeconds: 120
```

### 6. Configure PodDisruptionBudget

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: rag-service-pdb
  namespace: genai-platform
spec:
  minAvailable: 2  # At least 2 pods must be running during disruptions
  selector:
    matchLabels:
      app: rag-service
```

### 7. Deploy and Verify

```bash
# Deploy via ArgoCD
oc apply -f argocd-application.yaml -n openshift-gitops

# Or deploy directly (for debugging only)
oc apply -f rag-service-deployment.yaml -n genai-platform

# Monitor the rollout
oc rollout status deployment/rag-service -n genai-platform -w

# Check deployment health
oc get deployment rag-service -n genai-platform
oc get pods -l app=rag-service -n genai-platform
oc get endpoints rag-service -n genai-platform

# Verify the route
oc get route rag-service -n genai-platform
curl -k https://$(oc get route rag-service -n genai-platform -o jsonpath='{.spec.host}')/health

# Check resource usage
oc adm top pods -n genai-platform -l app=rag-service

# View events for the deployment
oc get events -n genai-platform --field-selector involvedObject.name=rag-service

# Rollback if something goes wrong
oc rollout undo deployment/rag-service -n genai-platform
```

## Common Pitfalls

| Pitfall | Consequence | Fix |
|---------|------------|-----|
| Using DeploymentConfig instead of Deployment | Missing modern K8s features, harder GitOps | Use `Deployment` kind, not `DeploymentConfig` |
| No resource limits on GenAI pods | Memory pressure kills other pods on the node | Always set requests and limits |
| Readiness probe too aggressive for large models | Pods marked as not ready even though they're loading | Increase `initialDelaySeconds` for model loading |
| Route without TLS termination | Security policy violation, audit failure | Use `reencrypt` or `edge` termination |
| Running containers as root | SCC rejection, security audit failure | Set `runAsNonRoot: true` |
| Not setting `readOnlyRootFilesystem` | Security audit finding, compliance risk | Use `readOnlyRootFilesystem: true` with `/tmp` volume |
| Missing PDB | Rolling update takes all pods down at once | Set `minAvailable` in PDB |
| HPA scaling too aggressively | Cost explosion, cluster instability | Use stabilization windows and conservative policies |
| Not using ArgoCD for production | Manual drift, no audit trail, irreproducible deployments | Adopt GitOps workflow |
| Forgetting `imagePullPolicy: Always` | Running stale images after deployment | Set `imagePullPolicy: Always` or use image digests |
| ConfigMap/Secret not updated before deployment | Pods crash with missing config | Use ArgoCD sync waves or Helm hooks |

## Banking-Specific Concerns

1. **Air-Gapped Deployments** — Banking OpenShift clusters cannot pull from public registries. Images must be mirrored through an internal registry (e.g., Quay/Harbor). Use `ImageStreamTag` references that point to the internal registry.
2. **Audit Trail** — Every deployment change must be tracked. ArgoCD provides this via Git history. Manual `oc apply` commands must be logged and justified.
3. **Segregation of Environments** — Dev, UAT, and Prod clusters must be physically separate. Use Kustomize overlays or Helm values files to manage environment-specific configuration.
4. **Regulatory Change Freeze** — During month-end, quarter-end, or regulatory reporting periods, deployments may be frozen. Plan release calendars accordingly.
5. **DR Deployment** — GenAI services must be deployable to DR clusters with the same configuration. Use GitOps with cluster-specific overlays.
6. **Data Residency** — Some banking data cannot leave specific geographic regions. Ensure pods are scheduled on nodes in compliant regions using node affinity.

## GenAI-Specific Concerns

1. **Model Loading Memory** — Loading a 7B parameter model requires ~14GB of RAM. Ensure node capacity and resource limits account for this. Consider init containers that pre-warm the model cache.
2. **GPU Scheduling** — If using GPU inference, configure GPU resource requests: `nvidia.com/gpu: 1`. Ensure the OpenShift cluster has the NVIDIA GPU Operator installed.
3. **Graceful Shutdown** — GenAI services may be processing requests when a SIGTERM arrives. Set `terminationGracePeriodSeconds: 60` and handle SIGTERM in the application to finish in-flight requests.
4. **Batch Embedding Workloads** — Document ingestion is a batch process. Use `Job` or `CronJob` resources instead of `Deployment` for embedding pipelines.
5. **Multi-Model Routing** — If a single service routes to multiple model backends, use init containers to download model configs and set up routing tables before the main container starts.

## Metrics to Monitor

| Metric | Alert Threshold | Why It Matters |
|--------|----------------|----------------|
| Deployment rollout progress | Not complete in 10 min | Stuck rollout, possible image or config issue |
| Pod ready percentage | < 100% for > 5 min | Deployment health degradation |
| Container restart count | > 2 in 1 hour | Application instability |
| Route response time (p99) | > 2s | Service performance degradation |
| HPA replica count | At max for > 10 min | Insufficient capacity, need higher maxReplicas |
| Node memory pressure | Any node under pressure | Cluster-wide resource exhaustion |
| Image pull time | > 5 min | Large images or registry performance issues |
| SSL certificate expiry | < 30 days | Route TLS certificate expiration |
| ArgoCD sync status | OutOfSync for > 1 hour | GitOps drift, manual changes detected |
| SCC violation count | > 0 | Security policy violation |

## Interview Questions

1. What is the difference between a Deployment and a DeploymentConfig in OpenShift? Which should you use?
2. How would you design a zero-downtime deployment strategy for a GenAI service that takes 3 minutes to load its model?
3. Explain how OpenShift SCCs work and why they matter in a banking environment.
4. How do you configure a Route in OpenShift? What are the different TLS termination types?
5. What happens during a rolling update if you set `maxUnavailable: 0` and `maxSurge: 2`?
6. How would you set up ArgoCD for a multi-environment (dev, UAT, prod) GenAI platform?
7. What is a PodDisruptionBudget and why is it important for high-availability services?
8. How do you handle secrets in OpenShift deployments without exposing them in Git?

## Hands-On Exercise

### Exercise: Deploy a GenAI RAG Service to OpenShift with Zero Downtime

**Problem:** Your team has built a new RAG service (`rag-service:v3.0.0`) that uses a larger embedding model and improved retrieval algorithm. The current version (`v2.3.1`) serves 500 requests per second with 3 replicas. You need to deploy the new version without any downtime.

**Constraints:**
- The new version requires 1.5x more memory (model size increased)
- Model loading takes 90 seconds
- The service must remain available to 10,000+ concurrent users
- Banking policy requires at least 2 replicas running at all times
- Deployment must be managed through ArgoCD (GitOps)
- All security constraints (SCCs, non-root, read-only filesystem) must be met

**Expected Output:**
- Complete Kubernetes manifests (Deployment, Service, Route, HPA, PDB)
- ArgoCD Application manifest
- A deployment runbook with pre-checks, execution steps, and rollback criteria
- Post-deployment verification checklist

**Hints:**
- Use `maxUnavailable: 0` and `maxSurge: 1` for rolling updates
- Set `initialDelaySeconds` high enough for model loading
- Use a PDB to ensure minimum availability during the rollout
- Consider using read probes that check model loading status

**Extension:**
- Implement a blue-green deployment with a traffic switch script
- Add a custom health check that verifies the embedding model is loaded and the vector database connection is healthy
- Design a canary analysis using Prometheus metrics (error rate, latency, throughput) before full rollout

---

**Related files:**
- `kubernetes-openshift/openshift-fundamentals.md` — OpenShift core concepts
- `kubernetes-openshift/kubernetes-security.md` — Security in K8s/OpenShift
- `cicd-devops/argocd-gitops.md` — GitOps with ArgoCD
- `skills/kubernetes-debugging.md` — Debugging K8s issues
- `skills/ci-cd-pipeline-design.md` — CI/CD pipeline design
- `genai-platforms/rag-deployment.md` — RAG deployment patterns
- `security/container-security.md` — Container security best practices
