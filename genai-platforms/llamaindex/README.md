# LlamaIndex for RAG

This directory covers using LlamaIndex for building production RAG (Retrieval-Augmented Generation) systems in a banking environment.

## Key Topics

- LlamaIndex data connectors for banking data sources
- Document parsing and chunking strategies
- Query pipelines for complex retrieval
- Index structures (vector, tree, keyword, summary)
- When to use LlamaIndex vs. LangChain vs. custom RAG
- Production performance optimization

## Banking Data Connectors

LlamaIndex provides connectors for common banking data sources:

| Data Source | Connector | Use Case |
|-------------|-----------|----------|
| SharePoint/OneDrive | SharePointLoader | Policy documents, procedures |
| Confluence | ConfluenceLoader | Internal knowledge base |
| Jira | JiraReader | Issue tracking, compliance tickets |
| SQL Database | SQLDatabase | Customer data, transaction metadata |
| S3/MinIO | SimpleDirectoryReader | Document repository |
| APIs | APIRetriever | External regulatory data |

## Query Pipeline Example

```python
from llama_index.core.query_pipeline import QueryPipeline
from llama_index.core.indices.query.query_transform import HyDEQueryTransform

# Banking regulatory search pipeline
pipeline = QueryPipeline(chain=[
    HyDEQueryTransform(),          # Hypothetical document embedding
    VectorIndexRetriever(top_k=5), # Semantic retrieval
    CohereReranker(top_n=3),       # Reranking for relevance
    PromptTemplate(),              # Assemble prompt with context
    LLM(),                         # Generate response
    OutputValidator(),             # Validate output format
])
```

## Cross-References

- [../rag-and-search/](../rag-and-search/) — RAG architecture patterns
- [../embeddings.md](../embeddings.md) — Embedding model selection
- [../vector-databases/](../vector-databases/) — Vector database integration
- [../caching.md](../caching.md) — Caching retrieval results
