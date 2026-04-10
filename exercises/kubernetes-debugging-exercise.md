# Kubernetes Debugging Exercise

> Debug CrashLoopBackOff, OOMKilled, and other common pod failures in a GenAI platform.

## Problem Statement

You are the platform engineer for the GenAI system running on OpenShift/Kubernetes. Multiple pods are failing. Debug and resolve each issue.

## Scenario 1: CrashLoopBackOff

```
$ kubectl get pods -n genai
NAME                              READY   STATUS             RESTARTS   AGE
genai-chat-6b4d7f8c9-x2k4p       0/1     CrashLoopBackOff   8          15m
genai-chat-6b4d7f8c9-m9n3q       0/1     CrashLoopBackOff   7          15m
genai-retriever-5c8d6e7f1-a1b2   1/1     Running            0          2d
genai-audit-7d9e8f0a2-c3d4       1/1     Running            0          2d

$ kubectl logs genai-chat-6b4d7f8c9-x2k4p -n genai
Traceback (most recent call last):
  File "/app/main.py", line 12, in <module>
    from genai_service import ChatService
  File "/app/genai_service/__init__.py", line 5, in <module>
    from .config import Settings
  File "/app/genai_service/config.py", line 23, in <module>
    settings = Settings()
  File "/app/genai_service/config.py", line 15, in __init__
    self.llm_api_key = os.environ["LLM_API_KEY"]
                       ~~~~~~~~~~^^^^^^^^^^^^^^^
  KeyError: 'LLM_API_KEY'

$ kubectl describe pod genai-chat-6b4d7f8c9-x2k4p -n genai
...
Environment:
  DB_HOST:        genai-postgres.genai.svc.cluster.local
  DB_PORT:        5432
  REDIS_URL:      genai-redis.genai.svc.cluster.local:6379
  MODEL_NAME:     gpt-4-turbo
  # LLM_API_KEY is MISSING
...
```

**Your task:** Diagnose and fix the CrashLoopBackOff.

### Solution

The `LLM_API_KEY` environment variable is not set in the deployment. It should come from a Kubernetes Secret.

```bash
# Check if the secret exists
$ kubectl get secrets -n genai | grep llm
llm-api-credentials   Opaque   1   30d

# Check what keys the secret has
$ kubectl get secret llm-api-credentials -n genai -o jsonpath='{.data}' | jq
{
  "api-key": "c2stbGxtLXByb2QteHl6..."  # base64 encoded
}

# The secret key is 'api-key' but the deployment expects 'LLM_API_KEY'
# Check the deployment spec
$ kubectl get deployment genai-chat -n genai -o yaml | grep -A 5 envFrom
# The secret reference is missing or misconfigured

# Fix: Add the secret reference to the deployment
$ kubectl set env deployment/genai-chat \
    --from=secret/llm-api-credentials \
    -n genai \
    --keys=api-key=LLM_API_KEY
```

**Root cause:** The deployment manifest was missing the secret reference for `LLM_API_KEY`. The secret existed but wasn't mounted as an environment variable.

## Scenario 2: OOMKilled

```
$ kubectl get pods -n genai
NAME                              READY   STATUS      RESTARTS   AGE
genai-retriever-5c8d6e7f1-e5f6   0/1     OOMKilled   3          8m
genai-retriever-5c8d6e7f1-g7h8   1/1     Running     0          2d
genai-chat-6b4d7f8c9-x2k4p       1/1     Running     0          1h

$ kubectl describe pod genai-retriever-5c8d6e7f1-e5f6 -n genai
...
Last State:     Terminated
  Reason:       OOMKilled
  Exit Code:    137
...
Containers:
  retriever:
    Limits:
      memory:  512Mi
      cpu:     500m
    Requests:
      memory:  256Mi
      cpu:     250m
...

$ kubectl top pod genai-retriever-5c8d6e7f1-e5f6 -n genai
error: Metrics not available for terminated pod

# Check the surviving pod
$ kubectl top pod genai-retriever-5c8d6e7f1-g7h8 -n genai
NAME                              CPU    MEMORY
genai-retriever-5c8d6e7f1-g7h8   180m   480Mi
```

**Your task:** Diagnose and fix the OOMKilled.

### Solution

The memory limit (512Mi) is too tight. The surviving pod uses 480Mi, which is close to the limit.

```yaml
# The retriever loads embedding models into memory
# Model size: all-MiniLM-L6-v2 = ~90MB
# Plus Python overhead, document buffers, connection pools
# Actual usage: 450-520Mi depending on query load

# Fix: Increase memory limits
apiVersion: apps/v1
kind: Deployment
metadata:
  name: genai-retriever
spec:
  template:
    spec:
      containers:
      - name: retriever
        resources:
          requests:
            memory: "512Mi"
            cpu: "250m"
          limits:
            memory: "1Gi"     # Increased from 512Mi
            cpu: "500m"
```

**Root cause:** Memory limits were set too low. The embedding model + application needs ~500Mi, but the limit was 512Mi. Under load, memory spiked above the limit and the pod was OOMKilled.

**Why did one pod survive?** It happened to be scheduled on a node with slightly lower load, keeping memory just under the limit. This is luck, not reliability.

