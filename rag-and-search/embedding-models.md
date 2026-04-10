# Embedding Models

## What Are Embeddings?

Embeddings are dense vector representations of text that capture semantic meaning. Similar texts produce similar vectors (smaller distance in vector space). They are the foundation of retrieval in RAG systems.

```
"The interest rate is 5%"     -> [0.12, -0.34, 0.56, ...]  (1536 dimensions)
"Rate of interest: 5 percent" -> [0.14, -0.31, 0.58, ...]  (very similar vector)
"The sky is blue"             -> [-0.45, 0.78, -0.12, ...] (very different vector)
```

In banking RAG, embeddings enable semantic search: a query about "loan fees" can retrieve documents about "processing charges" even though the words don't overlap.

## Key Embedding Model Properties

| Property | Description | Why It Matters |
|---|---|---|
| **Dimension size** | Vector length (256-4096) | Higher = more nuanced but more storage |
| **Max input tokens** | Max tokens per chunk | Limits chunk size |
| **Multilingual** | Supports multiple languages | Required for global banks |
| **Latency** | Time to embed one chunk | Affects ingestion speed |
| **Cost** | Per-token or self-hosted cost | Total cost of ownership |
| **MTEB score** | Benchmark ranking | Quality indicator |

## Model Comparison (2024-2025)

### Cloud API Models

| Model | Dimensions | Max Tokens | MTEB Score | Cost ($/M tokens) | Multilingual | Notes |
|---|---|---|---|---|---|---|
| OpenAI text-embedding-3-small | 1536 | 8191 | 62.3 | $0.02 | No | Best price/performance |
| OpenAI text-embedding-3-large | 3072 | 8191 | 64.6 | $0.13 | No | Highest quality OpenAI |
| Cohere embed-v3 | 1024 | 4000 | 65.4 | $0.10 | Yes (50+) | Best multilingual |
| Cohere embed-english-v3 | 1024 | 4000 | 63.2 | $0.10 | No | English only, cheaper |
| Google text-embedding-004 | 768 | 2048 | 62.8 | $0.025 | Yes | Good multilingual |

### Open Source (Self-Hosted) Models

| Model | Dimensions | Max Tokens | MTEB Score | Inference Cost | Notes |
|---|---|---|---|---|---|
| BGE-M3 | 1024 | 8192 | 65.0 | GPU required | Best open source, multilingual |
| E5-large-v2 | 1024 | 512 | 63.5 | GPU required | Strong English performance |
| GTE-Qwen2-7B | 3584 | 8192 | 72.0 | GPU (7B) | State-of-the-art, heavy |
| Jina-embeddings-v3 | 1024 | 8192 | 64.2 | GPU required | Multilingual, good quality |
| nomic-embed-text | 768 | 8192 | 59.8 | CPU possible | Lightweight, good for edge |

### Banking Recommendation Matrix

| Scenario | Recommended Model | Rationale |
|---|---|---|
| Cloud-first, English only | OpenAI text-embedding-3-large | Best quality, simple integration |
| Cloud-first, multilingual | Cohere embed-v3 | 50+ languages, strong quality |
| Self-hosted, English only | E5-large-v2 or BGE-large | Good quality, manageable size |
| Self-hosted, multilingual | BGE-M3 | Best multilingual open source |
| Cost-constrained | OpenAI text-embedding-3-small | $0.02/M, good quality |
| Edge/on-device | nomic-embed-text | Runs on CPU, small model |
| Maximum quality (budget unlimited) | GTE-Qwen2-7B | SOTA on MTEB |

## Cost Analysis

### Example: 100,000 Banking Documents

Assume each document averages 2000 tokens, chunked at 400 tokens = 5 chunks per document = 500,000 total chunks.

| Model | Ingestion Cost (500K chunks) | Query Cost (10K queries/day) | Monthly Query Cost |
|---|---|---|---|
| OpenAI text-embedding-3-small | $0.10 | $0.02 | $0.60 |
| OpenAI text-embedding-3-large | $0.65 | $0.13 | $3.90 |
| Cohere embed-v3 | $0.50 | $0.10 | $3.00 |
| BGE-M3 (self-hosted) | ~$50 (GPU hours) | ~$5 (GPU hours) | ~$150 |
| nomic-embed-text (CPU) | ~$5 (CPU hours) | ~$1 (CPU hours) | ~$30 |

**Key insight**: For high query volumes, self-hosted models become cost-effective. For low volume, API models are cheaper. Break-even for BGE-M3 vs OpenAI-small is around 2M queries/month.

## Performance Benchmarks

### Embedding Latency

| Model | Latency per chunk | Throughput (chunks/sec) | Hardware |
|---|---|---|---|
| OpenAI text-embedding-3-small | 50-200ms (API) | ~10-20 (rate limited) | API |
| OpenAI text-embedding-3-large | 100-300ms (API) | ~5-10 (rate limited) | API |
| BGE-large (ONNX) | 5-15ms | 200-500 | A10 GPU |
| E5-large (ONNX) | 8-20ms | 150-400 | A10 GPU |
| nomic-embed-text | 10-30ms | 100-250 | CPU (8 cores) |

### Storage Requirements

| Model | Dimensions | Bytes per vector (FP32) | 500K vectors | 5M vectors |
|---|---|---|---|---|
| text-embedding-3-small | 1536 | 6,144 | 3.0 GB | 30 GB |
| text-embedding-3-large | 3072 | 12,288 | 6.0 GB | 60 GB |
| BGE-M3 | 1024 | 4,096 | 2.0 GB | 20 GB |
| nomic-embed-text | 768 | 3,072 | 1.5 GB | 15 GB |

