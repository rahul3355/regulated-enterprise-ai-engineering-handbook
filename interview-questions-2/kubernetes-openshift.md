# Kubernetes & OpenShift Interview Questions — 20 Questions with Full Answers

## Overview
| Detail | Value |
|--------|-------|
| Topic | Kubernetes & OpenShift (Pods, Deployments, Services, Helm, RBAC, Operators, Routes, SCC, GPU Workloads, Production Debugging) |
| Questions | 20 (5 Must-Know, 10 Medium, 5 Advanced) |
| Source Files | kubernetes-openshift/ (23 files: pods, deployments, services, ingress, configmaps, secrets, statefulsets, helm, autoscaling, rbac, network-policies, gpu-workloads, debugging-pods, blue-green-deployments, canary-releases, multi-cluster, production-incidents, openshift-routes, openshift-operators, openshift-pipelines, openshift-scc, secure-deployment-patterns) |
| Citi Relevance | Citi runs everything on Red Hat OpenShift. Understanding both vanilla Kubernetes AND OpenShift-specific features (Operators, Routes, SCC) is essential for this role. |

---

## Difficulty Legend
- 🔵 **Must-Know** — Foundational. Every candidate should know these.
- 🟡 **Medium** — Core competency. What you'll actually be tested on.
- 🔴 **Advanced** — Differentiator. Shows principal-engineer depth.

---

## Questions & Answers

### Q1: 🔵 Explain the complete Pod lifecycle from creation to termination. How do liveness, readiness, and startup probes fit into this lifecycle, and why are all three important for a banking GenAI API?

**Strong Answer:**

A Pod goes through five distinct phases: **Pending**, **Running**, **Succeeded**, **Failed**, and **Unknown**. When you apply a Deployment manifest, the scheduler first places the Pod on a suitable node (Pending phase — pulling images, allocating resources). Once all containers start, the Pod enters Running. If all containers exit with code 0, it becomes Succeeded (typical for Jobs). If any container exits non-zero and the restart policy doesn't recover it, the Pod enters Failed. Unknown means the node has lost contact with the control plane.

Within the Running phase, the three probes govern how Kubernetes treats the Pod:

- **Startup Probe**: This runs first and disables liveness/readiness checks until it succeeds. For a GenAI API that loads a 4 GB embedding model on startup (which can take 30–60 seconds), the startup probe prevents the liveness probe from killing the container prematurely. You set a high `failureThreshold` (e.g., 30) with a reasonable `periodSeconds` (e.g., 10) to allow up to 5 minutes for startup.

- **Readiness Probe**: Once the startup probe succeeds, readiness determines whether the Pod receives traffic through its Service. If the readiness endpoint `/health/ready` returns non-200, the Pod is removed from the Service's endpoint list. During a rolling update, this ensures zero-downtime — the old Pod stays in service until the new Pod passes readiness.

- **Liveness Probe**: This checks if the application is internally healthy. If it fails repeatedly, Kubernetes kills and restarts the container. Critical caveat: a poorly designed liveness probe that checks an overloaded database dependency can cause cascading restarts. Best practice is to check only the process itself (e.g., a simple HTTP 200 from an in-memory endpoint).

In a banking context, probes are not optional — they are a regulatory requirement for high availability. The preStop lifecycle hook (e.g., `sleep 15`) combined with `terminationGracePeriodSeconds: 30` ensures in-flight requests complete gracefully before the Pod is removed, which is essential for audit-compliant transaction processing.

```yaml
startupProbe:
  httpGet:
    path: /health/startup
    port: 8080
  failureThreshold: 30
  periodSeconds: 10
readinessProbe:
  httpGet:
    path: /health/ready
    port: 8080
  initialDelaySeconds: 10
  periodSeconds: 5
  failureThreshold: 3
livenessProbe:
  httpGet:
    path: /health/live
    port: 8080
  initialDelaySeconds: 30
  periodSeconds: 10
  failureThreshold: 3
lifecycle:
  preStop:
    exec:
      command: ["/bin/sh", "-c", "sleep 15"]
terminationGracePeriodSeconds: 30
```

**Key Points to Hit:**
- [ ] Five Pod phases: Pending, Running, Succeeded, Failed, Unknown
- [ ] Startup probe protects slow-starting containers (model loading)
- [ ] Readiness probe controls traffic routing — critical for zero-downtime
- [ ] Liveness probe should only check process health, not dependencies
- [ ] preStop hook + terminationGracePeriodSeconds for graceful shutdown

**Follow-Up Questions:**
1. What happens if the liveness probe and readiness probe check the same endpoint?
2. How would you configure probes for a Pod that loads a 10 GB LLM model at startup?
3. What is the difference between `restartPolicy: Always`, `OnFailure`, and `Never`?

**Source:** `banking-genai-engineering-academy/kubernetes-openshift/pods.md`, `banking-genai-engineering-academy/kubernetes-openshift/debugging-pods.md`

---

### Q2: 🔵 What are resource requests and limits in Kubernetes? What happens when a container exceeds its memory limit vs. its CPU limit?

**Strong Answer:**

Resource **requests** are guarantees — the scheduler uses them to decide which node can host the Pod. Resource **limits** are ceilings — the container cannot exceed them. Requests affect scheduling; limits affect runtime behavior.

When a container exceeds its **memory limit**, the kernel's OOM (Out-Of-Memory) killer terminates the process immediately. Kubernetes reports this as `OOMKilled` with exit code 137. The restart policy determines whether the container is restarted. For a GenAI API, OOM kills during traffic spikes are a common production incident — the fix is either increasing the limit or implementing backpressure at the application level.

When a container exceeds its **CPU limit**, it is **throttled** (not killed). The container gets CPU time proportional to its limit relative to other containers. This can cause increased latency but not crashes. For a Python-based GenAI service doing heavy computation (e.g., batch embedding generation), CPU throttling will manifest as slow response times, which the readiness probe should detect.

```yaml
resources:
  requests:
    cpu: 250m       # Guaranteed: 0.25 CPU core
    memory: 512Mi   # Guaranteed: 512 MiB RAM
  limits:
    cpu: "1"        # Throttled beyond 1 core
    memory: 1Gi     # OOMKilled beyond 1 GiB
```

In banking environments, setting appropriate requests and limits is not just an operational concern — it is a capacity planning and compliance requirement. Under-provisioning leads to noisy-neighbor problems and SLA breaches. Over-provisioning wastes expensive GPU nodes. A common pattern is to start with VPA in recommendation mode to observe actual usage, then set requests at the 90th percentile and limits at 120% of the observed peak.

The `ephemeral-storage` request/limit is also worth mentioning — it controls disk usage for container writable layers and emptyDir volumes, preventing a single Pod from filling the node's disk.

**Key Points to Hit:**
- [ ] Requests = scheduling guarantee; Limits = runtime ceiling
- [ ] Memory exceeded → OOMKilled (exit 137); CPU exceeded → throttling
- [ ] VPA in recommendation mode to right-size resources
- [ ] ephemeral-storage limits prevent node disk exhaustion
- [ ] Banking context: capacity planning and SLA compliance

**Follow-Up Questions:**
1. How does the OOM score work when multiple containers share a node?
2. What is the difference between `requests` and `limits` for burstable workloads?
3. How would you detect which pods are being CPU-throttled in production?

**Source:** `banking-genai-engineering-academy/kubernetes-openshift/pods.md`, `banking-genai-engineering-academy/kubernetes-openshift/autoscaling.md`, `banking-genai-engineering-academy/kubernetes-openshift/production-incidents.md`

---

### Q3: 🔵 Explain how a Kubernetes Deployment rolling update works. What do `maxSurge` and `maxUnavailable` control, and how would you configure them for zero-downtime deployment of a banking API?

**Strong Answer:**

A rolling update replaces old Pods with new ones gradually, ensuring the application remains available throughout. The Deployment controller creates a new ReplicaSet with the updated Pod template and scales it up while scaling down the old ReplicaSet.

`maxSurge` controls how many **extra** Pods can exist above the desired replica count during the update. `maxUnavailable` controls how many Pods can be **unavailable** during the update.

For zero-downtime deployment of a banking API with 3 replicas:

```yaml
strategy:
  type: RollingUpdate
  rollingUpdate:
    maxSurge: 1        # Create 1 new Pod before removing any old ones
    maxUnavailable: 0  # Never reduce below 3 serving Pods
minReadySeconds: 10    # Wait 10s after readiness before considering Pod available
progressDeadlineSeconds: 300  # Fail rollout if not complete in 5 minutes
revisionHistoryLimit: 5  # Keep last 5 revisions for rollback
```

The step-by-step flow with `replicas=3, maxSurge=1, maxUnavailable=0`:

1. **Initial state**: `[v1-A] [v1-B] [v1-C]` — 3 Pods serving
2. **Step 1**: Scale up to 4 Pods — `[v1-A] [v1-B] [v1-C] [v2-A]` (surge +1)
3. **Step 2**: Wait for v2-A readiness probe to pass (minReadySeconds: 10)
4. **Step 3**: Terminate v1-C — `[v1-A] [v1-B] [v2-A]` (still 3 available)
5. **Step 4**: Scale up to 4 — `[v1-A] [v1-B] [v2-A] [v2-B]`
6. **Step 5**: Wait for v2-B readiness, terminate v1-B
7. **Step 6**: Scale up to 4 — `[v1-A] [v2-A] [v2-B] [v2-C]`
8. **Step 7**: Wait for v2-C readiness, terminate v1-A
9. **Complete**: `[v2-A] [v2-B] [v2-C]` — all new version

Setting `maxUnavailable: 0` guarantees that at no point do we have fewer than 3 healthy Pods. This is essential for a banking API where even brief unavailability can impact trading systems, customer transactions, or regulatory reporting. The `minReadySeconds` prevents premature removal of old Pods — it verifies the new Pod is stable before proceeding.

