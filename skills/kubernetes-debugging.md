# Skill: Kubernetes Debugging

## Core Principles

1. **Start from the Pod** — Most issues are visible at the pod level. `kubectl describe pod` and `kubectl logs` are your first tools.
2. **Understand the Lifecycle** — Pods go through Pending → ContainerCreating → Running → (potentially) CrashLoopBackOff. Each state points to a different class of problem.
3. **Check Dependencies in Order** — Image pull → Config/Secrets → Networking → Readiness → Application logic. Debug in this order.
4. **Events Tell the Story** — `kubectl get events` shows what the cluster tried to do. Read them before guessing.
5. **Reproduce Locally First** — If possible, run the container locally with the same env vars and resource limits to isolate cluster vs. application issues.

## Mental Models

### The Kubernetes Debugging Flowchart
```
Pod Not Starting?
        │
        ▼
┌───────────────────┐
│ kubectl get pods  │  ← What state is it in?
└────────┬──────────┘
         │
    ┌────┼────┬─────────────┬──────────────────┐
    ▼    ▼    ▼             ▼                  ▼
  Pending  CrashLoop    ImagePull    Readiness
  │        BackOff      BackOff      Not Ready
  ▼        ▼            ▼            ▼
┌────┐  ┌──────┐   ┌────────┐   ┌────────┐
│Check│  │Check │   │Check   │   │Check   │
│Events│  │Logs  │   │Image   │   │Probe   │
│&    │  │&     │   │Name/   │   │Config  │
│Resources│App  │   │Pull    │   │& App   │
│      │  │Code  │   │Secrets │   │Health  │
└──────┘ └──────┘   └────────┘   └────────┘
```

### The Pod State Decision Tree
```
State: Pending
├── No node available? → Check node capacity, taints, tolerations
├── PVC not bound? → Check StorageClass, volume provisioning
├── ServiceAccount missing? → Check RBAC, token creation
└── Scheduler error? → Check scheduler logs, node selectors

State: ContainerCreating
├── Image pull failed? → Check image name, tag, registry credentials, imagePullSecrets
├── Secret/configmap missing? → Check they exist in the namespace
├── Volume mount failed? → Check volume definitions, permissions
└── Init container failed? → Check init container logs

State: Running but not Ready
├── Readiness probe failing? → Check probe config, app startup time
├── App crashed on startup? → Check container logs
├── Dependency unavailable? → Check downstream services, databases
└── Resource throttling? → Check CPU/memory limits, OOMKilled

State: CrashLoopBackOff
├── App panic on startup? → Check logs, env vars, config
├── Missing secret/configmap? → Check all referenced resources exist
├── Database connection failed? → Check network policies, credentials
├── OOMKilled? → Check memory limits, increase and retest
└── Liveness probe too aggressive? → Check initialDelaySeconds, periodSeconds
```

### The Debugging Checklist
```
□ kubectl get pods -n <namespace> — What state?
□ kubectl describe pod <pod> -n <namespace> — Events and details
□ kubectl logs <pod> -n <namespace> — Application logs
□ kubectl logs <pod> -n <namespace> --previous — Previous crash logs
□ kubectl get events -n <namespace> — Cluster-level events
□ kubectl get svc,endpoints -n <namespace> — Networking
□ kubectl get pvc,pv -n <namespace> — Storage
□ kubectl get configmap,secret -n <namespace> — Configuration
□ kubectl top pods -n <namespace> — Resource usage
□ kubectl exec -it <pod> -n <namespace> -- sh — Interactive debugging
□ kubectl port-forward <pod> <local>:<remote> — Test from localhost
```

## Step-by-Step Approach

### 1. Diagnose a CrashLoopBackOff

