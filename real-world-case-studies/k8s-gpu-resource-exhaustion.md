# Case Study: GPU Resource Exhaustion in Kubernetes Model Serving

## Executive Summary

GPU resource exhaustion on the production Kubernetes cluster caused model serving degradation for 3 hours 45 minutes. A runaway fine-tuning job consumed all available GPU memory on shared inference nodes, causing OOM kills of production model serving pods. The incident affected 8,500 customer-facing AI queries and triggered a SEV-2 classification.

**Severity:** SEV-2 (Service Degradation)
**Duration:** 3 hours 45 minutes
**Impact:** 8,500 AI queries failed or were severely degraded
**Affected services:** AssistBot (degraded to fallback), WealthAdvisor AI (complete outage for 45 min)
**Financial impact:** £65,000 (compute waste, customer compensation)

---

## Background and Context

### The Infrastructure

The Kubernetes cluster served multiple workloads:
- **Production Inference Nodes** (4x NVIDIA A100 80GB): Model serving for customer-facing services
- **Training Nodes** (2x NVIDIA A100 80GB): Model fine-tuning and experimentation
- **Shared Nodes** (2x NVIDIA A100 80GB): Both inference and training (cost optimization)

### The Architecture

```
Model Serving Architecture:
├── Triton Inference Server (pods on production nodes)
│   ├── AssistBot model (BERT-based intent classifier)
│   ├── WealthAdvisor model (fine-tuned LLaMA-2-7B)
│   └── Embedding model (text-embedding-ada-002 proxy)
├── Training Jobs (pods on shared nodes)
│   ├── Fine-tuning jobs (LoRA adapters)
│   └── Experiment runs (hyperparameter tuning)
└── GPU Resource Management
    ├── GPU memory requests/limits configured
    ├── NVIDIA Device Plugin for GPU scheduling
    └── No GPU memory isolation between pods on shared nodes
```

### Resource Configuration (The Problem)

```yaml
# Production inference pod
resources:
  requests:
    nvidia.com/gpu: 1
    memory: 16Gi
  limits:
    nvidia.com/gpu: 1
    memory: 32Gi
    # No GPU memory limit!
    # nvidia.com/gpu.memory: NOT SET

# Training job pod (shared nodes)
resources:
  requests:
    nvidia.com/gpu: 1
    memory: 32Gi
  limits:
    nvidia.com/gpu: 1
    memory: 64Gi
    # No GPU memory limit!
    # Training can consume entire GPU
```

### The Critical Gap

Kubernetes does not natively support GPU memory limits. The NVIDIA Device Plugin allocates whole GPUs to pods but cannot enforce memory partitions. When a training pod and an inference pod share a GPU, the training job can consume all GPU memory, causing inference pods to OOM.

---

## Timeline of Events

```mermaid
timeline
    title GPU Resource Exhaustion Timeline
    section Pre-Incident
        Shared node setup : Training and inference<br/>pods share 2x A100 GPUs<br/>(cost optimization decision)
        : No GPU memory limits<br/>enforced (K8s limitation)
        : No monitoring of<br/>GPU memory utilization<br/>per pod
    section Incident Day
        09:00 : Fine-tuning job submitted<br/>for WealthAdvisor v2.1<br/>(LoRA adapter training)
        09:00:05 : Job scheduled on shared<br/>node GPU-1 (co-located<br/>with inference pod)
        09:15 : Training job begins<br/>loading dataset into<br/>GPU memory
        09:30 : GPU-1 memory at 65%<br/>Inference latency begins<br/>increasing
        09:45 : GPU-1 memory at 85%<br/>WealthAdvisor pod<br/>experiences memory pressure
        10:00 : GPU-1 memory at 95%<br/>WealthAdvisor pod OOM killed<br/>by Kubernetes
        10:00:05 : K8s restarts WealthAdvisor<br/>pod, but GPU memory<br/>still exhausted
        10:00:30 : Pod crashes again (OOM)<br/>CrashLoopBackOff begins
        10:01 : AssistBot on GPU-1<br/>degraded: latency<br/>increases from 120ms to 800ms
        10:05 : Alert: WealthAdvisor<br/>pod crash loop detected
        10:08 : Alert: AssistBot<br/>latency exceeds SLA
        10:15 : On-call engineer<br/>paged, investigation<br/>begins
        10:30 : Root cause identified:<br/>GPU memory exhaustion<br/>by training job
        10:35 : Mitigation: kill<br/>training job
        10:36 : Training job terminated,<br/>GPU memory freed
        10:38 : WealthAdvisor pod<br/>starts successfully
        10:42 : AssistBot latency<br/>returning to normal
        10:45 : Full service restoration<br/>Total duration: 3h 45m<br/>from job start to resolution
```