If the new Pod fails its readiness probe, the rollout pauses automatically. The Deployment controller will not terminate more old Pods until the new ones are ready. You can then rollback with `kubectl rollout undo deployment/genai-api`.

**Key Points to Hit:**
- [ ] maxSurge = extra Pods allowed; maxUnavailable = pods allowed to be down
- [ ] Zero-downtime: maxUnavailable=0, maxSurge=1 (or higher for faster rollout)
- [ ] minReadySeconds verifies Pod stability before proceeding
- [ ] progressDeadlineSeconds detects stalled rollouts
- [ ] Rollback available via `kubectl rollout undo` to any revision

**Follow-Up Questions:**
1. What happens if a new Pod passes readiness but then crashes 5 seconds later?
2. How would you pause a rollout to run manual integration tests?
3. Compare rolling updates with blue-green and canary deployments — when would you use each?

**Source:** `banking-genai-engineering-academy/kubernetes-openshift/deployments.md`, `banking-genai-engineering-academy/kubernetes-openshift/blue-green-deployments.md`

---

### Q4: 🔵 What are the different Kubernetes Service types? Explain when you would use each type in a banking environment, and how does service discovery work?

**Strong Answer:**

There are four Service types in Kubernetes, each with a specific networking scope:

**ClusterIP** (default): Exposes the Service on an internal cluster IP only. Pods within the cluster can reach it via DNS: `genai-api.banking-genai.svc.cluster.local`. This is the primary service type for banking microservices — all inter-service communication (GenAI API to PostgreSQL, API to Redis) uses ClusterIP. It is the most secure option because nothing is exposed externally.

**NodePort**: Exposes the Service on each node's IP at a static port (30000–32767). External clients can reach it at `NodeIP:NodePort`. In banking, this is rarely used in production due to security concerns — it bypasses load balancers and network policies. It is sometimes used for debugging or for tools that need direct node access.

**LoadBalancer**: Provisions an external load balancer (cloud provider or MetalLB on-prem) that forwards traffic to the Service. In a cloud banking environment, this might front the ingress controller. On OpenShift, Routes are preferred over LoadBalancer Services.

**ExternalName**: Maps the Service to an external DNS name (CNAME record). It creates no selector-based endpoints — it just returns a CNAME. Useful for proxying external services (e.g., a managed PostgreSQL instance at `postgres-prod.managed-db.internal`). Applications inside the cluster use the internal DNS name `external-postgres.banking-data`, making external dependencies swappable.

```yaml
# ClusterIP: internal microservice communication
apiVersion: v1
kind: Service
metadata:
  name: genai-api
  namespace: banking-genai
spec:
  type: ClusterIP
  selector:
    app: genai-api
  ports:
    - name: http
      port: 80
      targetPort: 8080
---
# ExternalName: proxy to managed database
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
```

**Service Discovery**: When a Pod needs to call `genai-api.banking-genai`, it queries CoreDNS (the cluster DNS server at `10.96.0.10`). CoreDNS returns the ClusterIP of the Service. Then `kube-proxy` (running on every node) intercepts traffic to that ClusterIP and load-balances across the backing Pods using iptables or IPVS rules. The application code doesn't need to know individual Pod IPs — it just uses the stable DNS name.

A **headless Service** (`clusterIP: None`) is a special case — it skips kube-proxy's load balancing and returns all Pod IPs directly via DNS. This is used with StatefulSets (databases) when the application needs to connect to specific pods (e.g., primary vs. replica in PostgreSQL).

**Key Points to Hit:**
- [ ] ClusterIP = default, internal only; most common in banking
- [ ] NodePort = direct node access; rarely used in production
- [ ] LoadBalancer = external LB; on OpenShift, Routes are preferred
- [ ] ExternalName = CNAME to external service
- [ ] DNS format: `service-name.namespace.svc.cluster.local`
- [ ] Headless Service (clusterIP: None) returns all Pod IPs for StatefulSets

**Follow-Up Questions:**
1. How does kube-proxy implement service load balancing under the hood?
2. When would you use a headless Service instead of a regular ClusterIP?
3. How do you debug service connectivity issues (Pod A cannot reach Service B)?

**Source:** `banking-genai-engineering-academy/kubernetes-openshift/services.md`, `banking-genai-engineering-academy/kubernetes-openshift/statefulsets.md`

---

### Q5: 🔵 What is the difference between mounting a ConfigMap/Secret as a volume vs. injecting it as an environment variable? Which approach is better for banking applications?

**Strong Answer:**

There are two primary ways to consume ConfigMaps and Secrets in Kubernetes:

**Environment Variables**: Values are injected at container creation time. The Pod's environment contains the key-value pairs. This approach is simple but has a critical limitation — **changes to the ConfigMap or Secret do not propagate to running Pods**. You must restart the Pod for changes to take effect.

```yaml
env:
  - name: APP_ENV
    valueFrom:
      configMapKeyRef:
        name: genai-api-config
        key: APP_ENV
  - name: DATABASE_URL
    valueFrom:
      secretKeyRef:
        name: db-credentials
        key: url
```

**Volume Mounts**: The ConfigMap or Secret is projected as files in a directory. Kubernetes automatically updates the files when the ConfigMap/Secret changes (within ~60 seconds). Applications can implement a **hot reload** pattern by watching the file for changes using a library like `watchdog` in Python, or by polling the file's hash.

```yaml
volumeMounts:
  - name: config-volume
    mountPath: /app/config
    readOnly: true
volumes:
  - name: config-volume
    configMap:
      name: genai-api-config
```

**For banking applications, volume mounts are generally preferred** for several reasons:

1. **Secret rotation**: Banking compliance (e.g., PCI DSS, SOX) mandates periodic credential rotation. With volume-mounted secrets, the application can reload credentials without restarting. When combined with External Secrets Operator syncing from HashiCorp Vault, this enables automated rotation on a 24-hour cycle.

2. **Hot reload of configuration**: Log levels, rate limits, and feature flags can change at runtime. Volume-mounted ConfigMaps allow dynamic reconfiguration without deployment restarts, which is critical for 24/7 banking services.

3. **Security**: Volume mounts allow restrictive file permissions (`defaultMode: 0400` — read-only for owner), whereas environment variables are visible to any process in the container via `/proc/<pid>/environ`.

4. **Large configuration**: ConfigMaps support multi-line YAML/JSON configuration files (up to 1 MB). Environment variables are awkward for structured config.

The exception: for values that **never change** at runtime (e.g., `APP_ENV=production`), environment variables are fine and simpler.

**Key Points to Hit:**
- [ ] Env vars are static (set at container creation); volumes auto-update
- [ ] Volume mounts enable hot reload via file watching
- [ ] Volume mounts support restrictive file permissions (0400)
- [ ] Env vars visible via /proc/<pid>/environ — security concern
- [ ] Banking compliance requires secret rotation — volumes support this without restart

**Follow-Up Questions:**
1. How does the hot reload mechanism work with ConfigMap volume mounts?
2. What is the External Secrets Operator and how does it integrate with HashiCorp Vault?
3. How would you implement graceful secret rotation in a Python application without dropping in-flight requests?

**Source:** `banking-genai-engineering-academy/kubernetes-openshift/configmaps.md`, `banking-genai-engineering-academy/kubernetes-openshift/secrets.md`

---

### Q6: 🟡 Explain how Helm works and its chart structure. How do you manage environment-specific configurations (dev, staging, prod) for a banking GenAI application using Helm?

**Strong Answer:**

Helm is the package manager for Kubernetes. It solves three key problems: **templating** (parameterizing Kubernetes manifests), **versioning** (tracking releases), and **dependency management** (bundling sub-charts like Redis or PostgreSQL).

**Chart Structure**:

```
banking-genai-api/
├── Chart.yaml          # Metadata: name, version, appVersion, dependencies
├── values.yaml         # Default configuration values
├── values-prod.yaml    # Production-specific overrides
├── values-staging.yaml # Staging-specific overrides
├── templates/
│   ├── deployment.yaml
│   ├── service.yaml
│   ├── configmap.yaml
│   ├── hpa.yaml
│   └── _helpers.tpl    # Reusable template functions
└── README.md
```

**Chart.yaml** defines the chart version (Helm release version) and the appVersion (the actual application version being deployed):

```yaml
apiVersion: v2
name: banking-genai-api
description: GenAI API service for banking platform
type: application
version: 1.0.0          # Chart version (increment with template changes)
appVersion: "1.2.0"     # Application version (the actual image tag)
dependencies:
  - name: redis
    version: "17.0.0"
    repository: https://charts.bitnami.com/bitnami
    condition: redis.enabled
```

**Environment-specific management** uses separate values files:

```bash
# Dev: use defaults from values.yaml
helm install genai-api ./banking-genai-api -n banking-genai-dev

# Staging: override with staging values
helm install genai-api ./banking-genai-api -n banking-genai-staging -f values-staging.yaml

# Production: override with production values
helm install genai-api ./banking-genai-api -n banking-genai-prod -f values-prod.yaml
```

The `values-prod.yaml` overrides key production settings:

```yaml
# values-prod.yaml
replicaCount: 5
rollingUpdate:
  maxSurge: 2
  maxUnavailable: 0
resources:
  requests:
    cpu: "1"
    memory: 2Gi
  limits:
    cpu: "2"
    memory: 4Gi
image:
  tag: "1.2.0"
redis:
  enabled: false  # Production uses external Redis cluster
```

In the templates, values are accessed via `{{ .Values.replicaCount }}`, `{{ .Values.resources.requests.cpu }}`, etc.

