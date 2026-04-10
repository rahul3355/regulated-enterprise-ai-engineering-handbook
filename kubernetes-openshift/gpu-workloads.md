# GPU Workloads: Scheduling, MIG, and GenAI Model Serving

## Overview

GPU-enabled nodes in Kubernetes serve ML models for GenAI workloads including embedding generation, inference, and fine-tuning. This guide covers GPU scheduling, NVIDIA MIG (Multi-Instance GPU), and production model serving patterns for banking.

## GPU Node Configuration

```yaml
# Node with GPU labeled
apiVersion: v1
kind: Node
metadata:
  name: gpu-node-1
  labels:
    nvidia.com/gpu.present: "true"
    nvidia.com/gpu.product: NVIDIA-A100-SXM4-40GB
spec:
  # GPU capacity (auto-detected by NVIDIA device plugin)
  capacity:
    nvidia.com/gpu: "1"
```

## GPU Pod Scheduling

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: embedding-service
  namespace: banking-genai
spec:
  replicas: 2
  selector:
    matchLabels:
      app: embedding-service
  template:
    metadata:
      labels:
        app: embedding-service
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
          env:
            - name: MODEL_NAME
              value: "text-embedding-3-large"
            - name: BATCH_SIZE
              value: "100"
```

## MIG (Multi-Instance GPU)

```yaml
# Split A100 into 7 instances
apiVersion: apps/v1
kind: Deployment
metadata:
  name: genai-inference
spec:
  template:
    spec:
      containers:
        - name: inference
          image: quay.io/banking/genai-inference:1.0.0
          resources:
            limits:
              nvidia.com/mig-1g.5gb: 1  # 1 GPU instance with 5GB
            requests:
              cpu: "1"
              memory: 4Gi
```

## Cross-References

- **Autoscaling**: See [autoscaling.md](autoscaling.md) for GPU-aware autoscaling
- **Embedding Pipelines**: See [embedding-pipelines.md](../data-engineering/embedding-pipelines.md) for generation

## Interview Questions

1. **How do you schedule GPU workloads in Kubernetes?**
2. **What is NVIDIA MIG? How does it improve GPU utilization?**
3. **How do you autoscale GPU workloads based on inference demand?**
4. **What are the challenges of running ML models on Kubernetes?**
5. **How do you monitor GPU utilization and memory in K8s?**
6. **When would you use GPU vs CPU for embedding generation?**

## Checklist: GPU Workloads

- [ ] NVIDIA device plugin installed
- [ ] GPU node labels configured
- [ ] Tolerations for GPU taints
- [ ] Resource limits set for GPU requests
- [ ] MIG configured for multi-tenant GPU usage
- [ ] GPU monitoring enabled (DCGM exporter)
- [ ] GPU-aware autoscaling configured
- [ ] Model loading time accounted for in readiness probes
- [ ] GPU memory monitored for OOM prevention
