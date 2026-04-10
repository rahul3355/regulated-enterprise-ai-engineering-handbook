# Kubernetes Terms

> Essential Kubernetes and container orchestration terminology for GenAI platform engineers. Each term includes definition, banking/GenAI example, related concepts, and common misunderstandings.

## Glossary

### 1. Pod
**Definition:** The smallest deployable unit in Kubernetes — one or more containers that share storage, network, and lifecycle.
**Example:** The GenAI chat service runs as a pod containing the FastAPI application container and a sidecar container for audit logging.
**Related Concepts:** Container, deployment, replica, node
**Common Misunderstanding:** A pod is not a container — it's a wrapper around one or more containers that always run together on the same node.

### 2. Deployment
**Definition:** A Kubernetes resource that manages a set of identical pods, handling rollout, rollback, and scaling.
**Example:** The `genai-chat` deployment manages 4 replicas of the chat service pod, with rolling update strategy (1 pod at a time).
**Related Concepts:** ReplicaSet, rollout, rollback, strategy
**Common Misunderstanding:** Deployments are for stateless applications. Stateful applications (databases) use StatefulSets.

### 3. Service
**Definition:** An abstraction that exposes a set of pods as a network service with a stable IP and DNS name.
**Example:** The `genai-chat` service (ClusterIP) provides a stable internal endpoint `genai-chat.genai.svc.cluster.local:8000` that other services use to reach the chat pods.
**Related Concepts:** ClusterIP, NodePort, LoadBalancer, Endpoint, Ingress
**Common Misunderstanding:** A Service is not a pod — it's a network abstraction that routes traffic to pods matching a label selector.

### 4. Namespace
**Definition:** A virtual cluster within a physical cluster, used to isolate resources.
**Example:** The GenAI platform uses separate namespaces: `genai-dev`, `genai-staging`, and `genai-production` for environment isolation.
**Related Concepts:** Resource quota, network policy, RBAC
**Common Misunderstanding:** Namespaces provide logical isolation, not security isolation. Pods in different namespaces can still communicate unless NetworkPolicies restrict them.

### 5. ReplicaSet
**Definition:** Ensures a specified number of pod replicas are running at all times.
**Example:** The genai-chat deployment creates a ReplicaSet that maintains 4 running pods. If one pod crashes, the ReplicaSet creates a replacement.
**Related Concepts:** Deployment, pod, scaling, self-healing
**Common Misunderstanding:** You rarely create ReplicaSets directly — Deployments manage them automatically.

### 6. ConfigMap
**Definition:** A Kubernetes resource for storing non-confidential configuration data as key-value pairs.
**Example:** The GenAI chat service's configuration (model name, rate limit, log level) is stored in a ConfigMap and mounted as environment variables.
**Related Concepts:** Secret, environment variable, volume mount
**Common Misunderstanding:** ConfigMaps are not encrypted and should NOT store sensitive data like API keys. Use Secrets for those.

### 7. Secret
**Definition:** A Kubernetes resource for storing sensitive data (API keys, passwords, certificates) in base64-encoded form.
**Example:** The LLM API key and database credentials are stored as Kubernetes Secrets and mounted as environment variables in the GenAI pods.
**Related Concepts:** ConfigMap, encryption at rest, external secrets (Vault)
**Common Misunderstanding:** Secrets are base64-encoded, NOT encrypted. Anyone with `kubectl get secret` access can decode them. Enable encryption at rest on etcd.

### 8. Ingress
**Definition:** A Kubernetes resource that manages external HTTP/S access to services within the cluster, with routing rules.
**Example:** The Ingress routes `genai.bank.internal/chat` to the genai-chat service and `genai.bank.internal/docs` to the genai-docs service, with TLS termination.
**Related Concepts:** Service, load balancer, TLS, path-based routing
**Common Misunderstanding:** Ingress is just a routing rule — it needs an Ingress Controller (like nginx or Traefik) to actually work.

