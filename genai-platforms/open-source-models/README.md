# Open Source Models

This directory covers deploying and operating open source models (Llama 3, Mistral, etc.) in a banking environment, with emphasis on data residency compliance and self-hosted infrastructure.

## Key Topics

- Llama 3 model family (8B, 70B, 405B) deployment
- Mistral model family deployment
- Self-hosted inference with vLLM, TGI, TRT-LLM
- Data residency and sovereignty compliance
- GPU infrastructure planning
- Fine-tuning open source models for banking
- When to use open source vs. commercial APIs

## Model Options

| Model | Parameters | Context | Best Use Case | GPU Requirement |
|-------|-----------|---------|---------------|----------------|
| Llama 3 8B | 8B | 8K-128K | Simple classification, internal search | 1x A10G / 1x L4 |
| Llama 3 70B | 70B | 8K-128K | General tasks, code, analysis | 2-4x A100 80GB |
| Llama 3 405B | 405B | 8K-128K | Highest quality self-hosted | 8-16x H100 |
| Mistral 7B | 7B | 32K | Fast, efficient tasks | 1x A10G |
| Mixtral 8x7B | 47B | 32K | MoE efficiency | 2x A100 |
| Mixtral 8x22B | 141B | 64K | Complex tasks, MoE | 4-8x A100 |

## Banking Rationale

1. **Data residency**: Models run on bank infrastructure — data never leaves premises
2. **Cost at scale**: Self-hosted becomes cheaper at very high volumes
3. **Customization**: Fine-tune on bank-specific data without API provider restrictions
4. **Availability**: No dependency on external provider uptime
5. **Regulatory comfort**: Full control over model behavior and data handling

## Infrastructure Requirements

```
Minimum production setup (Llama 3 70B):
- 4x NVIDIA A100 80GB GPUs
- vLLM for high-throughput serving
- Kubernetes for orchestration
- 200+ GB system RAM
- 10 Gbps+ network

Estimated cost: $80,000-150,000/year (hardware + ops)
```

## Cross-References

- [../fine-tuning.md](../fine-tuning.md) — Fine-tuning open source models
- [../cost-optimization.md](../cost-optimization.md) — Self-hosted vs. API cost analysis
- [../multi-model-architecture.md](../multi-model-architecture.md) — Self-hosted as fallback provider
- [../infrastructure/](../infrastructure/) — GPU infrastructure planning
- [../kubernetes-openshift/](../kubernetes-openshift/) — Kubernetes deployment