### GPU Memory Timeline

| Time | GPU-1 Memory | Event |
|------|-------------|-------|
| 09:00 | 42 GB / 80 GB | Normal: inference pods using 42 GB |
| 09:15 | 55 GB / 80 GB | Training job starts loading data |
| 09:30 | 68 GB / 80 GB | Inference latency increasing |
| 09:45 | 74 GB / 80 GB | WealthAdvisor memory pressure |
| 10:00 | 79 GB / 80 GB | WealthAdvisor OOM killed |
| 10:00-10:35 | 79 GB / 80 GB | CrashLoopBackOff, GPU fully exhausted |
| 10:36 | 42 GB / 80 GB | Training job killed, memory freed |
| 10:38 | 45 GB / 80 GB | WealthAdvisor pod starts |

---

## Root Cause Analysis

### Technical Root Causes

1. **Shared GPU Without Memory Isolation**
   - Training and inference pods shared GPUs on the "shared" node pool
   - Kubernetes cannot enforce GPU memory limits per pod
   - NVIDIA Device Plugin allocates whole GPUs, not memory partitions
   - No MPS (Multi-Process Service) or MIG (Multi-Instance GPU) configuration

2. **No GPU Memory Monitoring**
   - No per-pod GPU memory utilization metrics
   - No alerting on GPU memory approaching capacity
   - DCGM (Data Center GPU Manager) was deployed but not configured for per-pod monitoring

3. **Runaway Training Job**
   - The fine-tuning job loaded the entire dataset into GPU memory (no data streaming)
   - Batch size was not tuned for shared GPU environments
   - No memory ceiling configured in the training script

4. **Scheduling Policy Gap**
   - Training jobs were allowed to schedule on shared nodes
   - No taint/toleration separation between training and inference workloads
   - No priority-based preemption (inference should preempt training)

### Organizational Root Causes

1. **Cost Optimization Over Reliability**
   - Shared nodes were created to save costs (2 fewer A100 instances)
   - The reliability trade-off was not formally assessed
   - No capacity planning for GPU workloads

2. **Lack of GPU Expertise**
   - The team had Kubernetes expertise but limited GPU-specific operational knowledge
   - MIG and MPS were not understood or configured
   - GPU debugging tools (nsys, ncu) were not in the team's toolkit

3. **Missing GPU Runbook**
   - No runbook existed for GPU resource exhaustion scenarios
   - On-call engineers did not know how to diagnose GPU memory issues
   - No standard procedure for managing shared GPU workloads

---

## What Went Wrong Technically

### The Training Job Code

```python
# TRAINING SCRIPT (problematic - no memory management)
def train_lora_adapter():
    model = AutoModelForCausalLM.from_pretrained("llama-2-7b")
    model.cuda()  # Loads entire model to GPU (~14 GB for 7B model)

    # Loads ENTIRE dataset into GPU memory
    dataset = load_dataset("banking_finetuning")
    train_data = dataset["train"]  # 500K examples

    # No memory management
    trainer = Trainer(
        model=model,
        train_dataset=train_data,
        args=TrainingArguments(
            per_device_train_batch_size=32,  # Too large for shared GPU
            gradient_accumulation_steps=1,
            fp16=True,
        ),
    )
    trainer.train()  # GPU memory explodes
```

