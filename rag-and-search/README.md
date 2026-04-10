# RAG (Retrieval-Augmented Generation)

## Overview

Retrieval-Augmented Generation (RAG) is an architecture pattern that combines information retrieval with large language models to produce accurate, grounded, and up-to-date responses. Instead of relying solely on a model's parametric knowledge (which is frozen at training time), RAG fetches relevant documents from an external knowledge base and injects them into the prompt as context.

```
User Query -> [Retriever] -> Relevant Documents -> [LLM + Context] -> Grounded Response
```

In a banking environment, RAG is the dominant pattern for building AI assistants because:
- **Accuracy is non-negotiable**: Financial advice and compliance answers must be factually correct
- **Regulations change frequently**: Fine-tuned models become stale; RAG pulls from live documents
- **Auditability is required**: Every response can be traced back to source documents
- **Data privacy**: Sensitive banking data never leaves your infrastructure

## RAG vs. Fine-Tuning: When to Use Each

| Criterion | RAG | Fine-Tuning |
|---|---|---|
| **Knowledge freshness** | Real-time (update index) | Requires re-training |
| **Factual accuracy** | High (grounded in sources) | Varies (hallucination risk) |
| **Traceability** | Full citation support | No source attribution |
| **Cost at scale** | Lower (no training costs) | Higher (GPU training runs) |
| **Response latency** | Higher (retrieval + generation) | Lower (generation only) |
| **Style adaptation** | Limited (prompt-based) | Excellent (learned patterns) |
| **Data privacy** | Easy (local retrieval) | Training data exposure risk |
| **Regulatory compliance** | Strong (auditable sources) | Weak (black box) |

