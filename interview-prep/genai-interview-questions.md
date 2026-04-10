# GenAI Interview Questions (40+)

## Foundational Questions

### Q1: What is RAG and why is it preferred over fine-tuning for banking applications?

**Strong Answer**: "RAG (Retrieval-Augmented Generation) combines information retrieval with LLM generation. Instead of relying on the model's frozen parametric knowledge, RAG fetches relevant documents from an external knowledge base and provides them as context. For banking, RAG is preferred because: (1) **Accuracy** -- answers are grounded in specific source documents, reducing hallucination risk. (2) **Freshness** -- when policies change, you update the index, not retrain a model. (3) **Auditability** -- every response cites specific sources, which is required for compliance. (4) **Data privacy** -- sensitive banking data stays in your infrastructure; only the query goes to the LLM. (5) **Cost** -- no expensive GPU training runs. Fine-tuning is better suited for style adaptation (learning the bank's tone) rather than knowledge injection."

### Q2: Explain the difference between bi-encoders and cross-encoders.

**Strong Answer**: "Bi-encoders process queries and documents independently, producing fixed embeddings that can be pre-computed and compared efficiently. They scale to millions of documents but are less accurate because they can't model query-document interactions. Cross-encoders process the query and document together through a single transformer, capturing fine-grained interactions. They're much more accurate but must process each query-document pair, making them too slow for large-scale retrieval. In practice, bi-encoders are used for first-stage retrieval (find top-50 candidates) and cross-encoders for re-ranking (score the top-50 to select the top-5)."

### Q3: What is the 'lost in the middle' problem and how do you address it?

**Strong Answer**: "Research shows that LLMs pay most attention to information at the beginning and end of their context window, and less to information in the middle. When you stuff 10 retrieved documents into a prompt, the model may miss critical information buried in documents 5-7. Solutions: (1) Limit context to the most relevant 3-5 documents. (2) Arrange documents strategically -- most relevant first and last, less relevant in the middle. (3) Use structured prompts with clear source boundaries so the model can parse the context more effectively. (4) For critical information, repeat it in the system prompt or as a dedicated 'key facts' section."

### Q4: How do you measure RAG quality?

**Strong Answer**: "I measure at three levels: (1) **Retrieval quality** -- Precision@K, Recall@K, MRR, NDCG on a golden dataset. These tell me if the retriever finds the right documents. (2) **Generation quality** -- Faithfulness (are claims supported by context?), Answer Relevance (does the answer address the question?), and Groundedness (overall grounding score). I use RAGAS or LLM-as-judge for these. (3) **End-to-end quality** -- Answer accuracy against expected answers, hallucination rate, user satisfaction scores, and task completion rate. I also track operational metrics: latency, cost per query, and cache hit rate."

### Q5: What strategies reduce hallucination in RAG?

**Strong Answer**: "Multiple layers: (1) **System prompt**: Explicit instructions to use only provided context, say 'I don't know' when uncertain. (2) **Temperature**: Set to 0.0 for factual responses. (3) **Source boundaries**: Clear `[BEGIN DOC]` / `[END DOC]` markers. (4) **Citation enforcement**: Require inline citations for every factual claim. (5) **Retrieval quality gate**: Don't generate if retrieval scores are too low. (6) **Post-generation verification**: NLI models to check if claims are entailed by context. (7) **Number validation**: Verify all numbers in the response appear in the context. (8) **Self-consistency**: Ask the model to fact-check its own answer. In banking, I target < 5% hallucination rate."

## Intermediate Questions

### Q6: How do you handle access control in a multi-tenant RAG system?

**Strong Answer**: "Access control must be enforced at retrieval time, not after. I implement: (1) **Database-level filtering**: Every vector search includes metadata filters (tenant_id, department, clearance_level). In pgvector, this is a WHERE clause before the ORDER BY distance. (2) **Row-level security**: PostgreSQL RLS policies as a safety net. (3) **Cache isolation**: Cache keys include tenant_id to prevent cross-tenant cache hits. (4) **Response validation**: After generation, verify all cited sources belong to the user's tenant. (5) **Defense in depth**: Each layer independently enforces access control so a bug in one doesn't cause data leakage."

### Q7: When would you use query rewriting vs. direct retrieval?

**Strong Answer**: "I use query rewriting when: (1) The query is ambiguous or too brief ('loan fees' -- which loan?). (2) The query is multi-part and needs decomposition ('what are the fees AND eligibility criteria?'). (3) The vocabulary in the query doesn't match the documents. I skip rewriting for: (1) Simple, well-formed factual queries. (2) When latency budget is tight (< 500ms). (3) When retrieval precision is already > 80% without rewriting. The key is query classification first -- route simple queries directly, rewrite complex ones."

### Q8: How do you decide on chunk size?

**Strong Answer**: "It depends on the document type and retrieval needs. Small chunks (100-200 tokens) give precise retrieval but may lose context. Large chunks (500-1000 tokens) provide richer context but add noise. My approach: (1) Start with 300-500 tokens and 15% overlap as a baseline. (2) Evaluate on a golden dataset -- test chunk sizes of 200, 400, 600, and 800 tokens. (3) Choose the size that maximizes retrieval precision. (4) For banking policies, I typically find 300-400 tokens works best because policy sections are often self-contained paragraphs. I also use document-type specific chunking: legal contracts by clause, tables separately, FAQs by Q&A pair."