## Scenario 3: Readiness Probe Failing

```
$ kubectl get pods -n genai
NAME                              READY   STATUS    RESTARTS   AGE
genai-chat-6b4d7f8c9-p5q6r       0/1     Running   0          3m
genai-chat-6b4d7f8c9-s7t8u       1/1     Running   0          2d

$ kubectl describe pod genai-chat-6b4d7f8c9-p5q6r -n genai
...
Conditions:
  Type              Status
  Initialized       True
  Ready             False
  ContainersReady   False
  PodScheduled      True
Events:
  Type     Reason     Age   From     Message
  ----     ------     ----  ----     -------
  Warning  Unhealthy  30s   kubelet  Readiness probe failed:
           HTTP probe failed with statuscode: 503
...

$ kubectl exec genai-chat-6b4d7f8c9-p5q6r -n genai -- \
    curl -s http://localhost:8000/health
{"status": "starting", "model_loaded": false}

$ kubectl exec genai-chat-6b4d7f8c9-p5q6r -n genai -- \
    curl -s http://localhost:8000/healthz
{"status": "unhealthy", "reason": "vector_db connection failed"}
```

**Your task:** Diagnose and fix the readiness probe failure.

### Solution

The readiness probe checks `/healthz` which verifies database connectivity. The vector DB connection is failing on this pod.

```bash
# Check if the vector DB service is reachable
$ kubectl exec genai-chat-6b4d7f8c9-p5q6r -n genai -- \
    nc -zv genai-vector-db.genai.svc.cluster.local 5432
nc: connect to genai-vector-db.genai.svc.cluster.local:5432 failed: Connection refused

# Check the vector DB pods
$ kubectl get pods -n genai | grep vector
genai-vector-db-0   1/1   Running   0   5d

# The DB pod is running. Check if it accepts connections
$ kubectl exec genai-vector-db-0 -n genai -- \
    pg_isready -U rag_user -d genai
/var/run/postgresql:5432 - accepting connections

# The DB is fine. Check network policy
$ kubectl get networkpolicy -n genai
NAME               POD-SELECTOR    AGE
deny-all           <none>          5d
allow-chat-to-db   app=genai-chat  5d

# Check the network policy rules
$ kubectl get networkpolicy allow-chat-to-db -n genai -o yaml
# The policy allows chat pods to reach the DB on port 5432

# Check DNS resolution
$ kubectl exec genai-chat-6b4d7f8c9-p5q6r -n genai -- \
    nslookup genai-vector-db.genai.svc.cluster.local
Server:  10.96.0.10
Address: 10.96.0.10#53

** server can't find genai-vector-db.genai.svc.cluster.local: NXDOMAIN

# DNS issue! The service name doesn't resolve
$ kubectl get svc -n genai | grep vector
genai-vector-db   ClusterIP   10.96.45.123   <none>   5432/TCP   5d
# Service exists. Check if the CoreDNS pods are healthy
$ kubectl get pods -n kube-system | grep coredns
coredns-5d78c9869d-abc12   1/1   Running   0   10d

# CoreDNS is healthy. This might be a transient DNS caching issue.
# Restart the pod and it may resolve.
$ kubectl delete pod genai-chat-6b4d7f8c9-p5q6r -n genai
pod "genai-chat-6b4d7f8c9-p5q6r" deleted

# New pod comes up healthy
$ kubectl get pods -n genai | grep chat
genai-chat-6b4d7f8c9-v9w0x   1/1   Running   0   30s
```

**Root cause:** Transient DNS resolution failure for the vector DB service. The new pod resolved correctly. This could be caused by CoreDNS cache staleness or a brief network policy propagation delay.

**Prevention:** Add `initialDelaySeconds` to the readiness probe to give DNS time to resolve, and add DNS policy `dnsPolicy: ClusterFirstWithHostNet` if needed.

## Extensions

1. **Debug ImagePullBackOff:** A pod can't pull its container image. Diagnose registry auth, image name, and network issues.

2. **Debug ConfigMap issues:** A pod starts but uses wrong configuration because the ConfigMap was updated but not remounted.

3. **Debug resource contention:** Two pods on the same node compete for CPU, causing one to be throttled. Use `kubectl top` and node metrics.

4. **Write a troubleshooting runbook:** Create a decision-tree runbook for common pod failures (CrashLoopBackOff, OOMKilled, ImagePullBackOff, Pending, Readiness failing).

5. **Debug a StatefulSet:** A StatefulSet pod fails because its PVC is stuck in `Terminating` state. Diagnose and resolve.

## Interview Relevance

Kubernetes debugging is essential for platform engineering roles:

| Skill | Why It Matters |
|-------|---------------|
| kubectl diagnostics | Core debugging tool |
| Understanding probes | Liveness vs. readiness vs. startup |
| Resource management | Requests, limits, OOMKill |
| Secret/ConfigMap management | Configuration in K8s |
| Network troubleshooting | DNS, network policies, service routing |

**Follow-up questions:**
- "What's the difference between liveness and readiness probes?"
- "When would you use requests vs. limits?"
- "How do you debug a pod that's in Pending state?"
- "What causes a pod to be in ImagePullBackOff?"