```bash
# Step 1: Identify the failing pod
kubectl get pods -n genai-platform
# NAME                              READY   STATUS             RESTARTS   AGE
# rag-service-6d4f8b7c9-xk2mv      0/1     CrashLoopBackOff   5          10m

# Step 2: Describe the pod for events
kubectl describe pod rag-service-6d4f8b7c9-xk2mv -n genai-platform
# Look at the Events section at the bottom:
#   Normal   Scheduled    pod assigned to node worker-03
#   Normal   Pulling      Pulling image "registry.internal/genai/rag-service:v2.3.1"
#   Normal   Pulled       Successfully pulled image
#   Normal   Created      Created container rag-service
#   Normal   Started      Started container rag-service
#   Warning  BackOff      Back-off restarting failed container

# Step 3: Check logs
kubectl logs rag-service-6d4f8b7c9-xk2mv -n genai-platform
# 2025-01-15T10:30:00Z ERROR Failed to connect to database:
#   connection refused: host=pg-primary.genai-db.svc.cluster.local port=5432

# Step 4: Check if the database service exists and is reachable
kubectl get svc pg-primary -n genai-db
kubectl get endpoints pg-primary -n genai-db
# ENDPOINTS: <none>  <-- No ready pods! The database is down.

# Step 5: Check the database pods
kubectl get pods -n genai-db
# NAME                              READY   STATUS
# pg-primary-0                      0/1     CrashLoopBackOff
```

### 2. Debug OOMKilled Pods

```bash
# Check if pod was OOMKilled
kubectl get pod rag-service-6d4f8b7c9-xk2mv -n genai-platform -o jsonpath='{.status.containerStatuses[0].lastState.terminated.reason}'
# Output: OOMKilled

# Check resource limits in the deployment
kubectl get deployment rag-service -n genai-platform -o jsonpath='{.spec.template.spec.containers[0].resources}'
# {"limits":{"cpu":"1","memory":"512Mi"},"requests":{"cpu":"250m","memory":"256Mi"}}

# Check actual memory usage before it crashed
kubectl top pod rag-service-6d4f8b7c9-xk2mv -n genai-platform --containers
# NAME           CPU(cores)   MEMORY(bytes)
# rag-service    150m         480Mi  <-- Approaching 512Mi limit!

# Fix: Increase memory limits
kubectl edit deployment rag-service -n genai-platform
# Change memory limit from 512Mi to 1Gi

# Or patch it directly
kubectl patch deployment rag-service -n genai-platform --type='json' \
  -p='[{"op": "replace", "path": "/spec/template/spec/containers/0/resources/limits/memory", "value": "1Gi"}]'
```

### 3. Debug Networking Issues in OpenShift

```bash
# Check if services can communicate
# From within a pod, test connectivity to downstream service
kubectl exec -it debug-pod -n genai-platform -- curl -v http://model-gateway.genai-inference.svc.cluster.local:8080/health

# Check NetworkPolicies
kubectl get networkpolicy -n genai-platform
kubectl describe networkpolicy genai-platform-policy -n genai-platform

# In OpenShift, check Routes (external access)
oc get routes -n genai-platform
oc describe route rag-service -n genai-platform

# Check DNS resolution inside the cluster
kubectl exec -it debug-pod -n genai-platform -- nslookup pg-primary.genai-db.svc.cluster.local

# Check if CoreDNS is healthy
kubectl get pods -n kube-system -l k8s-app=kube-dns
kubectl logs -n kube-system -l k8s-app=kube-dns
```

### 4. Debug Image Pull Failures

```bash
# The error typically shows:
# Failed to pull image "registry.internal/banking/genai-assistant:v1.2.3":
#   rpc error: code = Unknown desc = failed to pull and unpack image:
#   failed to resolve reference: unauthorized

# Check imagePullSecrets
kubectl get deployment rag-service -n genai-platform -o jsonpath='{.spec.template.spec.imagePullSecrets}'

# Verify the secret exists and is valid
kubectl get secret registry-credentials -n genai-platform -o jsonpath='{.data.\.dockerconfigjson}' | base64 -d

# Test image pull manually
docker pull registry.internal/banking/genai-assistant:v1.2.3

# In OpenShift, check the ImageStream
oc get is genai-assistant -n genai-platform
oc describe is genai-assistant -n genai-platform

# Common fix: ensure the service account has the right pull secret
kubectl patch serviceaccount default -n genai-platform -p '{"imagePullSecrets": [{"name": "registry-credentials"}]}'
```

### 5. Debug Readiness/Liveness Probe Failures

```yaml
# Problematic probe configuration (too aggressive)
readinessProbe:
  httpGet:
    path: /health
    port: 8080
  initialDelaySeconds: 5    # Too short for a Java/Python app
  periodSeconds: 3          # Too frequent
  timeoutSeconds: 1         # Too short
  failureThreshold: 2       # Too low

# Fixed configuration for a GenAI service (slow startup)
readinessProbe:
  httpGet:
    path: /health
    port: 8080
  initialDelaySeconds: 30   # Give time for model loading
  periodSeconds: 10
  timeoutSeconds: 5
  failureThreshold: 3
  successThreshold: 1

livenessProbe:
  httpGet:
    path: /healthz
    port: 8080
  initialDelaySeconds: 60   # Longer than readiness
  periodSeconds: 15
  timeoutSeconds: 5
  failureThreshold: 3
```