### Q9: Explain hybrid search and why it's better than pure vector search.

**Strong Answer**: "Hybrid search combines BM25 (keyword matching) with vector search (semantic matching). BM25 catches exact term matches that embeddings might miss -- product codes, regulation names, specific terminology. Vector search catches semantic matches that keyword search misses -- 'processing fee' vs 'processing charges.' Combined, they achieve 10-25% higher retrieval precision than either alone. The key is score normalization (BM25 and vector scores are on different scales) and alpha tuning (the weight given to each component). For banking, alpha of 0.3-0.4 typically works best, favoring semantic search slightly while still catching exact terms."

### Q10: How do you handle document freshness in RAG?

**Strong Answer**: "Banking policies change frequently, so stale documents are a real risk. My approach: (1) **Content hashing**: Track document hashes to detect changes at the byte level. (2) **Incremental indexing**: When a document changes, delete its old chunks and re-index. (3) **Metadata tracking**: Every chunk has effective_from, effective_until, and status fields. (4) **Freshness filtering**: Retrieval prefers active, current documents. (5) **Webhook-based updates**: When the CMS publishes a change, trigger immediate re-indexing. (6) **Freshness monitoring**: Alert when any document category hasn't been updated in the index within the expected SLA. (7) **Full rebuilds weekly** to clean up accumulated inconsistencies."

## Advanced Questions

### Q11: How would you design a RAG system that serves 50,000 users with 500K queries/day?

**Strong Answer**: "Architecture: (1) **API Gateway** with rate limiting and authentication. (2) **Redis cache** for common queries -- target 30%+ cache hit rate. (3) **Retrieval**: pgvector with HNSW index, partitioned by department. Read replicas for query load. (4) **Re-ranker**: Self-hosted BGE-reranker on GPU, load-balanced across 2-3 instances. (5) **LLM**: gpt-4o-mini via API with multiple API keys for rate limit distribution. (6) **Ingestion**: Event-driven pipeline with webhook triggers for document updates. (7) **Monitoring**: Full observability -- latency, cost, quality, errors per component. Scaling: horizontal scaling for stateless services, read replicas for database, and consider multi-region if needed. Cost estimate: ~$25-50K/month depending on model choices."

### Q12: Your RAG system's retrieval precision dropped from 75% to 55% overnight. How do you debug?

**Strong Answer**: "Systematic debugging: (1) **Check recent changes**: Any deployments, model updates, or index changes in the last 24 hours? (2) **Check document freshness**: Did a batch document update corrupt the index? (3) **Check embedding model**: Are all embeddings still the same dimension? Any mixed models? (4) **Run golden dataset**: Quick smoke test against known queries -- is it a general degradation or specific to certain query types? (5) **Check vector DB health**: Index corruption? OOM event? (6) **Compare retrieval results**: For the same query, what was retrieved yesterday vs. today? (7) **Check upstream dependencies**: Did the embedding model API change? Most likely causes: embedding model change without full reindex, document update that corrupted metadata, or vector DB index issue."

### Q13: How do you fine-tune an embedding model for banking domain?

**Strong Answer**: "I need labeled query-document pairs. Steps: (1) **Collect training data**: 1000+ pairs from (a) user queries and clicked documents (implicit feedback), (b) manually labeled pairs from subject matter experts, (c) synthetic pairs generated from document headings. (2) **Choose base model**: BGE-large or E5-large as starting point. (3) **Training**: Use contrastive loss (InfoNCE) or triplet loss. Positive pairs are relevant query-document, negatives are hard negatives (documents that share keywords but aren't relevant). (4) **Validation**: Evaluate on held-out golden dataset -- compare fine-tuned vs. base model on retrieval precision. (5) **Deploy**: If fine-tuned model improves precision by > 5% on the validation set, deploy. Important: fine-tuning only helps if you have enough quality training data. With fewer than 500 pairs, the base model is likely better."

### Q14: How do you handle multi-hop reasoning in RAG?

**Strong Answer**: "Multi-hop questions require information from multiple documents that aren't directly connected. Example: 'What is the processing fee for the loan type that has the lowest interest rate?' Steps: (1) **Query decomposition**: Break into sub-questions: 'What loan types exist?' -> 'What are the interest rates for each?' -> 'Which has the lowest rate?' -> 'What is the processing fee for that loan type?' (2) **Sequential retrieval**: Answer each sub-question in order, using previous answers to inform subsequent queries. (3) **Agentic approach**: Use a ReAct loop where the LLM decides what to search for next based on intermediate results. (4) **Graph-based retrieval**: If you have a knowledge graph, traverse relationships between entities. The key challenge is that each hop introduces error -- if step 1 retrieves the wrong information, all subsequent steps compound the error."

### Q15: How do you optimize RAG latency for a customer-facing application?

