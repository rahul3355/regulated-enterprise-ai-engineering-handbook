# Compute Options

## Overview

This document covers compute options for banking GenAI workloads, including VMs, containers, serverless functions, and GPU instances. It provides decision criteria, sizing guidance, and cost optimization strategies.

---

## Compute Hierarchy

```
┌──────────────────────────────────────────────────────┐
│                Compute Decision Tree                  │
│                                                      │
│  Does it need a GPU?                                 │
│  ├── Yes → GPU Instance (A100, H100, L40S)           │
│  └── No → Does it need long-running state?           │
│       ├── Yes → Container (Kubernetes)               │
│       └── No → Is it event-driven?                   │
│            ├── Yes → Serverless (Lambda, Functions)  │
│            └── No → Container (Kubernetes)           │
└──────────────────────────────────────────────────────┘
```

---

## Virtual Machines

### When to Use VMs

- Legacy applications that cannot be containerized
- GPU workloads requiring direct hardware access
- Database workloads with specific I/O requirements
- Bare-metal performance needs

### VM Sizing for GenAI

| Workload | Recommended VM | vCPUs | RAM | Storage | Notes |
|----------|---------------|-------|-----|---------|-------|
| API Gateway | Standard_D4s_v5 | 4 | 16 GB | 100 GB SSD | Low latency, moderate CPU |
| Application Server | Standard_D8s_v5 | 8 | 32 GB | 100 GB SSD | Moderate CPU, memory |
| Database (PostgreSQL) | Standard_E8s_v5 | 8 | 64 GB | 1 TB Premium SSD | Memory-optimized |
| Vector DB Node | Standard_L8s_v3 | 8 | 64 GB | 2 TB NVMe | Storage-optimized |
| Monitoring Node | Standard_D4s_v5 | 4 | 16 GB | 500 GB SSD | Moderate I/O |

---

## Containers (Kubernetes)

### When to Use Containers

- Microservices architectures
- Services that need auto-scaling
- Services that need self-healing
- Multi-cloud portability requirements
- CI/CD-driven deployment workflows

### Node Pool Design for GenAI

```yaml
# Production AKS/EKS Node Pools

nodePools:
  # General application workloads
  - name: general
    vmSize: Standard_D4s_v5  # 4 vCPU, 16 GB RAM
    count: 6
    minCount: 3
    maxCount: 20
    taints: []
    labels:
      workload: general

  # GPU inference workloads
  - name: gpu-inference
    vmSize: Standard_NC24ads_A100_v4  # 1x A100 80GB
    count: 4
    minCount: 2
    maxCount: 8
    taints:
      - key: sku
        value: gpu
        effect: NoSchedule
    labels:
      workload: inference

  # GPU training workloads
  - name: gpu-training
    vmSize: Standard_ND96amsr_A100_v4  # 8x A100 80GB
    count: 2
    minCount: 0
    maxCount: 4
    taints:
      - key: workload
        value: training
        effect: NoSchedule
    labels:
      workload: training

  # High-memory workloads (vector DB, cache)
  - name: high-memory
    vmSize: Standard_E16s_v5  # 16 vCPU, 128 GB RAM
    count: 3
    minCount: 2
    maxCount: 6
    labels:
      workload: data-intensive
```

### Container Resource Requests

```yaml
# AssistBot API Pod
resources:
  requests:
    cpu: "500m"
    memory: "1Gi"
  limits:
    cpu: "2000m"
    memory: "4Gi"

# Embedding Service Pod
resources:
  requests:
    cpu: "1000m"
    memory: "2Gi"
  limits:
    cpu: "4000m"
    memory: "8Gi"

# GPU Inference Pod (Triton)
resources:
  requests:
    cpu: "2000m"
    memory: "8Gi"
    nvidia.com/gpu: 1
  limits:
    cpu: "8000m"
    memory: "32Gi"
    nvidia.com/gpu: 1
```

---

## GPU Instances

### GPU Selection for GenAI

| GPU | Memory | Best For | Hourly Cost (approx) |
|-----|--------|----------|---------------------|
| **NVIDIA A100 80GB** | 80 GB HBM2e | Large model inference (70B+), training | $3.50 |
| **NVIDIA H100 80GB** | 80 GB HBM3 | Training, very large model inference | $5.00 |
| **NVIDIA L40S 48GB** | 48 GB GDDR6 | Medium model inference (7B-70B) | $1.80 |
| **NVIDIA L4 24GB** | 24 GB GDDR6 | Small model inference (up to 7B) | $0.80 |
| **NVIDIA T4 16GB** | 16 GB GDDR6 | Embedding, small models | $0.50 |

### Model-to-GPU Mapping

| Model | Parameters | Min GPU Memory | Recommended GPU |
|-------|-----------|---------------|----------------|
| GPT-4 (API) | N/A | N/A | Azure OpenAI (managed) |
| Llama-3-8B | 8B | 16 GB | L4 24GB |
| Llama-3-70B | 70B | 80 GB (quantized: 40 GB) | A100 80GB |
| Mistral-7B | 7B | 14 GB | L4 24GB |
| BERT-base | 110M | 2 GB | T4 16GB |
| Embedding (Ada-002) | N/A | N/A | Azure OpenAI (managed) |

