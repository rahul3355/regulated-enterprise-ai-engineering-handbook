# OpenAI Integration

This directory covers integrating OpenAI models (GPT-4o, GPT-4o-mini, embeddings) into enterprise banking GenAI platforms.

## Key Topics

- Model selection matrix for banking use cases
- Enterprise account setup and quota management
- Rate limiting and throughput optimization
- Azure OpenAI vs. OpenAI direct
- Data privacy and zero-retention policies
- Cost optimization with prompt caching
- Fine-tuning GPT models for banking tasks

## Model Selection

| Model | Best For | Cost (per 1M tokens) | Latency |
|-------|----------|---------------------|---------|
| GPT-4o | General purpose, customer-facing chat, summarization | $2.50 in / $10.00 out | ~1-3s |
| GPT-4o-mini | Classification, simple Q&A, high-volume tasks | $0.15 in / $0.60 out | ~0.5-1s |
| GPT-4 Turbo | Complex reasoning (legacy, prefer GPT-4o) | $10 in / $30 out | ~3-8s |
| text-embedding-3-large | High-quality embeddings for compliance/legal | $0.13 per 1K | ~100ms |
| text-embedding-3-small | Cost-effective embeddings for general search | $0.02 per 1K | ~50ms |

## Enterprise Considerations

- **Data residency**: Azure OpenAI offers regional deployment for data residency compliance
- **Zero data retention**: Configure API to not log requests (required for banking)
- **Private networking**: Use Azure Private Link for secure connectivity
- **Throughput**: Provisioned Throughput Units (PTUs) for guaranteed capacity

## Cross-References

- [../cost-optimization.md](../cost-optimization.md) — Cost management
- [../multi-model-architecture.md](../multi-model-architecture.md) — Multi-provider setup
- [../fine-tuning.md](../fine-tuning.md) — Fine-tuning GPT models
- [../embeddings.md](../embeddings.md) — Embedding model selection