### The Fixed Training Script

```python
# TRAINING SCRIPT (fixed - with memory management)
def train_lora_adapter():
    model = AutoModelForCausalLM.from_pretrained(
        "llama-2-7b",
        load_in_8bit=True,  # Quantization: 14 GB -> 7 GB
        device_map="auto",
    )

    # Streaming dataset (doesn't load everything into memory)
    dataset = load_dataset("banking_finetuning", streaming=True)
    train_data = dataset["train"].take(10000)  # Batch streaming

    # Conservative batch size for shared GPU
    trainer = Trainer(
        model=model,
        train_dataset=train_data,
        args=TrainingArguments(
            per_device_train_batch_size=8,  # Reduced for shared GPU
            gradient_accumulation_steps=4,
            fp16=True,
            gradient_checkpointing=True,  # Trade compute for memory
        ),
    )
    trainer.train()
```

### The Missing Kubernetes Configuration

```yaml
# WHAT SHOULD HAVE BEEN CONFIGURED:

# 1. Separate node pools with taints
---
apiVersion: v1
kind: NodePool
metadata:
  name: inference-nodes
spec:
  taints:
    - key: workload
      value: inference
      effect: NoSchedule
  resources:
    nvidia.com/gpu: 4
    # Use MIG to partition GPUs
    nvidia.com/mig.strategy: mixed

---
apiVersion: v1
kind: NodePool
metadata:
  name: training-nodes
spec:
  taints:
    - key: workload
      value: training
      effect: NoSchedule

# 2. Pod priority and preemption
---
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: inference-critical
value: 1000000
globalDefault: false
description: "Production inference pods should not be evicted"

---
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: training-best-effort
value: 100000
globalDefault: false
description: "Training jobs can be preempted"

# 3. GPU memory monitoring via DCGM exporter
---
apiVersion: monitoring.coreos.com/v1
kind: PodMonitor
metadata:
  name: dcgm-exporter
spec:
  selector:
    matchLabels:
      app: dcgm-exporter
  metrics:
    - path: /metrics
      port: 9400
      interval: 15s
```

---

## What Went Wrong Organizationally

1. **Cost-Driven Architecture Without Reliability Assessment**: The shared node decision was made to save £8,000/month but introduced a SEV-2 risk that was not assessed.

2. **GPU Skills Gap**: The team operated GPU workloads without GPU-specific operational expertise.

3. **Missing Capacity Planning**: No capacity planning process existed for GPU workloads. Training jobs were submitted ad-hoc.

4. **No GPU Runbook**: On-call engineers had no guidance for diagnosing and resolving GPU resource issues.

---

## Immediate Response and Mitigation

### First Hour

1. **Training Job Killed**: The fine-tuning job was terminated at 10:35
2. **Service Restoration**: WealthAdvisor and AssistBot recovered within 5 minutes
3. **Impact Assessment**: 8,500 queries identified as affected
4. **Temporary Fix**: Training jobs banned from shared nodes until proper isolation was implemented

### Days 1-7

1. **Node Pool Separation**: Dedicated training node pool created (cost: +£8,000/month)
2. **MIG Configuration**: Production inference GPUs partitioned using MIG (2x 40GB instances per A100)
3. **GPU Monitoring**: DCGM exporter configured with per-pod GPU memory metrics
4. **Priority Classes**: Inference pods assigned higher priority than training pods

### Cost vs. Reliability Decision

The incident forced a re-evaluation:

| Approach | Monthly Cost | Reliability Risk |
|----------|-------------|------------------|
| Shared nodes (before) | £32,000 | HIGH (this incident) |
| Separate nodes (after) | £40,000 | LOW |
| MIG partitioning | £32,000 | MEDIUM (some risk remains) |
| Separate + MIG | £40,000 | LOWEST |

Decision: Separate nodes + MIG for production inference = £40,000/month

---