With vector index overhead (HNSW), multiply by 1.5-2x.

## Dimensionality Reduction

If storage is a concern, you can reduce embedding dimensions:

```python
import numpy as np
from sklearn.decomposition import PCA

# Original embeddings: 500K x 3072 (OpenAI large)
original = np.array(embeddings)

# Reduce to 768 dimensions
pca = PCA(n_components=768)
reduced = pca.fit_transform(original)

# Test quality impact
# Expected: 2-5% retrieval quality loss for 4x storage reduction
```

**Tradeoffs**:
- 3072 -> 1536: ~1% quality loss, 50% storage savings
- 3072 -> 768: ~3-5% quality loss, 75% storage savings
- 3072 -> 256: ~10-15% quality loss, 92% storage savings

**Banking recommendation**: Do not reduce below 512 dimensions for production banking RAG. The quality impact on retrieval precision is not acceptable for compliance-critical applications.

## Selecting Embedding Models: Decision Framework

```
Is multilingual support needed?
├── Yes -> Cohere embed-v3 (cloud) or BGE-M3 (self-hosted)
└── No -> Continue

Is there a budget constraint?
├── Yes -> OpenAI text-embedding-3-small ($0.02/M)
└── No -> Continue

Is query volume > 2M/month?
├── Yes -> Self-hosted (BGE-large or E5-large)
└── No -> Continue

Is maximum quality required?
├── Yes -> OpenAI text-embedding-3-large or GTE-Qwen2-7B
└── No -> OpenAI text-embedding-3-small (best value)
```

## Banking-Specific Evaluation

Always evaluate embedding models on your own data, not just MTEB benchmarks.

### Custom Evaluation Pipeline

```python
def evaluate_embedding_model(model_name, golden_queries):
    """Test embedding model on banking-specific golden dataset."""
    from sentence_transformers import util
    
    results = {
        "model": model_name,
        "precision_at_5": [],
        "precision_at_10": [],
        "mrr": [],
        "latency_ms": []
    }
    
    for query in golden_queries:
        # Embed query
        start = time.time()
        query_emb = embed(query.text, model_name)
        latency = (time.time() - start) * 1000
        results["latency_ms"].append(latency)
        
        # Search
        scores = []
        for doc_emb in all_doc_embeddings:
            sim = util.cos_sim(query_emb, doc_emb).item()
            scores.append((sim, doc_metadata))
        
        # Sort and compute metrics
        scores.sort(reverse=True, key=lambda x: x[0])
        top_10 = scores[:10]
        
        # Check relevance
        relevant_in_5 = sum(
            1 for _, meta in top_10[:5] 
            if meta["doc_id"] in query.relevant_docs
        )
        results["precision_at_5"].append(relevant_in_5 / 5)
        
        relevant_in_10 = sum(
            1 for _, meta in top_10[:10] 
            if meta["doc_id"] in query.relevant_docs
        )
        results["precision_at_10"].append(relevant_in_10 / 10)
        
        # MRR
        for i, (_, meta) in enumerate(top_10):
            if meta["doc_id"] in query.relevant_docs:
                results["mrr"].append(1.0 / (i + 1))
                break
    
    return {k: np.mean(v) for k, v in results.items()}
```

### Banking Domain-Specific Challenges

Embedding models can struggle with:
1. **Numbers and percentages**: "5% interest" vs "five percent interest"
2. **Abbreviations**: "KYC" vs "Know Your Customer"
3. **Negations**: "not eligible" vs "eligible" (should be different!)
4. **Domain jargon**: "amortization schedule", "collateral coverage ratio"

**Mitigation**: Include domain-specific examples in your evaluation. If a model fails on banking terms, consider fine-tuning.

## Fine-Tuning Embeddings

For specialized banking domains, you can fine-tune embedding models:

```python
from sentence_transformers import SentenceTransformer, losses
from torch.utils.data import DataLoader

# Load pre-trained model
model = SentenceTransformer('BAAI/bge-large-en')

# Training pairs: (anchor, positive, negative)
train_examples = [
    InputExample(texts=["What is the loan processing fee?", 
                        "Processing fee is 1.5% of loan amount", 
                        "The weather is nice today"]),
    InputExample(texts=["KYC requirements", 
                        "Know Your Customer documentation needed", 
                        "How to cook pasta"]),
]

train_dataloader = DataLoader(train_examples, batch_size=16)
train_loss = losses.TripletLoss(model)

model.fit(
    train_objectives=[(train_dataloader, train_loss)],
    epochs=3,
    warmup_steps=100,
    output_path="./banking-embeddings"
)
```

**When to fine-tune**:
- You have 1000+ labeled query-document pairs
- General models fail on banking-specific terminology
- Retrieval precision is below target (< 60% P@5)

**When NOT to fine-tune**:
- You have fewer than 500 training pairs
- General model already achieves > 70% P@5
- You lack engineering resources for model maintenance

## Best Practices

1. **Benchmark before choosing**: Run your golden dataset against 3 candidate models
2. **Monitor embedding drift**: Re-evaluate quarterly as documents change
3. **Batch embeddings**: Send chunks in batches of 100-1000 for efficiency
4. **Cache embeddings**: Never re-embed the same content; store in vector DB
5. **Version your embeddings**: Track which model version was used for each document
6. **Test with banking data**: MTEB scores don't reflect banking domain performance
7. **Plan for dimension changes**: Switching models means re-indexing everything