For banking environments, Helm releases should be integrated into CI/CD (Tekton on OpenShift) with `helm lint` as a quality gate, `helm template` for dry-run validation, and `helm upgrade --install` for idempotent deployments. Rollbacks are straightforward: `helm rollback genai-api 1 -n banking-genai-prod`.

**Key Points to Hit:**
- [ ] Helm solves templating, versioning, and dependency management
- [ ] Chart.yaml has chart version (Helm) vs appVersion (application)
- [ ] Environment-specific values files override defaults
- [ ] Templates use Go template syntax with {{ .Values.* }}
- [ ] helm lint + helm template for pre-deployment validation
- [ ] helm rollback for instant release rollback

**Follow-Up Questions:**
1. What is the difference between chart version and appVersion? Why do both exist?
2. How do Helm hooks work, and when would you use them?
3. How do you share common configuration across multiple Helm charts in a banking platform?

**Source:** `banking-genai-engineering-academy/kubernetes-openshift/helm.md`, `banking-genai-engineering-academy/kubernetes-openshift/deployments.md`

---

### Q7: 🟡 Explain the Horizontal Pod Autoscaler (HPA) in Kubernetes. How would you configure it for a GenAI API that experiences unpredictable traffic spikes, and what are the risks of aggressive autoscaling?

**Strong Answer:**

The HPA automatically adjusts the number of Pod replicas based on observed metrics. In Kubernetes v2 (the current API), HPA supports CPU, memory, custom metrics (from Prometheus via the metrics-server), and external metrics (from KEDA).

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: genai-api-hpa
  namespace: banking-genai
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: genai-api
  minReplicas: 3
  maxReplicas: 20
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
  behavior:
    scaleUp:
      stabilizationWindowSeconds: 60
      policies:
        - type: Pods
          value: 2
          periodSeconds: 60
        - type: Percent
          value: 50
          periodSeconds: 60
      selectPolicy: Max
    scaleDown:
      stabilizationWindowSeconds: 300
      policies:
        - type: Pods
          value: 1
          periodSeconds: 120
      selectPolicy: Min
```

**Key configuration decisions**:

- **minReplicas: 3** ensures baseline availability even at zero traffic. For a banking API, you never want to scale below the number required to serve requests during off-peak hours.
- **maxReplicas: 20** prevents runaway scaling. Without a cap, a misconfigured metric could cause the HPA to scale to hundreds of replicas, exhausting cluster resources.
- **CPU target: 70%** leaves headroom for traffic spikes. Setting it too high (e.g., 90%) means there is no buffer before pods become unresponsive. Setting it too low (e.g., 40%) wastes resources.
- **Scale-down stabilization: 300 seconds** (5 minutes) prevents "flapping" — rapidly scaling up and down during variable traffic. GenAI workloads often have bursty patterns (e.g., end-of-day batch processing), so a longer cooldown is essential.

**Risks of aggressive autoscaling**:

1. **Cold start latency**: New Pods take time to pull images, start containers, load models, and pass readiness probes. If traffic spikes faster than scaling, requests fail.
2. **Resource contention**: Scaling to maxReplicas can exhaust node resources (CPU, memory, GPU), causing existing Pods to be throttled or OOMKilled.
3. **Cascade failures**: If the autoscaler scales up too fast and the cluster cannot provision nodes fast enough (Cluster Autoscaler lag), Pods remain Pending, and the entire service degrades.
4. **Cost explosion**: In cloud environments, aggressive scaling can lead to unexpected infrastructure costs.

For GenAI-specific workloads, KEDA (Kubernetes Event-Driven Autoscaling) is often more appropriate than HPA. Instead of scaling on CPU, KEDA can scale on queue depth (Kafka lag), embedding request queue size, or custom Prometheus metrics — which better reflect actual workload demand.

**Key Points to Hit:**
- [ ] HPA scales based on CPU, memory, or custom metrics
- [ ] minReplicas ensures baseline; maxReplicas prevents runaway scaling
- [ ] Scale-down stabilization prevents flapping (300s recommended)
- [ ] Risks: cold starts, resource contention, cascade failures, cost
- [ ] KEDA is better for event-driven scaling (queue depth, Kafka lag)

**Follow-Up Questions:**
1. How does the HPA calculate the desired replica count from metrics?
2. What is the difference between HPA and VPA? Can you use both together?
3. How would you autoscale a GPU-based inference service?

**Source:** `banking-genai-engineering-academy/kubernetes-openshift/autoscaling.md`, `banking-genai-engineering-academy/kubernetes-openshift/production-incidents.md`

---

### Q8: 🟡 Explain Kubernetes RBAC in detail. How would you implement least-privilege access for a GenAI development team working across dev, staging, and prod namespaces?

**Strong Answer:**

RBAC (Role-Based Access Control) controls **who** can perform **what actions** on **which resources** in a Kubernetes cluster. It has four core objects:

- **Role**: Namespace-scoped permissions (e.g., "can read pods in banking-genai-dev")
- **ClusterRole**: Cluster-wide permissions (e.g., "can view all nodes")
- **RoleBinding**: Binds a Role to a User, Group, or ServiceAccount within a namespace
- **ClusterRoleBinding**: Binds a ClusterRole to a User, Group, or ServiceAccount cluster-wide

Each rule specifies `apiGroups`, `resources`, and `verbs` (get, list, watch, create, update, patch, delete).

For a GenAI development team with least-privilege access across environments:

```yaml
# Dev namespace: developers can do most things except delete
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: genai-developer
  namespace: banking-genai-dev
rules:
  - apiGroups: [""]
    resources: ["pods", "pods/log", "services", "configmaps"]
    verbs: ["get", "list", "watch", "create", "update", "patch"]
  - apiGroups: ["apps"]
    resources: ["deployments", "replicasets"]
    verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
  - apiGroups: [""]
    resources: ["secrets"]
    verbs: ["get", "list"]
    resourceNames: ["genai-api-dev"]  # Only specific secrets
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: genai-developer-dev-binding
  namespace: banking-genai-dev
subjects:
  - kind: Group
    name: genai-team
    apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: genai-developer
  apiGroup: rbac.authorization.k8s.io
---
# Prod namespace: developers can only READ (no write access)
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: genai-viewer
  namespace: banking-genai-prod
rules:
  - apiGroups: [""]
    resources: ["pods", "pods/log", "services"]
    verbs: ["get", "list", "watch"]
  - apiGroups: ["apps"]
    resources: ["deployments"]
    verbs: ["get", "list", "watch"]
```

**Least-privilege principles**:

1. **Never use the default ServiceAccount** for application Pods. Each application should have its own ServiceAccount with minimal permissions.
2. **Restrict secret access by name** using `resourceNames` — don't grant blanket secret read access.
3. **No wildcard verbs or resources** except for cluster-admin roles.
4. **Disable ServiceAccount token automount** when not needed: `automountServiceAccountToken: false`.
5. **Group-based access** rather than individual users — use AD/LDAP groups mapped through your identity provider.
6. **Production namespace**: developers get read-only access. Only CI/CD pipelines (via their own ServiceAccounts) can deploy to prod. This satisfies banking audit requirements.

**Auditing**: With audit logging enabled at the API server level, every RBAC-evaluated request is logged. This is mandatory for banking compliance (SOX, PCI DSS). You can query audit logs to answer "who accessed secret X at time Y?"

**Key Points to Hit:**
- [ ] Role = namespace-scoped; ClusterRole = cluster-wide
- [ ] RoleBinding binds subjects to roles; resourceNames restrict access to specific resources
- [ ] Separate roles per environment (dev=read+write, prod=read-only)
- [ ] Default ServiceAccount should never be used by application pods
- [ ] Audit logging required for banking compliance
- [ ] No wildcards, group-based access, token automount disabled

**Follow-Up Questions:**
1. What happens if a Role and a ClusterRole grant conflicting permissions to the same user?
2. How does ServiceAccount token projection work, and why is it more secure than the legacy token?
3. How would you implement just-in-time (JIT) access for emergency production debugging?

**Source:** `banking-genai-engineering-academy/kubernetes-openshift/rbac.md`, `banking-genai-engineering-academy/kubernetes-openshift/secrets.md`

---

### Q9: 🟡 What are Kubernetes Network Policies? How would you implement a zero-trust networking model for a banking GenAI platform running across multiple namespaces?

**Strong Answer:**

Network Policies are Kubernetes resources that control **which Pods can communicate with which other Pods**. By default, **all Pods can communicate with all other Pods** — there is no network isolation. Network Policies are the only way to enforce network segmentation in Kubernetes.

A zero-trust model means: **deny everything by default, then explicitly allow only what is needed**.

**Step 1: Default deny all ingress traffic in each namespace**:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-ingress
  namespace: banking-genai
spec:
  podSelector: {}  # Selects all pods
  policyTypes:
    - Ingress
```

**Step 2: Explicitly allow required communication paths**:

```yaml
# Allow ingress to GenAI API only from the ingress controller namespace
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-ingress-to-genai-api
  namespace: banking-genai
spec:
  podSelector:
    matchLabels:
      app: genai-api
  policyTypes:
    - Ingress
  ingress:
    - from:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: ingress-nginx
      ports:
        - protocol: TCP
          port: 8080
---
# Allow GenAI API to reach PostgreSQL in the banking-data namespace
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-genai-to-postgres
  namespace: banking-data
spec:
  podSelector:
    matchLabels:
      app: postgresql
  policyTypes:
    - Ingress
  ingress:
    - from:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: banking-genai
          podSelector:
            matchLabels:
              app: genai-api
      ports:
        - protocol: TCP
          port: 5432
---
# Egress: restrict outbound traffic from GenAI API
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: genai-api-egress
  namespace: banking-genai
spec:
  podSelector:
    matchLabels:
      app: genai-api
  policyTypes:
    - Egress
  egress:
    # DNS is always required
    - to: []
      ports:
        - protocol: UDP
          port: 53
        - protocol: TCP
          port: 53
    # PostgreSQL access
    - to:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: banking-data
          podSelector:
            matchLabels:
              app: postgresql
      ports:
        - protocol: TCP
          port: 5432
    # Redis access (same namespace)
    - to:
        - podSelector:
            matchLabels:
              app: redis
      ports:
        - protocol: TCP
          port: 6379
```