### 6. Interactive Debugging in a Running Pod

```bash
# Exec into a running pod
kubectl exec -it rag-service-6d4f8b7c9-xk2mv -n genai-platform -- /bin/bash

# Inside the pod:
# Check environment variables
env | grep -i database
env | grep -i api_key

# Check network connectivity
curl http://localhost:8080/health
curl http://pg-primary.genai-db.svc:5432  # Should fail (it's TCP, not HTTP)
nc -zv pg-primary.genai-db.svc 5432       # Test TCP connection

# Check file system
ls -la /app/config/
cat /app/config/settings.yaml

# Check resource usage
cat /sys/fs/cgroup/memory/memory.limit_in_bytes  # cgroup v1
cat /sys/fs/cgroup/memory.max                      # cgroup v2

# Run the application manually to see startup errors
python -m app.main --verbose

# Exit and check resource usage from outside
kubectl top pod rag-service-6d4f8b7c9-xk2mv -n genai-platform
```

### 7. Use Ephemeral Debug Containers

```bash
# Launch an ephemeral debug container in the same pod namespace
kubectl debug -it rag-service-6d4f8b7c9-xk2mv -n genai-platform \
  --image=nicolaka/netshoot --target=rag-service

# Now you have a full debugging toolkit:
# - tcpdump for network capture
# - curl for HTTP testing
# - dig for DNS
# - netstat for connections
# - strace for system call tracing

# Example: capture traffic to the database
tcpdump -i any host pg-primary.genai-db.svc and port 5432 -w /tmp/db-traffic.pcap
```

## Common Pitfalls

| Pitfall | Consequence | Fix |
|---------|------------|-----|
| Liveness probe more aggressive than readiness | Pod killed before it finishes starting | Liveness `initialDelaySeconds` > readiness |
| No resource limits set | Pod consumes all node resources, affects others | Always set requests and limits |
| Resource limits too low | OOMKilled or CPU throttling | Monitor actual usage, set limits with headroom |
| Missing imagePullSecrets | ImagePullBackOff on private registries | Configure service account with pull secrets |
| Hardcoded IPs instead of service DNS | Breaks when pods reschedule | Use `service-name.namespace.svc.cluster.local` |
| Not checking `--previous` logs | Misses the actual crash reason | Always check previous container logs |
| Ignoring node-level issues | Blaming the app when the node is the problem | Check `kubectl describe node`, disk pressure, PID pressure |
| Debugging the wrong namespace | Wasting time in the wrong context | Always specify `-n <namespace>` explicitly |
| Not using `kubectl get events` | Missing scheduler, volume, or RBAC errors | Events are the first place to look |
| Overlapping pod disruption budgets | Rolling updates get stuck | Ensure PDB allows at least one pod to be disrupted |

## Banking-Specific Concerns

1. **Air-Gapped Clusters** — Banking clusters often have no internet access. Image pulls must go through internal registries (e.g., Harbor). Debug by checking registry sync status and image promotion pipelines.
2. **Network Policies** — Strict network policies isolate namespaces. A common issue is forgetting to allow egress to the database namespace. Verify with `kubectl describe networkpolicy`.
3. **Security Context Constraints (SCCs)** — OpenShift SCCs restrict what containers can do. If a pod fails with `permission denied` on volume mounts, check SCCs: `oc describe scc restricted`.
4. **Compliance Logging** — All pod logs must be shipped to the centralized logging platform (e.g., Splunk). Debug missing logs by checking Fluentd/DaemonSet status.
5. **Certificate Rotation** — Banking clusters use internal PKI. Expired certs cause TLS handshake failures. Check cert-manager: `kubectl get certificates -A`.
6. **Multi-Cluster Routing** — GenAI services may run across DR clusters. Debug by checking service mesh (Istio) routing and health checks.

## GenAI-Specific Concerns