### GPU Optimization Strategies

1. **Quantization**: 8-bit or 4-bit quantization reduces GPU memory by 2-4x with minimal quality loss
2. **vLLM / TensorRT-LLM**: Optimized inference engines that increase throughput 3-5x
3. **PagedAttention**: Efficient KV cache management for higher batch sizes
4. **Continuous Batching**: Process requests as they arrive rather than static batches
5. **Speculative Decoding**: Use a small draft model to speed up large model generation

### GPU Cost Optimization

| Strategy | Savings | Trade-off |
|----------|---------|-----------|
| **Reserved instances (1 year)** | 40-60% | Commitment |
| **Spot instances (training)** | 60-90% | Preemption risk |
| **MIG partitioning** | 2-7x utilization | Performance isolation |
| **Multi-tenant inference** | 2-3x utilization | Noise between tenants |
| **Right-sizing GPU** | 30-50% | May need multiple GPU types |

---

## Serverless Functions

### When to Use Serverless

- Event-driven processing (document uploads, webhook handling)
- Sporadic, unpredictable workloads
- Short-lived tasks (< 15 minutes)
- Glue code between services

### GenAI Serverless Use Cases

| Use Case | Function | Timeout | Memory | Trigger |
|----------|----------|---------|--------|---------|
| Document OCR | Azure Functions | 5 min | 2 GB | Blob upload |
| Embedding Queue Processor | Lambda | 15 min | 1 GB | SQS message |
| Webhook Handler | Cloud Functions | 30 sec | 512 MB | HTTP |
| Log Processor | Lambda | 5 min | 1 GB | CloudWatch Logs |
| Scheduled Cleanup | Functions | 10 min | 512 MB | Timer |

### Serverless Limitations for GenAI

| Limitation | Impact |
|-----------|--------|
| Max execution time (15 min) | Cannot run long inference jobs |
| Cold start latency (1-10 sec) | Unsuitable for low-latency inference |
| Memory limit (10 GB) | Cannot load large models |
| No GPU access | Cannot run GPU-accelerated inference |
| State limitation | Must use external storage for state |

---

## Auto-Scaling

### HPA (Horizontal Pod Autoscaler) for GenAI

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: assistbot-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: assistbot
  minReplicas: 3
  maxReplicas: 20
  metrics:
    # Scale on CPU utilization
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70
    # Scale on request rate (custom metric)
    - type: Pods
      pods:
        metric:
          name: requests_per_second
        target:
          type: AverageValue
          averageValue: "500"
    # Scale on inference queue depth
    - type: External
      external:
        metric:
          name: inference_queue_depth
        target:
          type: AverageValue
          averageValue: "100"
```

### GPU Node Auto-Scaling

```yaml
# Cluster Autoscaler for GPU nodes
apiVersion: autoscaling.k8s.io/v1
kind: ScaleSet:
  name: gpu-inference
  minSize: 2
  maxSize: 8
  # Scale up when pods are pending due to GPU resource requests
  # Scale down when GPU utilization < 30% for 10 minutes
```

---

## Compute Capacity Planning

### Baseline Capacity

| Component | Instances | Utilization Target | Peak Headroom |
|-----------|-----------|-------------------|---------------|
| API Gateway | 3 | 50% | 2x |
| Application Pods | 6 | 60% | 1.5x |
| GPU Inference | 4 | 70% | 1.3x |
| Database | 3 (primary + 2 replicas) | 50% | 2x |
| Vector DB | 3 | 60% | 1.5x |

### Peak Capacity Planning

| Scenario | Capacity Increase | Auto-Scale? |
|----------|------------------|-------------|
| Normal business hours | Baseline | Yes |
| Month-end processing | 1.5x baseline | Yes |
| Product launch | 2-3x baseline | Yes, pre-scale |
| DR failover | Full capacity in secondary region | Manual or automated |

---

## Compute Cost Management

### Cost Allocation by Component

| Component | Monthly Cost | % of Total | Optimization Strategy |
|-----------|-------------|------------|----------------------|
| GPU Inference | £15,000 | 35% | Reserved instances, right-sizing |
| Application Compute | £5,000 | 12% | Auto-scaling, spot for non-prod |
| LLM API Calls | £18,000 | 42% | Prompt optimization, caching |
| Database | £3,000 | 7% | Right-sizing, reserved |
| Other | £2,000 | 4% | Various |
| **Total** | **£43,000** | **100%** | |

---

## Cross-References

- [README.md](README.md) -- Infrastructure overview
- [cloud-providers.md](cloud-providers.md) -- Cloud provider selection
- [infrastructure-cost-management.md](infrastructure-cost-management.md) -- Cost management
- [../kubernetes-openshift/README.md](../kubernetes-openshift/README.md) -- Kubernetes operations
- [../genai-platforms/README.md](../genai-platforms/README.md) -- GenAI platform