**Critical details for banking**:

1. **DNS must always be allowed** in egress policies — without it, service discovery breaks and Pods cannot resolve any DNS names.
2. **Inter-namespace policies** require both `namespaceSelector` and `podSelector` in the same `from` entry to AND the conditions (not OR). This ensures only specific Pods from specific namespaces can connect.
3. **CNI plugin requirement**: Network Policies require a CNI that supports them — Calico or Cilium. The default CNI on OpenShift (OVN-Kubernetes) supports them natively.
4. **Common incident**: A network policy update accidentally blocking DNS or database egress causes immediate service-wide failures. The fix is to always test policies in staging first and include both DNS and database rules. Always have a rollback-ready policy.

**Key Points to Hit:**
- [ ] Default behavior: all-to-all communication (no isolation)
- [ ] Zero-trust = default-deny + explicit allow rules
- [ ] DNS egress must always be allowed (port 53 UDP + TCP)
- [ ] Inter-namespace policies use namespaceSelector + podSelector (AND logic)
- [ ] Requires compatible CNI (Calico, Cilium, or OVN-Kubernetes on OpenShift)

**Follow-Up Questions:**
1. What happens if you apply a default-deny policy but forget to add the allow rules?
2. How do you debug network connectivity when Network Policies are in effect?
3. How do Network Policies differ from Service Mesh (Istio) authorization policies?

**Source:** `banking-genai-engineering-academy/kubernetes-openshift/network-policies.md`, `banking-genai-engineering-academy/kubernetes-openshift/production-incidents.md`

---

### Q10: 🟡 What are OpenShift Routes? How do they differ from Kubernetes Ingress, and what are the three TLS termination modes?

**Strong Answer:**

OpenShift Routes are OpenShift's native mechanism for exposing Services externally. They serve a similar purpose to Kubernetes Ingress but are built into OpenShift (powered by HAProxy) and offer features that standard Ingress requires third-party controllers for.

**Key differences from Ingress**:

| Feature | OpenShift Route | K8s Ingress |
|---------|----------------|-------------|
| Built-in | Yes (no extra controller needed) | Requires NGINX, Traefik, etc. |
| TLS Modes | edge, passthrough, reencrypt | Standard TLS only |
| Weighted routing | Native (for canary/blue-green) | Requires annotations or Istio |
| Rate limiting | Native HAProxy annotations | Controller-dependent |
| Multiple paths | Single path per Route | Multiple paths per Ingress rule |

**Three TLS Termination Modes**:

1. **Edge Termination** (most common): TLS is terminated at the OpenShift router (HAProxy). Traffic between the router and the Pod is unencrypted HTTP. This is simplest and sufficient when the internal cluster network is trusted.

```yaml
apiVersion: route.openshift.io/v1
kind: Route
metadata:
  name: genai-api
spec:
  host: genai-api.bank.com
  to:
    kind: Service
    name: genai-api
  port:
    targetPort: 8080
  tls:
    termination: edge
    insecureEdgeTerminationPolicy: Redirect  # HTTP -> HTTPS
```

2. **Passthrough Termination**: TLS is passed through the router directly to the Pod. The router does not decrypt the traffic. The Pod must have its own TLS certificate. This is used when end-to-end encryption is required (e.g., PCI DSS compliance mandates encryption at every hop).

```yaml
tls:
  termination: passthrough
```

3. **Re-encrypt Termination**: The router terminates the external TLS, then re-encrypts the traffic with a separate TLS connection to the backend Pod. This provides both external TLS termination and internal encryption. The Route must include the `destinationCACertificate` to verify the backend.

```yaml
tls:
  termination: reencrypt
  destinationCACertificate: |
    -----BEGIN CERTIFICATE-----
    ...
    -----END CERTIFICATE-----
```

**For banking GenAI APIs**, edge termination is typically sufficient for internal-facing endpoints. Passthrough or re-encrypt is required for externally-facing APIs that handle PII (Personally Identifiable Information) or payment data, where compliance mandates end-to-end encryption.

Routes also support built-in rate limiting via HAProxy annotations:
```yaml
annotations:
  haproxy.router.openshift.io/rate-limit-requests: "100"
  haproxy.router.openshift.io/rate-limit-seconds: "60"
```

**Key Points to Hit:**
- [ ] Routes are OpenShift-native; Ingress requires separate controller
- [ ] Edge = TLS at router; Passthrough = TLS to Pod; Re-encrypt = TLS at router + TLS to Pod
- [ ] Banking compliance may require passthrough or re-encrypt for PII endpoints
- [ ] Routes support weighted traffic natively for canary deployments
- [ ] Rate limiting via HAProxy annotations

**Follow-Up Questions:**
1. How would you implement a canary deployment using OpenShift Routes?
2. What happens to the Route if the backing Service has no endpoints?
3. Can you use both Routes and Ingress in the same OpenShift cluster?

**Source:** `banking-genai-engineering-academy/kubernetes-openshift/openshift-routes.md`, `banking-genai-engineering-academy/kubernetes-openshift/ingress.md`

---

### Q11: 🟡 What are OpenShift Operators? Explain the difference between using an existing Operator and building a custom one with the Operator SDK. When would you choose each approach?

**Strong Answer:**

Operators are Kubernetes extensions that automate the management of complex applications. They encode operational knowledge (provisioning, scaling, backups, upgrades, recovery) into software that runs as a controller inside the cluster. An Operator watches a **Custom Resource Definition (CRD)** and reconciles the actual state to match the desired state.

**Using an existing Operator** (via OperatorHub/OLM):

Operators for common software (PostgreSQL via CrunchyData, Redis, Kafka via Strimzi, Elasticsearch) are available from OperatorHub. You install them via a `Subscription` and then create the application's CR:

```yaml
# Install PostgreSQL Operator
apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
metadata:
  name: CrunchyPostgres
  namespace: openshift-operators
spec:
  channel: v5
  name: crunchy-postgres
  source: operatorhubio-catalog
  sourceNamespace: olm
---
# Create PostgreSQL cluster (the Operator handles everything)
apiVersion: postgres-operator.crunchydata.com/v1beta1
kind: PostgresCluster
metadata:
  name: banking-postgres
  namespace: banking-data
spec:
  postgresVersion: 15
  instances:
    - name: instance1
      replicas: 2
      dataVolumeClaimSpec:
        accessModes: ["ReadWriteOnce"]
        resources:
          requests:
            storage: 100Gi
  backups:
    pgbackrest:
      repos:
        - name: repo1
          volume:
            volumeClaimSpec:
              resources:
                requests:
                  storage: 50Gi
```

The Operator handles provisioning, replication, backups, failover, and version upgrades automatically. This is the preferred approach for infrastructure components.

**Building a custom Operator with the Operator SDK**:

When no existing Operator covers your application's operational complexity, you build one. The Operator SDK (based on controller-runtime) lets you write a Go-based controller that watches your CRD and manages child resources (Deployments, Services, ConfigMaps).

```go
// Simplified Operator reconcile loop
func (r *GenAIServiceReconciler) Reconcile(ctx context.Context, req ctrl.Request) (ctrl.Result, error) {
    // 1. Fetch the custom resource
    service := &genai.GenAIService{}
    r.Get(ctx, req.NamespacedName, service)

    // 2. Create/update managed Deployment
    deployment := r.deploymentForGenAI(service)
    r.CreateOrUpdate(ctx, deployment)

    // 3. Create/update managed Service
    svc := r.serviceForGenAI(service)
    r.CreateOrUpdate(ctx, svc)

    return ctrl.Result{}, nil
}
```

**When to choose each**:

| Scenario | Approach |
|----------|----------|
| PostgreSQL, Redis, Kafka, monitoring | Use existing Operator from OperatorHub |
| Custom GenAI model serving platform with complex lifecycle | Build custom Operator |
| Simple stateless application | Use Helm chart (not an Operator) |
| Application with automated backup/restore logic | Build custom Operator |

**OLM (Operator Lifecycle Manager)** manages Operator installation, upgrades, and dependency resolution. It ensures Operators are installed in the correct order and prevents incompatible versions.

For a banking platform, the guideline is: **use existing Operators for infrastructure, build custom Operators only when automation complexity justifies the engineering cost, and use Helm for everything else**. Custom Operators require ongoing maintenance, testing, and documentation — they should solve real operational pain points, not replace simple Helm charts.

**Key Points to Hit:**
- [ ] Operators encode operational knowledge as controllers watching CRDs
- [ ] Existing Operators via OperatorHub for databases, caches, message queues
- [ ] Custom Operators via SDK for complex proprietary applications
- [ ] OLM manages Operator lifecycle (install, upgrade, dependencies)
- [ ] Rule of thumb: Operator > Helm only when lifecycle complexity justifies cost
- [ ] Helm is sufficient for simple stateless deployments

**Follow-Up Questions:**
1. What is the reconcile loop in an Operator, and how does it handle failures?
2. How does an Operator handle version upgrades of the managed application?
3. What is the difference between an Operator and a Helm chart with `helm upgrade`?

**Source:** `banking-genai-engineering-academy/kubernetes-openshift/openshift-operators.md`, `banking-genai-engineering-academy/kubernetes-openshift/helm.md`

---

### Q12: 🟡 What are Security Context Constraints (SCC) in OpenShift? How do they differ from Kubernetes Pod Security Standards, and what SCC should a banking GenAI application use?

**Strong Answer:**