**Strong Answer**: "Latency budget breakdown: (1) **Caching**: 30%+ cache hit rate for common queries eliminates the full pipeline. (2) **Query rewriting**: Use a fast model (gpt-4o-mini) or skip for simple queries. (3) **Embedding**: Self-hosted model on CPU (< 10ms) instead of API call (50-200ms). (4) **Retrieval**: Tune HNSW ef_search parameter, partition by category, use read replicas. Target < 50ms. (5) **Re-ranking**: Reduce candidates from 20 to 10, use two-stage re-ranking (fast model first, then accurate on top-10). (6) **LLM**: Use streaming for better perceived latency (TTFT < 500ms), limit max tokens, use faster models. (7) **Parallel execution**: Run independent pipeline stages concurrently. Total target: P95 < 2 seconds for customer-facing."

## Scenario-Based Questions

### Q16: A compliance officer says the AI gave wrong regulatory information. What do you do?

**Strong Answer**: "Immediate actions: (1) **Reproduce the issue**: Get the exact query, response, and timestamp from the audit logs. (2) **Check source documents**: Was the retrieved context correct? Was it the latest version of the regulation? (3) **Check the generation**: Did the LLM misinterpret the context, or was the context itself wrong? (4) **Contain**: If it's a systemic issue (wrong documents indexed), pause the system for that query category. (5) **Fix**: Re-index correct documents, strengthen verification, or update the system prompt. (6) **Communicate**: Inform the compliance officer and their manager about the fix. (7) **Prevent**: Add the failure case to the golden dataset, implement additional verification checks, and consider requiring human review for compliance-critical queries until confidence is restored."

### Q17: How would you build a RAG system that works across 10 languages?

**Strong Answer**: "Key decisions: (1) **Embedding model**: Use a multilingual model like BGE-M3 or Cohere embed-v3 that supports all target languages. (2) **Chunking**: Language-aware chunking -- some languages have different sentence boundaries and paragraph structures. (3) **Retrieval**: Query in the user's language retrieves documents in the same language. For cross-language retrieval (English query, Spanish document), the multilingual embedding model handles translation implicitly. (4) **LLM**: Use a multilingual LLM or route to language-specific models. (5) **Evaluation**: Golden dataset in each language -- don't assume performance is uniform across languages. (6) **Metadata**: Track language per document and per chunk. (7) **Cost**: Multilingual models may be more expensive; evaluate cost-per-language to optimize."

### Q18: The business wants to reduce RAG costs by 50%. How do you approach this?

**Strong Answer**: "Cost breakdown and optimization: (1) **LLM generation** (60% of cost): Switch from gpt-4o to gpt-4o-mini for non-critical queries (saves 80% on those). Implement response caching (saves 30% on repeated queries). Limit context window (saves 20% on input tokens). (2) **Re-ranking** (15%): Selective re-ranking -- skip when retrieval confidence is high (saves 40%). (3) **Embeddings** (5%): Self-host embedding model (saves API costs at scale). (4) **Vector DB** (10%): Optimize storage (FP16 compression, remove unused indexes). (5) **Query routing**: Classify queries and route to cheapest appropriate model. Simple factual queries go to a small model; complex analysis goes to a larger one. Combined, these can achieve 50%+ savings with < 5% quality impact."

## Quick-Fire Questions

### Q19: What is HyDE and when does it help?
**Answer**: "Hypothetical Document Embeddings -- generate a fake answer, embed it, and use it for retrieval. It helps when the question vocabulary differs significantly from the answer vocabulary. Works well for conceptual questions but poorly for specific factual queries."

### Q20: What is the optimal number of documents to retrieve before re-ranking?
**Answer**: "15-25 documents. Fewer than 10 may miss relevant docs; more than 30 adds re-ranking cost with diminishing returns. The exact number depends on document collection size and retrieval quality."

### Q21: How do you handle tables in documents for RAG?
**Answer**: "Tables need special handling: detect them during parsing, convert to markdown, chunk at the row level (not character level), and embed separately. Standard text extraction destroys table structure."

### Q22: What is Reciprocal Rank Fusion?
**Answer**: "A method to combine multiple ranked lists (e.g., BM25 results and vector results) by scoring each document as the sum of 1/(k + rank) across all lists it appears in. It doesn't require score normalization and works well in practice."

### Q23: When should you NOT use RAG?
**Answer**: "When the task is style/format adaptation (use fine-tuning), when there's no external knowledge needed (use the model directly), when latency requirements are too tight for retrieval (< 500ms), or when the knowledge base is too small (< 100 documents) -- in that case, just put everything in the prompt."

### Q24: What is the difference between RAG and fine-tuning?
**Answer**: "RAG adds knowledge at inference time by retrieving and injecting context. Fine-tuning modifies the model's weights to learn patterns. RAG is better for factual, changing information; fine-tuning is better for style, format, and task-specific behavior."

### Q25: How do you evaluate whether a new embedding model is better?
**Answer**: "Run both models on the same golden dataset and compare retrieval metrics: Precision@K, MRR, NDCG. Don't rely on benchmark scores (MTEB) alone -- evaluate on your actual data. Also consider cost, latency, and operational complexity."