## Long-Term Fixes and Systemic Changes

### Technical Fixes

1. **GPU Node Pool Separation**
   - Inference-only nodes: 4x A100 with MIG (8x 40GB instances)
   - Training-only nodes: 2x A100 (full 80GB for training jobs)
   - Taint/toleration enforcement prevents cross-scheduling

2. **GPU Memory Monitoring**
   ```yaml
   # DCGM exporter alerts
   alerts:
     - name: HighGPUMemoryUsage
       condition: dcgm_fb_free / dcgm_fb_total < 0.15
       severity: warning
       action: Page on-call

     - name: GPUMemoryExhaustionImminent
       condition: dcgm_fb_free / dcgm_fb_total < 0.05
       severity: critical
       action: Page on-call + auto-kill lowest priority pod
   ```

3. **Training Job Resource Controls**
   - Maximum GPU memory per training job enforced in training scripts
   - Dataset streaming required (no full dataset loading)
   - Batch size limits for shared GPU environments
   - Automatic job termination if GPU memory exceeds 80%

4. **Pod Priority and Preemption**
   - Inference pods: Priority 1,000,000 (cannot be preempted)
   - Training pods: Priority 100,000 (can be preempted by inference)
   - Automatic eviction of training pods if inference needs GPU resources

### Process Changes

1. **GPU Capacity Planning**: All GPU workloads require capacity planning before scheduling
2. **Training Job Approval**: Training jobs on production clusters require engineering lead approval
3. **GPU Runbook**: Comprehensive runbook for GPU resource issues
4. **GPU On-Call Training**: All on-call engineers trained on GPU debugging and troubleshooting
5. **Quarterly GPU Capacity Review**: Regular review of GPU utilization and capacity planning

### Cultural Changes

1. **Reliability Over Cost for Production**: Production inference workloads never share resources with training
2. **GPU as a First-Class Resource**: GPU memory is monitored and managed like CPU and memory
3. **Preemptibility Design**: Non-critical workloads (training) are designed to be preemptable

---

## Lessons Learned

1. **Shared GPUs Without Isolation Are Dangerous**: Kubernetes cannot enforce GPU memory limits. Shared GPUs require MIG or MPS for isolation.

2. **GPU Monitoring Is Different**: GPU memory, utilization, and temperature require specialized monitoring tools (DCGM).

3. **Training Jobs Are Noisy Neighbors**: Training jobs consume all available GPU memory. They must be isolated from inference.

4. **Cost Savings Can Be False Economies**: Saving £8,000/month on shared nodes cost £65,000 in one incident plus £8,000/month in remediation.

5. **On-Call Engineers Need GPU Skills**: GPU debugging requires different tools and knowledge than CPU-based debugging.

6. **Priority and Preemption Matter**: Production inference should always be able to preempt training jobs.

---

## Interview Questions Derived From This Case Study

1. **Kubernetes**: "How do you manage GPU resources in Kubernetes? What are the limitations of the NVIDIA Device Plugin?"

2. **System Design**: "Design a GPU-sharing strategy for a Kubernetes cluster running both inference and training workloads."

3. **Incident Response**: "Your model serving pods are crashing with OOM errors. How do you diagnose and resolve the issue?"

4. **MLOps**: "How do you prevent training jobs from impacting production inference in a shared GPU environment?"

5. **Operations**: "What monitoring would you set up for GPU-based model serving? What alerts would you configure?"

6. **Architecture**: "Compare MIG vs. MPS for GPU partitioning in Kubernetes. When would you use each?"

---

## Cross-References

- See `../kubernetes-openshift/gpu-management.md` for Kubernetes GPU management patterns
- See `../infrastructure/compute.md` for GPU compute options
- See `../incident-management/triage-playbooks.md` for GPU troubleshooting playbooks
- See `../mlops/training-infrastructure.md` for ML training infrastructure best practices
- See `../engineering-philosophy/reliability-vs-cost.md` for reliability vs. cost trade-offs