Security Context Constraints (SCCs) are OpenShift's mechanism for controlling what permissions Pods can have at runtime. They are more granular and feature-rich than Kubernetes Pod Security Standards (PSS), which offer three levels: privileged, baseline, and restricted.

**How SCCs work**: When a Pod is created, OpenShift evaluates all SCCs against the Pod's `securityContext` and assigns the highest-priority SCC that the Pod's ServiceAccount is authorized to use. If no SCC matches, the Pod is rejected.

**Common SCCs**:

| SCC | Privileged | RunAs | Capabilities | Use Case |
|-----|-----------|-------|-------------|----------|
| privileged | Yes | Any | Any | System daemons (never for apps) |
| anyuid | No | Any UID | Default | Legacy apps requiring specific UID |
| nonroot | No | Non-root | Default | Modern containerized apps |
| restricted | No | Non-root | Drop ALL | Default (most secure) |

**For a banking GenAI application, the `restricted` SCC is the target**:

```yaml
apiVersion: security.openshift.io/v1
kind: SecurityContextConstraints
metadata:
  name: genai-restricted
allowPrivilegedContainer: false
runAsUser:
  type: MustRunAsNonRoot
requiredDropCapabilities:
  - ALL
readOnlyRootFilesystem: true
allowHostDirVolumePlugin: false
allowHostIPC: false
allowHostNetwork: false
allowHostPID: false
allowPrivilegeEscalation: false
volumes:
  - configMap
  - downwardAPI
  - emptyDir
  - persistentVolumeClaim
  - projected
  - secret
```

This SCC enforces:
- **No privileged containers**: Cannot access host resources
- **Must run as non-root**: Mitigates container escape attacks
- **Drop ALL Linux capabilities**: Minimal privilege model
- **Read-only root filesystem**: Prevents runtime code injection
- **No host access**: Cannot use hostNetwork, hostPID, hostIPC
- **Restricted volume types**: Only safe volume types allowed

**Granting SCC to a ServiceAccount**:
```bash
oc adm policy add-scc-to-user restricted -z genai-api-sa -n banking-genai
```

**Differences from Kubernetes PSS**:
- PSS are namespace-level labels (`pod-security.kubernetes.io/enforce: restricted`) that reject non-compliant Pods at admission time.
- SCCs are cluster-level resources evaluated per-Pod at creation time, with priority-based selection and ServiceAccount binding.
- SCCs offer finer control over capabilities, volume types, host access, and UID ranges.
- PSS are simpler but less configurable — they are a coarse "all or nothing" per namespace.

In banking environments, SCC configuration is a regulatory requirement. Auditors will check that no application Pods run with `privileged` or `anyuid` SCCs, that `readOnlyRootFilesystem` is enforced, and that SCC assignments are reviewed periodically.

**Key Points to Hit:**
- [ ] SCCs are OpenShift-specific; more granular than K8s Pod Security Standards
- [ ] restricted SCC is the target for banking applications
- [ ] Enforces: non-root, no privileged, drop ALL caps, read-only root FS
- [ ] Assigned via ServiceAccount with priority-based evaluation
- [ ] Banking auditors check SCC compliance as part of security reviews

**Follow-Up Questions:**
1. How do you debug a Pod that fails to start due to SCC violations?
2. Can you create a custom SCC that is more restrictive than `restricted`?
3. How do SCCs interact with init containers that need privileged access?

**Source:** `banking-genai-engineering-academy/kubernetes-openshift/openshift-scc.md`, `banking-genai-engineering-academy/kubernetes-openshift/secure-deployment-patterns.md`

---

### Q13: 🟡 How do you schedule GPU workloads in Kubernetes? Explain device plugins, node selectors, taints/tolerations, and MIG for multi-tenant GPU usage in a GenAI platform.

**Strong Answer:**

GPU scheduling in Kubernetes requires three components working together: the **NVIDIA device plugin**, **node labeling**, and **resource requests/limits**.

**1. NVIDIA Device Plugin**: This DaemonSet runs on GPU nodes and advertises GPU capacity to the Kubernetes scheduler. It detects the GPUs, their type (A100, V100, T4), and reports them as extended resources (`nvidia.com/gpu: 1`). Without the device plugin, Kubernetes has no awareness of GPUs.

**2. Node Selectors and Taints**: GPU nodes are expensive and should be used exclusively for GPU workloads. We taint them to prevent non-GPU Pods from being scheduled there:

```
Node taint: nvidia.com/gpu=NoSchedule
```

GPU workloads add a matching toleration:

```yaml
tolerations:
  - key: nvidia.com/gpu
    operator: Exists
    effect: NoSchedule
```

And a node selector ensures the Pod lands on a GPU node:

```yaml
nodeSelector:
  nvidia.com/gpu.present: "true"
```

**3. GPU Resource Requests**: The container must request GPU resources:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: embedding-service
spec:
  replicas: 2
  selector:
    matchLabels:
      app: embedding-service
  template:
    spec:
      nodeSelector:
        nvidia.com/gpu.present: "true"
      tolerations:
        - key: nvidia.com/gpu
          operator: Exists
          effect: NoSchedule
      containers:
        - name: embedding
          image: quay.io/banking/embedding-service:1.0.0
          resources:
            limits:
              nvidia.com/gpu: 1
            requests:
              cpu: "2"
              memory: 8Gi
```

**4. MIG (Multi-Instance GPU)**: NVIDIA A100 GPUs support MIG, which partitions a single physical GPU into up to 7 isolated instances. This is critical for multi-tenant GenAI platforms where different models or teams share GPU resources. Instead of dedicating an entire A100 to one Pod, MIG allows:

```yaml
resources:
  limits:
    nvidia.com/mig-1g.5gb: 1  # 1/7th of an A100 with 5GB memory
```

This dramatically improves GPU utilization. Without MIG, a Pod using only 20% of a GPU still blocks the entire GPU. With MIG, up to 7 teams can share one A100 with hardware-level isolation.

**Banking-specific considerations**:
- GPU nodes should be in dedicated node pools with autoscaling based on inference queue depth.
- MIG profiles should be assigned based on model requirements (large LLMs need full GPU, small embedding models can use 1g.5gb slices).
- GPU memory monitoring via DCGM exporter is essential — GPU OOM is harder to debug than CPU OOM.
- Readiness probes must account for model loading time (can be 30-60 seconds for large models).

**Key Points to Hit:**
- [ ] NVIDIA device plugin advertises GPU capacity to the scheduler
- [ ] Taints prevent non-GPU workloads on GPU nodes; tolerations allow GPU workloads
- [ ] Node selectors ensure Pods schedule to GPU-labeled nodes
- [ ] MIG partitions A100 into up to 7 isolated instances for multi-tenancy
- [ ] GPU memory monitoring via DCGM exporter prevents GPU OOM

**Follow-Up Questions:**
1. How would you autoscale GPU workloads? What metrics would you use?
2. What happens if the NVIDIA device plugin crashes on a GPU node?
3. How do you share a GPU between multiple Pods (e.g., time-slicing)?

**Source:** `banking-genai-engineering-academy/kubernetes-openshift/gpu-workloads.md`, `banking-genai-engineering-academy/kubernetes-openshift/autoscaling.md`

---

### Q14: 🟡 Describe the difference between blue-green deployments and canary releases. For a banking GenAI platform handling customer-facing transactions, which would you choose and why?

**Strong Answer:**

Both blue-green and canary are deployment strategies that enable zero-downtime releases, but they differ fundamentally in **how traffic is shifted** and **how risk is managed**.

**Blue-Green Deployment**:
Two identical environments run simultaneously. Blue (current version) serves production traffic. Green (new version) is deployed but receives no traffic. When green passes health checks, traffic is switched **instantly** by changing the Service selector or Route weight.

```bash
# Switch traffic from blue to green
oc patch service genai-api-active -p '{"spec":{"selector":{"active":"false"}}}' -n banking-genai

# Instant rollback: switch back
oc patch service genai-api-active -p '{"spec":{"selector":{"active":"true"}}}' -n banking-genai
```

**Advantages**: Instant switch, instant rollback, simple to understand.
**Disadvantages**: 2x resource cost (both environments run simultaneously), all traffic goes to the new version at once (no gradual validation).

**Canary Release**:
A small percentage of traffic (5-10%) is routed to the new version. If metrics (error rate, latency, business KPIs) look good, traffic is gradually increased: 10% -> 25% -> 50% -> 100%. If metrics degrade, the canary is rolled back.

```yaml
# Argo Rollouts automated canary
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
      - setWeight: 100