### 9. Horizontal Pod Autoscaler (HPA)
**Definition:** Automatically scales the number of pod replicas based on observed metrics (CPU, memory, custom metrics).
**Example:** The HPA scales the genai-chat deployment from 4 to 12 replicas when CPU utilization exceeds 70%, and scales back down when traffic decreases.
**Related Concepts:** VPA, Cluster Autoscaler, metrics server
**Common Misunderstanding:** HPA scales replicas, not individual pod resources. For per-pod resource adjustments, use VPA.

### 10. Resource Requests and Limits
**Definition:** The minimum (request) and maximum (limit) CPU and memory a container is guaranteed or allowed to use.
**Example:** The GenAI retriever pod requests 512Mi memory and 250m CPU, with limits of 1Gi memory and 500m CPU.
**Related Concepts:** QoS class, OOMKill, CPU throttling, scheduling
**Common Misunderstanding:** Requests affect scheduling (node must have enough resources). Limits affect runtime enforcement (container is killed/capped if exceeded).

### 11. OOMKilled
**Definition:** A pod termination state where the container exceeded its memory limit and was killed by the Linux OOM killer.
**Example:** The GenAI retriever pod is OOMKilled because the embedding model loaded into memory plus query buffers exceeded the 512Mi limit.
**Related Concepts:** Memory limit, resource requests, memory leak, eviction
**Common Misunderstanding:** OOMKill means the container used too much memory, NOT that the node ran out of memory (that's node pressure/eviction).

### 12. CrashLoopBackOff
**Definition:** A pod state where the container repeatedly crashes and Kubernetes is backing off before restarting it.
**Example:** The genai-chat pod enters CrashLoopBackOff because the LLM_API_KEY environment variable is missing, causing the application to crash on startup.
**Related Concepts:** Readiness probe, liveness probe, restart policy
**Common Misunderstanding:** CrashLoopBackOff is not a Kubernetes bug — it's Kubernetes correctly retrying a failing container. Fix the root cause in the container.

### 13. Readiness Probe
**Definition:** A check that determines whether a pod is ready to receive traffic.
**Example:** The GenAI chat pod's readiness probe checks `GET /healthz` — it returns 200 only when the database connection and model are loaded.
**Related Concepts:** Liveness probe, startup probe, rolling update
**Common Misunderstanding:** A failing readiness probe doesn't restart the pod — it just removes it from service traffic. A failing liveness probe restarts it.

### 14. Liveness Probe
**Definition:** A check that determines whether a pod is still alive and functioning. If it fails, Kubernetes restarts the pod.
**Example:** The GenAI retriever's liveness probe checks that the embedding model process is still responding. If it hangs, the probe fails and the pod restarts.
**Related Concepts:** Readiness probe, startup probe, restart policy
**Common Misunderstanding:** Don't use the same endpoint for liveness and readiness. A slow/deadlocked process might pass readiness (still accepting connections) but fail liveness (not actually processing).

### 15. StatefulSet
**Definition:** A workload API for managing stateful applications, providing stable pod identities and persistent storage.
**Example:** The pgvector PostgreSQL database runs as a StatefulSet with persistent volume claims, ensuring data persists across pod restarts.
**Related Concepts:** PersistentVolume, PersistentVolumeClaim, headless service
**Common Misunderstanding:** StatefulSets are for databases and similar stateful workloads. Use Deployments for stateless applications.

### 16. PersistentVolume (PV) and PersistentVolumeClaim (PVC)
**Definition:** PV is a piece of storage in the cluster. PVC is a request for storage by a user/pod.
**Example:** The PostgreSQL StatefulSet claims a 100Gi PVC, which is backed by a network-attached PV (EBS volume in AWS).
**Related Concepts:** StorageClass, StatefulSet, volume
**Common Misunderstanding:** PVs and PVCs are separate resources. PVCs request storage; PVs fulfill those requests. They're like supply and demand.

### 17. NetworkPolicy
**Definition:** A Kubernetes resource that controls network traffic between pods (like a firewall for pod-to-pod communication).
**Example:** A NetworkPolicy allows only genai-chat pods to connect to genai-vector-db on port 5432. All other pod-to-pod traffic to the database is denied.
**Related Concepts:** Zero Trust, micro-segmentation, firewall, CNI
**Common Misunderstanding:** NetworkPolicies are deny-by-default only when a policy exists. Without any NetworkPolicy, all pod-to-pod traffic is allowed.

### 18. Rolling Update
**Definition:** A deployment strategy that replaces pods one at a time, ensuring zero-downtime deployments.
**Example:** When deploying a new GenAI chat version, the rolling update creates 1 new pod, waits for it to be ready, then removes 1 old pod, repeating until all 4 pods are updated.
**Related Concepts:** maxSurge, maxUnavailable, deployment strategy
**Common Misunderstanding:** Rolling updates don't test the new version under full load before committing. Blue-green or canary deployments are safer for critical services.

### 19. DaemonSet
**Definition:** Ensures that all (or some) nodes run a copy of a pod.
**Example:** The logging agent (Fluentd) runs as a DaemonSet — one pod per node, collecting logs from all containers on that node.
**Related Concepts:** Node selector, tolerations, system-level pods
**Common Misunderstanding:** DaemonSets are for system-level infrastructure (logging, monitoring), not application services.

### 20. RBAC (Role-Based Access Control in Kubernetes)
**Definition:** Kubernetes-native access control that regulates who can perform what actions on which resources.
**Example:** A ServiceAccount for the GenAI deployment has permission to read ConfigMaps and Secrets in the `genai` namespace, but cannot modify Deployments or access other namespaces.
**Related Concepts:** ServiceAccount, Role, ClusterRole, RoleBinding
**Common Misunderstanding:** Kubernetes RBAC controls access to Kubernetes resources, not to application data. Application-level auth is separate.

### 21. ServiceAccount
**Definition:** An identity for processes (pods) that run in a pod, used to authenticate to the Kubernetes API.
**Example:** The GenAI chat pod uses a dedicated ServiceAccount with permissions to read its own ConfigMap and Secrets, but nothing else.
**Related Concepts:** RBAC, Role, ClusterRole, token
**Common Misunderstanding:** Every pod has a default ServiceAccount. Don't use it for production — create dedicated ServiceAccounts with minimal permissions.

### 22. Init Container
**Definition:** A container that runs to completion before the main application containers start, used for setup tasks.
**Example:** An init container runs database migrations before the GenAI chat pod starts, ensuring the schema is up to date before the application connects.
**Related Concepts:** Sidecar container, lifecycle, ordering
**Common Misunderstanding:** If an init container fails, the pod won't start. Init containers run sequentially — each must succeed before the next starts.

### 23. Sidecar Container
**Definition:** An additional container in the same pod as the main application, providing supporting functionality.
**Example:** The GenAI chat pod includes a sidecar container that captures and forwards audit logs to the central audit database, running alongside the main application.
**Related Concepts:** Init container, pod, ambassador pattern
**Common Misunderstanding:** Sidecars share the pod's network (localhost) and lifecycle. They're not independent services.

### 24. Helm
**Definition:** A package manager for Kubernetes that simplifies deploying complex applications using charts (templates).
**Example:** The GenAI platform is deployed using a Helm chart that parameterizes replica count, resource limits, image versions, and configuration for different environments.
**Related Concepts:** Chart, values, template, release
**Common Misunderstanding:** Helm is not required for Kubernetes. It's a convenience tool. Simple applications can be deployed with raw YAML manifests.

### 25. Operator
**Definition:** A method of packaging, deploying, and managing a Kubernetes application using custom resources and a controller.
**Example:** The PostgreSQL Operator manages database clusters automatically — handling backups, failover, scaling, and upgrades without manual intervention.
**Related Concepts:** CRD (Custom Resource Definition), controller, reconciliation loop
**Common Misunderstanding:** Operators are not just deployment scripts. They run continuously, watching for changes and reconciling the actual state with the desired state.