1. **GPU Resource Availability** — If using GPU-accelerated models, check that GPU nodes are available and properly labeled: `kubectl get nodes -l nvidia.com/gpu.present=true`.
2. **Large Model Loading Time** — LLMs can take 2-5 minutes to load into memory. Readiness probes must account for this. Use `initialDelaySeconds: 120` or higher.
3. **Model Cache Eviction** — If multiple models share a pod, cache misses cause latency spikes. Monitor with custom metrics: `kubectl get --raw /apis/custom.metrics.k8s.io/v1beta1/namespaces/genai-platform/pods/*/model_cache_hit_rate`.
4. **Token Rate Limiting** — External model APIs (OpenAI, Anthropic) have rate limits. If pods start crashing due to API throttling, check application-level rate limiter configuration.
5. **Embedding Pipeline Backlog** — Document ingestion pipelines can fall behind. Check queue depth and processing rate: `kubectl get --raw /apis/custom.metrics.k8s.io/v1beta1/.../embedding_queue_depth`.

## Metrics to Monitor

| Metric | Alert Threshold | Why It Matters |
|--------|----------------|----------------|
| Pod restart count | > 3 in 1 hour | CrashLoopBackOff or instability |
| OOMKilled events | > 0 | Memory limits too low or memory leak |
| Pod scheduling latency | > 5 minutes | Insufficient cluster capacity |
| Container CPU throttling | > 10% of requests | CPU limits too restrictive |
| Image pull duration | > 2 minutes | Large images or registry issues |
| Readiness probe failure rate | > 5% | Application health or probe misconfiguration |
| Node NotReady count | > 0 | Node-level failure affecting workloads |
| PersistentVolume claim failures | > 0 | Storage provisioning issues |
| NetworkPolicy denial rate | > 1% per minute | Misconfigured network policies |
| Certificate expiry | < 30 days | Upcoming TLS failures |

## Interview Questions

1. A pod is stuck in `Pending` state. Walk me through your debugging process.
2. What is the difference between a liveness probe and a readiness probe? What happens if each fails?
3. How would you debug a service that returns 503 errors intermittently?
4. A GenAI service with a 2GB model keeps getting OOMKilled. What do you check?
5. How do you debug network connectivity between two pods in different namespaces?
6. What are the common causes of `ImagePullBackOff` and how do you fix each?
7. How would you set up proactive alerting for pod health in a production cluster?
8. Explain how you would debug a slow rolling deployment that takes 20 minutes instead of 2.

## Hands-On Exercise

### Exercise: Debug a Crashing GenAI API Service

**Problem:** The `rag-service` in the `genai-platform` namespace keeps crashing after deployment. Your team lead says "it was working yesterday." The service provides RAG retrieval for the internal banking assistant and is currently unavailable.

**Scenario:**
- The deployment was updated from `v2.3.0` to `v2.3.1` last night
- The update included a new embedding model (text-embedding-3-large instead of text-embedding-3-small)
- The database connection string was also changed to use a new read replica
- No one updated the resource limits

**Constraints:**
- You cannot modify the application code (it's already built and deployed)
- You have full `kubectl` access to the cluster
- The cluster is an OpenShift 4.14 environment
- The service is critical — the risk management team cannot use the assistant

**Expected Output:**
- Identify all root causes (there are at least 3)
- Fix each issue using `kubectl` or `oc` commands
- Document the debugging steps you took
- Propume resource limit recommendations based on observed usage

**Hints:**
- Start with `kubectl get pods` and work through the state machine
- Check what changed between v2.3.0 and v2.3.1
- The larger embedding model needs more memory
- The read replica might not be ready yet
- Check the ConfigMap for the database connection string

**Extension:**
- Write a pre-deployment checklist that would have caught these issues
- Create a Kubernetes HorizontalPodAutoscaler that scales based on custom metrics (queue depth)
- Design a canary deployment strategy that would catch these issues before full rollout

---

**Related files:**
- `kubernetes-openshift/kubernetes-fundamentals.md` — Core K8s concepts
- `kubernetes-openshift/openshift-deployment-patterns.md` — OpenShift-specific patterns
- `cicd-devops/deployment-strategies.md` — Blue-green, canary deployments
- `observability/kubernetes-observability.md` — Monitoring K8s workloads
- `infrastructure/cluster-capacity-planning.md` — Resource sizing
- `skills/openshift-deployments.md` — OpenShift deployment patterns
- `skills/distributed-system-debugging.md` — Cross-service debugging