```

**Advantages**: Gradual risk exposure, metrics-driven promotion, lower resource cost (canary can start with 1 replica).
**Disadvantages**: More complex setup, slower rollout, requires metrics infrastructure.

**For a banking GenAI platform**:

I would recommend **canary releases** for the following reasons:

1. **GenAI quality validation**: LLM output quality can degrade subtly with model version changes. A canary allows monitoring of response quality, hallucination rates, and latency with real customer traffic before full commitment.
2. **Regulatory risk mitigation**: In banking, a deployment that produces incorrect financial advice or incorrect account information can have compliance implications. Canary catches this with limited blast radius.
3. **Resource efficiency**: Running two full production environments (blue-green) for GPU-heavy GenAI workloads is prohibitively expensive. Canary uses minimal additional resources.
4. **Automated rollback**: With Argo Rollouts and Prometheus-based analysis, the canary can automatically abort if error rate exceeds 1% or p99 latency exceeds 2 seconds.

Blue-green is appropriate when you need **instant rollback capability** for critical infrastructure (e.g., authentication service, API gateway) where any degraded traffic is unacceptable. For application-level GenAI services, canary is the better default.

**Key Points to Hit:**
- [ ] Blue-green = instant switch, 2x cost, all-or-nothing risk
- [ ] Canary = gradual traffic shift (10% -> 25% -> 50% -> 100%), metrics-driven
- [ ] Canary preferred for GenAI: validates model quality with real traffic
- [ ] Blue-green preferred for infrastructure where instant rollback is critical
- [ ] Automated canary with Argo Rollouts + Prometheus analysis for banking compliance

**Follow-Up Questions:**
1. How do you handle database schema migrations with blue-green deployments?
2. What metrics would you monitor during a canary release of a GenAI service?
3. How would you implement header-based canary routing (internal testers only)?

**Source:** `banking-genai-engineering-academy/kubernetes-openshift/blue-green-deployments.md`, `banking-genai-engineering-academy/kubernetes-openshift/canary-releases.md`, `banking-genai-engineering-academy/kubernetes-openshift/deployments.md`

---

### Q15: 🟡 A Pod is in CrashLoopBackOff. Walk me through your systematic debugging process. What are the most common causes in a GenAI application?

**Strong Answer:**

CrashLoopBackOff means the container started, crashed, was restarted by Kubernetes (restartPolicy: Always), crashed again, and Kubernetes is now applying exponential backoff (10s, 20s, 40s, up to 5 minutes) between restarts. Here is my systematic approach:

**Step 1: Check Pod status and events**
```bash
kubectl get pods -n banking-genai
kubectl describe pod genai-api-abc123 -n banking-genai
```

The `describe` output shows the **Events** section at the bottom, which reveals why the container is crashing. Look for: `Back-off restarting failed container`, OOMKilled, image pull errors, or readiness probe failures.

**Step 2: Check logs — current and previous**
```bash
kubectl logs genai-api-abc123 -n banking-genai           # Current attempt
kubectl logs genai-api-abc123 -n banking-genai --previous # Previous crash
```

The `--previous` flag is critical — it shows logs from the crashed instance. This is often where the actual error lives. Common GenAI application errors:
- Missing environment variable (e.g., `OPENAI_API_KEY` not set)
- Database connection failure (PostgreSQL not reachable, wrong credentials)
- Model loading failure (model file not found, GPU not available)
- Port conflict (another process already on port 8080)
- Python import error (missing dependency in container image)

**Step 3: Check resource usage**
```bash
kubectl describe pod genai-api-abc123 -n banking-genai | grep -A5 "Last State"
kubectl top pod genai-api-abc123 -n banking-genai
```

If `Last State` shows `OOMKilled` with exit code 137, the memory limit is too low. For a GenAI API that loads ML models, 1 GiB is often insufficient — 2-4 GiB is typical.

**Step 4: Exec into the container (if it survives long enough)**
```bash
kubectl exec -it genai-api-abc123 -n banking-genai -- /bin/sh
```

If the container crashes too quickly for exec, use an ephemeral debugging container:
```bash
kubectl debug -it genai-api-abc123 -n banking-genai --image=busybox --target=api
```

**Step 5: Verify dependencies**
```bash
# Check if database is reachable
kubectl exec genai-api-abc123 -n banking-genai -- curl -v postgres.banking-data:5432

# Check if secrets are mounted
kubectl exec genai-api-abc123 -n banking-genai -- cat /app/secrets/username

# Check environment variables
kubectl exec genai-api-abc123 -n banking-genai -- env
```

**Most common CrashLoopBackOff causes in GenAI applications**:

| Cause | Symptom | Fix |
|-------|---------|-----|
| Missing env var / secret | `KeyError` in logs | Check ConfigMap/Secret exists and keys match |
| OOMKilled | Exit code 137 | Increase memory limit |
| Database not reachable | Connection refused | Check Service, NetworkPolicy, database Pod status |
| Model file not found | `FileNotFoundError` | Verify volume mount path |
| Port already in use | `Address already in use` | Check for zombie processes |
| Readiness probe too aggressive | Killed before startup | Add startup probe with higher threshold |

In banking environments, CrashLoopBackOff during a deployment is usually caused by a configuration drift (new version expects a ConfigMap key that doesn't exist yet) or a secret rotation that changed the database credentials before the application was updated.

**Key Points to Hit:**
- [ ] Systematic flow: describe → logs (--previous) → resource check → exec → dependency check
- [ ] --previous flag shows logs from the crashed container
- [ ] OOMKilled (exit 137) = memory limit too low
- [ ] Ephemeral containers for Pods that crash too quickly
- [ ] Common GenAI causes: missing env vars, OOM, database unreachable, model not found
- [ ] Banking context: config drift and secret rotation are frequent culprits

**Follow-Up Questions:**
1. How do you debug a Pod that crashes before you can exec into it?
2. What is the exponential backoff pattern for CrashLoopBackOff restarts?
3. How would you set up automated alerting for CrashLoopBackOff in production?

**Source:** `banking-genai-engineering-academy/kubernetes-openshift/debugging-pods.md`, `banking-genai-engineering-academy/kubernetes-openshift/pods.md`, `banking-genai-engineering-academy/kubernetes-openshift/production-incidents.md`

---

### Q16: 🔴 Your GenAI API pods are experiencing OOMKilled events during traffic spikes. Describe the complete incident response, root cause analysis, and long-term prevention strategy.

**Strong Answer:**

**Incident Timeline (realistic scenario)**:

```
14:00 - Marketing campaign drives 3x traffic increase to GenAI API
14:03 - HPA begins scaling up (CPU utilization hits 70% threshold)
14:05 - New pods start, but memory usage per pod spikes to 950Mi of 1Gi limit
14:07 - First pods OOMKilled (exit 137)
14:08 - HPA scales further to compensate, but new pods also OOMKill
14:10 - Service degraded: 50% of requests return 503
14:12 - On-call alerted via PagerDuty
14:15 - Emergency action: kubectl patch deployment to increase memory limit
14:18 - New pods with 2Gi limit start successfully
14:25 - HPA stabilizes, all pods healthy
14:30 - Service fully recovered
```

**Immediate Response**:

1. **Identify the issue**: `kubectl describe pod <pod> -n banking-genai | grep -A5 "Last State"` confirms `Reason: OOMKilled, Exit Code: 137`.
2. **Emergency fix**: Increase memory limit immediately:
```bash
kubectl patch deployment genai-api -n banking-genai -p \
  '{"spec":{"template":{"spec":{"containers":[{"name":"api","resources":{"limits":{"memory":"2Gi"}}}]}}}}'
```
3. **Verify recovery**: `kubectl get pods -n banking-genai -w` and check error rates in monitoring dashboard.

**Root Cause Analysis**:

The memory limit of 1 GiB was set based on baseline traffic patterns, not peak load. GenAI APIs have memory characteristics that scale with concurrency — each request buffers the LLM response (potentially 4-8 KB per token × thousands of tokens), and concurrent requests multiply this. During the 3x traffic spike, concurrent request buffering exceeded the limit.

Contributing factors:
- No load testing was performed before the marketing campaign launch.
- HPA scaled Pods horizontally but did not account for per-Pod memory increase under load.
- No memory-based HPA metric was configured (only CPU was used).
- No Pod Disruption Budget meant Kubernetes could terminate Pods during the crisis.

**Long-Term Prevention Strategy**:

1. **Right-size resources**: Use VPA in recommendation mode for 2 weeks to observe actual memory usage across traffic patterns. Set requests at P90, limits at 120% of peak observed.

2. **Dual-metric HPA**: Scale on both CPU and memory:
```yaml
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
        averageUtilization: 75
```

3. **Pod Disruption Budget**:
```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: genai-api-pdb
spec:
  minAvailable: 2
  selector:
    matchLabels:
      app: genai-api
```

4. **Load testing**: Implement regular load testing (e.g., k6 or Locust) at 2x expected peak before any major release.

5. **Alerting**: Set Prometheus alerts for `kube_pod_container_status_last_terminated_reason{reason="OOMKilled"}` and memory utilization > 85%.

6. **Application-level backpressure**: Implement request queuing with maximum concurrency limits in the GenAI API to prevent memory explosion under extreme load.

**Key Points to Hit:**
- [ ] OOMKilled = exit 137; confirm with kubectl describe
- [ ] Emergency patch to increase memory limit as immediate fix
- [ ] Root cause: memory scales with concurrency, not just Pod count
- [ ] VPA recommendation mode to right-size; dual-metric HPA
- [ ] Pod Disruption Budget ensures minimum availability
- [ ] Load testing and alerting as preventive measures

**Follow-Up Questions:**
1. How does HPA interact with OOMKilled pods? Does it account for them?
2. What is a Pod Disruption Budget and how does it differ from resource limits?
3. How would you implement application-level backpressure for a streaming LLM response API?

**Source:** `banking-genai-engineering-academy/kubernetes-openshift/production-incidents.md`, `banking-genai-engineering-academy/kubernetes-openshift/autoscaling.md`, `banking-genai-engineering-academy/kubernetes-openshift/pods.md`

---

### Q17: 🔴 How would you design an automated canary release pipeline for a GenAI model serving application? Include traffic splitting, metric-based promotion, and rollback automation.

**Strong Answer:**

Designing an automated canary pipeline for a GenAI model serving application requires integrating deployment orchestration, traffic splitting, metrics analysis, and automated decision-making. Here is the complete architecture:

**Traffic Splitting with OpenShift Routes**:

OpenShift Routes natively support weighted traffic splitting, making canary deployments straightforward:

```yaml
# Stable route (receives 90% traffic initially)
apiVersion: route.openshift.io/v1
kind: Route
metadata:
  name: genai-api-stable
spec:
  host: genai-api.bank.com
  to:
    kind: Service
    name: genai-api-stable
    weight: 90
  port:
    targetPort: 8080
---
# Canary route (receives 10% traffic initially)
apiVersion: route.openshift.io/v1
kind: Route
metadata:
  name: genai-api-canary