**Rule of thumb for banking**: Use RAG for knowledge-intensive tasks (policy Q&A, compliance lookup, product information). Use fine-tuning for style/format adaptation (generating emails in the bank's tone, classifying intents).

**Hybrid approach**: Many enterprise systems use fine-tuning for the *generation* layer (learning the bank's writing style) while using RAG for the *knowledge* layer (factual content from documents).

## RAG Architecture

```mermaid
flowchart LR
    subgraph Ingestion Pipeline
        A[Documents] --> B[Parser/OCR]
        B --> C[Chunker]
        C --> D[Embedding Model]
        D --> E[(Vector DB)]
        C --> F[(Metadata Store)]
    end

    subgraph Query Pipeline
        Q[User Query] --> G[Query Rewriter]
        G --> H[Embedding Model]
        H --> I[Retriever]
        I --> E
        I --> J[Re-ranker]
        J --> K[Top-K Documents]
        K --> L[Context Assembler]
        L --> M[LLM]
        M --> N[Response + Citations]
    end

    F -. Metadata .-> I
```

### Core Components

**1. Document Ingestion Pipeline**
- Parse PDFs, Word docs, HTML, databases
- Extract text, tables, images (OCR when needed)
- Chunk into manageable segments (200-1000 tokens)
- Generate embeddings for each chunk
- Store in vector database with metadata

**2. Retrieval Pipeline**
- Receive and optionally rewrite user query
- Generate query embedding
- Search vector database for top-N candidates
- Re-rank results for relevance
- Select top-K for context window

**3. Generation Pipeline**
- Assemble prompt with system instructions + retrieved context + query
- Call LLM (OpenAI, Claude, Llama, Mistral, etc.)
- Post-process response
- Attach citations/sources
- Return to user

## Key Design Decisions

### Chunk Size
- **Small chunks (100-200 tokens)**: Precise retrieval but may lose context
- **Medium chunks (200-500 tokens)**: Good balance for most use cases
- **Large chunks (500-1000 tokens)**: Richer context but may introduce noise
- **Banking recommendation**: 300-500 tokens with 10-15% overlap for policy documents

### Embedding Model
- **OpenAI text-embedding-3-small**: Good quality, $0.02/M tokens
- **OpenAI text-embedding-3-large**: Best quality, $0.13/M tokens
- **Cohere embed-v3**: Multilingual, $0.10/M tokens
- **Open source (E5, BGE, GTE)**: Self-hosted, no API cost
- **Banking recommendation**: E5-large or BGE-M3 for self-hosted; text-embedding-3-large for cloud

### Retrieval Strategy
- **Simple top-K**: Baseline approach
- **Hybrid search**: BM25 + vector search (recommended for production)
- **Multi-query**: Expand query, retrieve from multiple angles
- **Hierarchical**: Summary-level then detailed-level retrieval

### Vector Database Selection
- **pgvector**: Best for teams already using PostgreSQL, cost-effective
- **Pinecone**: Managed, fastest to deploy, higher cost
- **Milvus**: Open source, scales well, complex operations
- **Weaviate**: Rich features, hybrid search built-in

## Banking-Specific Considerations

1. **Access Control**: Different employees see different documents. Retrieval must respect RBAC.
2. **Audit Trail**: Every query and response must be logged for compliance.
3. **Document Versioning**: Policies change; must track which version was used for each answer.
4. **Multilingual**: Global banks need RAG that works across languages.
5. **PII Redaction**: Queries may contain customer data; must redact before processing.
6. **Response Grounding**: Every answer must cite specific policy sections.

## Quick Start Example

```python
from langchain_community.vectorstores import PGVector
from langchain_openai import OpenAIEmbeddings, ChatOpenAI
from langchain_core.prompts import ChatPromptTemplate

# Setup
embeddings = OpenAIEmbeddings(model="text-embedding-3-large")
llm = ChatOpenAI(model="gpt-4o-mini")

# Vector store (already populated)
vectorstore = PGVector(
    connection_string="postgresql://user:pass@localhost:5432/bank_knowledge",
    embedding_function=embeddings,
    collection_name="policies"
)

# Retrieval + Generation
def rag_query(query: str) -> str:
    # Retrieve
    docs = vectorstore.similarity_search(query, k=4)
    context = "\n\n".join([doc.page_content for doc in docs])
    
    # Generate
    prompt = ChatPromptTemplate.from_messages([
        ("system", "You are a banking policy assistant. Answer using ONLY the provided context. If the context doesn't contain the answer, say so. Always cite your sources."),
        ("human", "Context:\n{context}\n\nQuestion: {question}")
    ])
    
    chain = prompt | llm
    return chain.invoke({"context": context, "question": query})
```

## Performance Benchmarks (Typical Production RAG)

| Metric | Target | Notes |
|---|---|---|
| Retrieval latency | < 200ms | P95, with 100K+ documents |
| Generation latency | < 2000ms | P95, depends on response length |
| End-to-end latency | < 3000ms | P95 total |
| Retrieval precision@5 | > 0.70 | Top 5 relevant docs |
| Groundedness score | > 0.85 | Fraction of claims backed by sources |
| Hallucination rate | < 5% | Measured on golden dataset |

## Files in This Module

| File | Topic |
|---|---|
| [chunking.md](chunking.md) | Chunking strategies and best practices |
| [embedding-models.md](embedding-models.md) | Model selection and comparison |
| [retrieval.md](retrieval.md) | Retrieval strategies and k-selection |
| [reranking.md](reranking.md) | Re-ranking with cross-encoders |
| [metadata-filtering.md](metadata-filtering.md) | Metadata-based filtering |
| [hybrid-search.md](hybrid-search.md) | BM25 + vector search combination |
| [vector-databases.md](vector-databases.md) | Database comparison and selection |
| [pgvector.md](pgvector.md) | pgvector deep-dive |
| [query-rewriting.md](query-rewriting.md) | Query expansion and rewriting |
| [citations.md](citations.md) | Citation and source attribution |
| [hallucination-reduction.md](hallucination-reduction.md) | Reducing hallucinations |
| [evaluation.md](evaluation.md) | RAG evaluation framework |
| [groundedness.md](groundedness.md) | Measuring groundedness |
| [retrieval-metrics.md](retrieval-metrics.md) | Precision, recall, NDCG |
| [latency-optimization.md](latency-optimization.md) | Latency optimization |
| [cost-optimization.md](cost-optimization.md) | Cost optimization |
| [freshness.md](freshness.md) | Document freshness strategies |
| [data-ingestion.md](data-ingestion.md) | Ingestion pipeline |
| [ocr-and-pdf-parsing.md](ocr-and-pdf-parsing.md) | OCR and PDF parsing |
| [incremental-indexing.md](incremental-indexing.md) | Incremental index updates |
| [feedback-loops.md](feedback-loops.md) | User feedback integration |
| [human-review.md](human-review.md) | Human review workflows |
| [production-incidents.md](production-incidents.md) | Real incidents and lessons learned |