spec:
  host: genai-api.bank.com
  to:
    kind: Service
    name: genai-api-canary
    weight: 10
  port:
    targetPort: 8080
```

**Automated Promotion with Argo Rollouts**:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Rollout
metadata:
  name: genai-api
spec:
  replicas: 3
  strategy:
    canary:
      steps:
        # Step 1: 10% traffic, wait 5 minutes
        - setWeight: 10
        - pause: {duration: 5m}

        # Step 2: Analyze error rate
        - analysis:
            templates:
              - templateName: genai-canary-analysis

        # Step 3: 25% traffic, wait 5 minutes
        - setWeight: 25
        - pause: {duration: 5m}

        # Step 4: Analyze again
        - analysis:
            templates:
              - templateName: genai-canary-analysis

        # Step 5: 50% traffic, wait 10 minutes
        - setWeight: 50
        - pause: {duration: 10m}

        # Step 6: 100% traffic, promote to stable
        - setWeight: 100

        # Step 7: Final verification, then complete
        - pause: {duration: 5m}
```

**GenAI-Specific Analysis Template**:

For a GenAI application, the analysis must check not just infrastructure metrics but also model quality metrics:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: AnalysisTemplate
metadata:
  name: genai-canary-analysis
spec:
  metrics:
    # Infrastructure: error rate
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

    # Infrastructure: p99 latency
    - name: p99-latency
      interval: 1m
      successCondition: result[0] < 3000
      failureLimit: 2
      provider:
        prometheus:
          address: http://prometheus.monitoring:9090
          query: |
            histogram_quantile(0.99, sum(rate(http_request_duration_seconds_bucket{service="genai-api"}[1m])) by (le))

    # GenAI-specific: average response tokens per request (detect model degradation)
    - name: response-quality
      interval: 2m
      successCondition: result[0] > 50
      failureLimit: 1
      provider:
        prometheus:
          address: http://prometheus.monitoring:9090
          query: |
            avg(genai_response_tokens{version="canary"})

    # GenAI-specific: model inference time
    - name: inference-time
      interval: 1m
      successCondition: result[0] < 5000
      failureLimit: 2
      provider:
        prometheus:
          address: http://prometheus.monitoring:9090
          query: |
            avg(genai_inference_duration_ms{version="canary"})
```

**Rollback Automation**: If any metric fails its success condition more times than `failureLimit` allows, Argo Rollouts automatically aborts the canary, scales down the canary deployment, and returns 100% traffic to the stable version. No human intervention required.

**Tekton Pipeline Integration** (OpenShift Pipelines):

The entire process is triggered by a Tekton Pipeline on git merge to main:
1. Build new image → scan for vulnerabilities → push to registry
2. Deploy canary with 1 replica and 10% traffic weight
3. Argo Rollouts manages the progressive promotion
4. If all stages pass, update the stable deployment and remove canary
5. If any stage fails, automatic rollback + alert to the team

**Key Points to Hit:**
- [ ] OpenShift Routes support weighted traffic splitting natively
- [ ] Argo Rollouts manages progressive promotion with pause + analysis steps
- [ ] Analysis must include both infrastructure metrics (error rate, latency) and GenAI-specific metrics (response quality, inference time)
- [ ] Automated rollback on metric failure — no human intervention needed
- [ ] Tekton Pipeline orchestrates the full CI/CD flow

**Follow-Up Questions:**
1. How would you implement header-based canary routing (route internal testers to canary only)?
2. What if the canary passes infrastructure metrics but produces lower-quality LLM responses?
3. How do you handle database migrations during a canary deployment?

**Source:** `banking-genai-engineering-academy/kubernetes-openshift/canary-releases.md`, `banking-genai-engineering-academy/kubernetes-openshift/openshift-routes.md`, `banking-genai-engineering-academy/kubernetes-openshift/openshift-pipelines.md`

---

### Q18: 🔴 A node in your OpenShift cluster goes down. Walk me through the cascading effects on running workloads, how Kubernetes recovers, and what you would do as the on-call engineer.

**Strong Answer:**

When a node goes down, several cascading events occur in sequence. Understanding each phase is critical for effective incident response.

**Phase 1: Node becomes unreachable (0-40 seconds)**

The kubelet on the node stops sending heartbeats to the API server. The Node controller marks the node as `NotReady` after 40 seconds of missed heartbeats (controlled by `--node-monitor-grace-period`, default 40s). During this window, Pods on the node continue running (if the node is still partially functional), but the scheduler cannot place new Pods on it.

**Phase 2: Pod eviction begins (5 minutes)**

After 5 minutes (`--pod-eviction-timeout`, default 300s), the Node controller begins evicting Pods from the NotReady node. Pods are marked as `Terminating` but cannot complete graceful shutdown because the node is unreachable. Their status becomes `Unknown`.

**Phase 3: Replacement Pods are scheduled**

For Pods managed by Deployments, ReplicaSets, or StatefulSets, the controller notices that replicas are below the desired count and schedules new Pods on healthy nodes. The speed of this depends on:
- Whether the cluster has spare capacity on remaining nodes
- Whether Cluster Autoscaler can provision new nodes (takes 3-5 minutes in cloud environments)
- Pod scheduling constraints (node selectors, taints, affinity rules)

**Phase 4: Service traffic is rerouted**

kube-proxy removes the dead node's Pods from the Service endpoint list. Traffic is redistributed to healthy Pods. If the Deployment had 3 replicas and one node went down, the Service now has 2 active endpoints until replacement Pods are ready.

**On-call engineer response**:

```bash
# 1. Verify node status
oc get nodes
oc describe node <node-name>  # Check conditions, events

# 2. Check affected pods
oc get pods -n banking-genai -o wide | grep <node-name>
oc get pods --field-selector=status.phase=Unknown -A

# 3. Check if replacement pods are running
oc get deployment genai-api -n banking-genai
oc rollout status deployment/genai-api -n banking-genai

# 4. Check cluster capacity
oc describe nodes | grep -A3 "Allocated resources"

# 5. If Cluster Autoscaler is not provisioning nodes, investigate
oc get events -n openshift-machine-api --sort-by='.lastTimestamp'

# 6. If node is permanently dead, drain and delete it
oc drain <node-name> --ignore-daemonsets --delete-emptydir-data
oc delete node <node-name>
```

**Critical banking-specific considerations**:

1. **GPU node failure**: If a GPU node goes down, Pods requiring `nvidia.com/gpu` cannot be rescheduled to non-GPU nodes. The service will remain degraded until a new GPU node is provisioned, which can take 10-15 minutes. This is why GPU node pools should have at least 2 nodes.

2. **Pod Disruption Budgets**: If the cluster is already at capacity and a node goes down, PDBs may block the scheduler from creating replacement Pods on nodes that would violate the budget. In practice, this means you need spare capacity (at least 20% headroom) to survive node failures.

3. **StatefulSets**: Pods in StatefulSets (databases) have persistent volumes tied to specific nodes. If a node goes down, the replacement Pod must wait for the volume to detach from the dead node before attaching to the new node, which can take several minutes.

4. **etcd**: If the node runs etcd and it goes down, the entire cluster's control plane is affected. etcd should always run on dedicated, highly available nodes (3 or 5 members).

**Prevention**: Monitor node health with alerts on `kube_node_status_condition{condition="Ready",status="false"}`, ensure Cluster Autoscaler is properly configured, and run regular chaos engineering tests (e.g., randomly terminating nodes in staging) to validate recovery procedures.

**Key Points to Hit:**
- [ ] Node NotReady after 40s, pod eviction after 5 minutes
- [ ] Controllers schedule replacement Pods; Service reroutes traffic
- [ ] GPU nodes: replacement Pods cannot schedule to non-GPU nodes
- [ ] PDBs and spare capacity (20% headroom) required for graceful recovery
- [ ] StatefulSet volumes must detach before reattaching to new node
- [ ] etcd must run on dedicated HA nodes (3 or 5 members)

**Follow-Up Questions:**
1. What is the difference between a node being NotReady and a node being deleted?
2. How would you design a multi-AZ OpenShift cluster to survive zone-level failures?
3. What happens to in-flight requests when a node goes down mid-request?

**Source:** `banking-genai-engineering-academy/kubernetes-openshift/production-incidents.md`, `banking-genai-engineering-academy/kubernetes-openshift/gpu-workloads.md`, `banking-genai-engineering-academy/kubernetes-openshift/statefulsets.md`

---

### Q19: 🔴 You need to debug a network partition in a multi-namespace banking GenAI setup where the API can intermittently reach PostgreSQL. Walk me through your debugging methodology.

**Strong Answer:**

Intermittent network connectivity between namespaces is one of the most difficult issues to debug in Kubernetes because the symptoms are inconsistent — sometimes it works, sometimes it doesn't. Here is my systematic approach:

**Step 1: Confirm the symptom and scope**

```bash
# From the GenAI API pod, test connectivity
kubectl exec -it genai-api-abc123 -n banking-genai -- \
  curl -v postgres.banking-data.svc.cluster.local:5432

# Test DNS resolution
kubectl exec genai-api-abc123 -n banking-genai -- \
  nslookup postgres.banking-data.svc.cluster.local

# Test from multiple pods (is it isolated to one pod or all?)
kubectl exec genai-api-def345 -n banking-genai -- \
  curl -v postgres.banking-data.svc.cluster.local:5432
```

If some pods can connect and others cannot, this points to a **per-Pod routing issue** (possibly kube-proxy/iptables state) rather than a Network Policy issue (which would block all pods uniformly).

**Step 2: Check Network Policies**

```bash
# List all NetworkPolicies in both namespaces
kubectl get networkpolicies -n banking-genai
kubectl get networkpolicies -n banking-data

# Review the specific egress policy
kubectl get networkpolicy genai-api-egress -n banking-genai -o yaml
```

Common Network Policy misconfigurations causing intermittent connectivity:
- DNS egress rule missing or incomplete (UDP port 53 allowed but TCP port 53 is not, or vice versa)
- Namespace selector using stale labels that don't match the current namespace
- Port mismatch (policy allows 5432 but PostgreSQL is listening on a different port)
- The `from` field using `namespaceSelector` OR `podSelector` instead of AND (both must be in the same `from` entry)

**Step 3: Check kube-proxy and iptables**

```bash
# Check kube-proxy logs on the node
kubectl logs -n kube-system -l k8s-app=kube-proxy

# Check if endpoint slices are updated
kubectl get endpointslices -n banking-data
```

If kube-proxy's iptables rules are stale (e.g., after a Network Policy update), some nodes may have old rules while others have new ones, causing intermittent behavior. Restarting kube-proxy or triggering a sync can resolve this.

**Step 4: Check CNI plugin health**

On OpenShift with OVN-Kubernetes:
```bash
# Check OVN-Kubernetes pod logs
oc logs -n openshift-ovn-kubernetes -l app=ovnkube-node

# Check for dropped packets
oc exec -n openshift-ovn-kubernetes ovnkube-node-xxxxx -- ovn-nbctl show
```

**Step 5: Check for resource contention**

```bash
# Check if the PostgreSQL pod is under heavy load
kubectl top pod -n banking-data
kubectl describe pod postgres-0 -n banking-data
```

If PostgreSQL is CPU-throttled or its connection pool is exhausted, connections may timeout rather than fail immediately, which looks like a network issue but is actually an application resource issue.

**Step 6: Check for Service endpoint churn**

```bash
kubectl get endpoints postgres -n banking-data -w
```

If endpoints are flapping (pods being added/removed rapidly due to readiness probe failures), connections will intermittently fail as kube-proxy updates its routing rules.

**Most likely root causes in order of probability**:

1. **Network Policy misconfiguration** (most common): DNS or database port not in egress allow list.
2. **kube-proxy iptables staleness**: After a policy update, some nodes have stale rules.
3. **PostgreSQL connection pool exhaustion**: Not a network issue — the database accepts TCP connections but rejects them at the application layer.
4. **CNI bug**: OVN-Kubernetes or Calico has a known issue with inter-namespace routing under specific conditions.

**Banking context**: Network Policy changes must go through a change approval process and be tested in staging. The most common production incident is a security team tightening egress policies without including all required destinations. Always test policies with connectivity checks before applying to production.

**Key Points to Hit:**
- [ ] Test from multiple pods to determine if issue is per-Pod or namespace-wide
- [ ] DNS egress must allow both UDP and TCP port 53
- [ ] Network Policy namespaceSelector + podSelector must be in same from entry (AND logic)
- [ ] kube-proxy iptables staleness causes intermittent (not consistent) failures
- [ ] Connection pool exhaustion looks like network issues but is application-layer
- [ ] CNI plugin logs (OVN-Kubernetes on OpenShift) reveal dropped packets

**Follow-Up Questions:**
1. How would you test Network Policies in a CI pipeline before applying to production?
2. What is the difference between iptables and IPVS mode in kube-proxy?
3. How do you implement network connectivity monitoring between critical services?

**Source:** `banking-genai-engineering-academy/kubernetes-openshift/network-policies.md`, `banking-genai-engineering-academy/kubernetes-openshift/debugging-pods.md`, `banking-genai-engineering-academy/kubernetes-openshift/services.md`

---

### Q20: 🔴 Design a production-grade OpenShift deployment architecture for a GenAI banking platform that includes GPU inference, automated CI/CD, zero-trust networking, and disaster recovery. Walk me through every component and its justification.

**Strong Answer:**

This is an architecture question that tests whether you can synthesize all Kubernetes/OpenShift concepts into a cohesive production design. Here is the complete architecture:

**Cluster Topology**:

Two OpenShift clusters in an active-standby configuration across regions:
- **Primary Cluster** (us-east): Runs all GenAI workloads
- **DR Cluster** (us-west): Hot standby with replicated data, runs minimal workloads

A Global Load Balancer (GLB) routes traffic to the primary cluster and fails over to DR during cluster-level incidents.

**Namespace Architecture**:

```
openshift-cluster-1 (us-east):
├── banking-genai-prod
│   ├── genai-api (Deployment, 5 replicas, HPA min=5 max=20)
│   ├── embedding-service (Deployment on GPU nodes, 2 replicas, MIG-enabled)
│   ├── document-processor (Deployment, KEDA-scaled from Kafka lag)
│   └── redis-client (Deployment, connects to external Redis)
├── banking-data
│   ├── postgresql (CrunchyData Operator, 2 replicas, async replication to DR)
│   └── kafka (Strimzi Operator, 3 brokers)
├── monitoring
│   ├── prometheus (cluster metrics, GenAI custom metrics)
│   └── grafana (dashboards, alerting)
└── openshift-pipelines
    └── genai-api-pipeline (Tekton: build → test → scan → deploy → canary)
```

**Component Justifications**:

1. **GPU Inference**: Dedicated GPU node pool with A100 GPUs, MIG enabled for multi-tenancy. The embedding service uses `nvidia.com/mig-1g.5gb` slices (5 GB per instance), allowing 7 concurrent inference workloads per GPU. Taints (`nvidia.com/gpu=NoSchedule`) prevent non-GPU workloads. GPU-aware autoscaling via KEDA scales on inference queue depth (Prometheus metric).

2. **Zero-Trust Networking**: Default-deny NetworkPolicies in all namespaces. Explicit allow rules for: ingress controller to genai-api, genai-api to PostgreSQL (port 5432), genai-api to Redis (port 6379), all pods to DNS (port 53 UDP+TCP). Egress policies restrict outbound traffic. SCC `restricted` enforced for all application pods.

3. **CI/CD Pipeline** (OpenShift Pipelines / Tekton):
   - Git push triggers Pipeline
   - Task 1: Build container image (Buildah)
   - Task 2: Run unit + integration tests
   - Task 3: Trivy security scan (block on CRITICAL vulnerabilities)
   - Task 4: Deploy to staging namespace
   - Task 5: Automated canary deployment via Argo Rollouts (10% → 25% → 50% → 100% with Prometheus analysis)
   - On canary success: update stable deployment, remove canary
   - On canary failure: automatic rollback, alert team

4. **Security Posture**:
   - SCC: `restricted` for all application pods (non-root, read-only root FS, drop ALL capabilities)
   - RBAC: Developers have read-write in dev, read-only in prod. CI/CD ServiceAccount has deploy access in prod.
   - Secrets: External Secrets Operator syncs from HashiCorp Vault with 24-hour rotation
   - Images: SHA digests only, scanned by Trivy, enforced by OPA/Gatekeeper policy
   - TLS: OpenShift Routes with re-encrypt termination for externally-facing endpoints

5. **Disaster Recovery**:
   - PostgreSQL: CrunchyData Operator handles async replication to DR cluster (RPO < 5 minutes)
   - Kafka: Strimzi MirrorMaker 2 replicates topics to DR
   - ConfigMaps/Secrets: External Secrets Operator ensures DR cluster has latest secrets from Vault
   - Deployments: ArgoCD manages identical deployments in both clusters (GitOps)
   - Failover: GLB switches to DR cluster. Runbook documents manual steps for DNS cutover. RTO < 30 minutes.

6. **Observability**:
   - Prometheus: Custom metrics for GenAI (inference latency, response quality, GPU utilization)
   - Grafana: Dashboards for SRE team, alerting via PagerDuty
   - Logs: OpenShift Logging (Loki or Elasticsearch) aggregates all Pod logs
   - Alerts: OOMKilled, CrashLoopBackOff, error rate > 1%, p99 latency > 3s, GPU memory > 90%

**Trade-offs and Decisions**:

- **Active-standby vs active-active**: Active-active across regions would double GPU costs. For a GenAI platform, active-standby with <30 minute RTO is a better cost/availability balance.
- **MIG vs full GPU**: MIG reduces per-Pod GPU performance but increases cluster utilization from ~20% to ~80%. For embedding inference (which is not GPU-bound), MIG is cost-effective. For LLM fine-tuning, full GPU allocation is needed.
- **Argo Rollouts vs manual canary**: Automated canary is essential for banking compliance — it provides auditable, metrics-driven deployment decisions with no human error in the rollback decision.

**Key Points to Hit:**
- [ ] Two-cluster active-standby with GLB for disaster recovery
- [ ] GPU node pool with MIG for multi-tenant inference efficiency
- [ ] Zero-trust: default-deny NetworkPolicies + explicit allow rules
- [ ] Tekton CI/CD → Argo Rollouts canary → Prometheus analysis → auto-rollback
- [ ] SCC restricted, RBAC least-privilege, External Secrets Operator + Vault
- [ ] GitOps (ArgoCD) for consistent multi-cluster deployments
- [ ] RPO < 5 min (DB replication), RTO < 30 min (GLB failover)

**Follow-Up Questions:**
1. How would you handle a scenario where both clusters lose connectivity to HashiCorp Vault?
2. What is your strategy for rolling out a new Kubernetes version across the OpenShift cluster?
3. How would you implement multi-tenant GPU isolation for different banking teams sharing the same cluster?

**Source:** `banking-genai-engineering-academy/kubernetes-openshift/README.md`, `banking-genai-engineering-academy/kubernetes-openshift/multi-cluster.md`, `banking-genai-engineering-academy/kubernetes-openshift/secure-deployment-patterns.md`, `banking-genai-engineering-academy/kubernetes-openshift/gpu-workloads.md`, `banking-genai-engineering-academy/kubernetes-openshift/openshift-pipelines.md`, `banking-genai-engineering-academy/kubernetes-openshift/canary-releases.md`
