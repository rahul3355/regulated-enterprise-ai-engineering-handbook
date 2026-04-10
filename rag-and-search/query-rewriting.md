# Query Rewriting

## Why Rewrite Queries?

User queries are often poorly formed for retrieval. They may be:
- Too brief ("loan fees" - which loan? what context?)
- Too verbose (a paragraph of context mixed with the actual question)
- Using different vocabulary than the documents
- Multi-part questions requiring multiple retrieval operations
- Ambiguous (could refer to multiple topics)

Query rewriting transforms the user's raw query into a form that the retriever can handle more effectively.

```
User Query: "Can I get a loan?"
Query Rewriter: 
  -> "personal loan eligibility criteria"
  -> "minimum requirements for personal loan application"
  -> "who is eligible to apply for a personal loan"
```

## Query Rewriting Techniques

### 1. Query Expansion with LLM

Generate multiple query variations using an LLM.

```python
def expand_query_with_llm(query: str, llm, num_variations: int = 3) -> list[str]:
    """Generate semantically equivalent query variations."""
    
    prompt = f"""You are a search query optimizer. Generate {num_variations} alternative 
    search queries for the following question. Each alternative should express the same 
    intent but use different vocabulary and phrasing.

    Original query: {query}

    Return only the queries, one per line, with no numbering or explanation."""
    
    response = llm.generate(prompt)
    variations = [line.strip() for line in response.strip().split("\n") if line.strip()]
    
    return [query] + variations[:num_variations]

# Example
query = "What are the processing fees?"
variations = expand_query_with_llm(query, llm)
# ['What are the processing fees?',
#  'Loan processing fee charges',
#  'How much does it cost to process a loan application?',
#  'Application processing fee amount']
```

**Banking-specific prompt**:
```python
def banking_query_expansion(query: str, llm):
    """Expand query with banking domain knowledge."""
    
    prompt = f"""You are a banking search expert. Generate 3 alternative search queries.
    
    Rules:
    - Include relevant banking terminology variations (e.g., "fee" <-> "charge" <-> "cost")
    - Expand abbreviations (KYC, AML, APR)
    - Add context that a banker would understand from the query
    
    Original: {query}
    
    Return 3 queries, one per line."""
    
    return parse_lines(llm.generate(prompt))
```

### 2. Query Decomposition

Break complex queries into sub-questions.

```python
def decompose_query(query: str, llm) -> list[str]:
    """Break a complex query into atomic sub-questions."""
    
    prompt = f"""Break down the following question into 2-4 sub-questions.
    Each sub-question should focus on a single, specific aspect.
    If the original question is already atomic, return it as-is.

    Question: {query}

    Return sub-questions as a numbered list (1. 2. 3.)."""
    
    response = llm.generate(prompt)
    sub_questions = parse_numbered_list(response)
    
    return sub_questions

# Example
query = "What are the eligibility criteria and fees for a personal loan, and how long does approval take?"
sub_questions = decompose_query(query, llm)
# [
#   "What are the eligibility criteria for a personal loan?",
#   "What fees are charged for personal loans?",
#   "How long does personal loan approval take?"
# ]
```

### 3. HyDE (Hypothetical Document Embeddings)

Generate a hypothetical answer document, then use it for retrieval.

```python
def hyde_query_expansion(query: str, llm) -> str:
    """Generate a hypothetical answer for retrieval."""
    
    prompt = f"""Write a plausible, detailed answer to the following question.
    The answer should be 2-3 paragraphs and sound authoritative.
    It does not need to be factually accurate - we're using it for search only.

    Question: {query}

    Answer:"""
    
    hypothetical_doc = llm.generate(prompt)
    return hypothetical_doc

# Use the hypothetical document as the search query
# It will be in the same vector space as real answers
```

### 4. Multi-Query Retrieval Pipeline

```python
class MultiQueryRetriever:
    def __init__(self, vectorstore, llm, base_retriever=None):
        self.vectorstore = vectorstore
        self.llm = llm
        self.base_retriever = base_retriever or vectorstore
    
    def retrieve(self, query: str, k: int = 4, k_per_query: int = 3) -> list:
        """Full multi-query retrieval pipeline."""
        
        # Step 1: Generate query variations
        variations = self._expand_query(query)
        all_queries = [query] + variations
        
        # Step 2: Retrieve from each query
        all_docs = {}  # doc_id -> {doc, matched_queries, max_score}
        for q in all_queries:
            results = self.base_retriever.similarity_search(q, k=k_per_query)
            for doc in results:
                doc_id = doc.metadata.get("id", hash(doc.page_content))
                if doc_id not in all_docs:
                    all_docs[doc_id] = {
                        "doc": doc,
                        "matched_queries": [],
                        "count": 0
                    }
                all_docs[doc_id]["matched_queries"].append(q)
                all_docs[doc_id]["count"] += 1
        
        # Step 3: Score by consensus (more query matches = more relevant)
        scored = sorted(
            all_docs.values(),
            key=lambda x: x["count"],
            reverse=True
        )
        
        return [item["doc"] for item in scored[:k]]
    
    def _expand_query(self, query: str) -> list[str]:
        prompt = f"""Generate 3 alternative search queries for: {query}
        Return one per line."""
        return [l.strip() for l in self.llm.generate(prompt).split("\n") if l.strip()]
```

### 5. Step-Back Prompting

Generate a more general "step-back" question to retrieve foundational context.

```python
def step_back_query(query: str, llm) -> list[str]:
    """Generate step-back questions for broader context."""
    
    prompt = f"""Generate 1-2 "step-back" questions that are more general than the original.
    These should retrieve foundational knowledge that helps answer the specific question.

    Original: {query}
    
    Step-back questions:"""
    
    return parse_lines(llm.generate(prompt))

# Example
query = "What is the processing fee for a $50,000 personal loan?"
step_back = step_back_query(query, llm)
# ["What fees are associated with personal loans?",
#  "How are personal loan fees calculated?"]
```

### 6. Sub-Question Retrieval with Aggregation

```python
def sub_question_retrieval(query: str, llm, vectorstore, k: int = 4) -> dict:
    """Retrieve for each sub-question and organize results."""
    
    # Decompose
    sub_questions = decompose_query(query, llm)
    
    # Retrieve for each sub-question
    results_by_subq = {}
    all_docs = {}
    
    for sq in sub_questions:
        docs = vectorstore.similarity_search(sq, k=3)
        results_by_subq[sq] = docs
        
        for doc in docs:
            doc_id = doc.metadata["id"]
            if doc_id not in all_docs:
                all_docs[doc_id] = {"doc": doc, "relevance_count": 0, "sub_questions": []}
            all_docs[doc_id]["relevance_count"] += 1
            all_docs[doc_id]["sub_questions"].append(sq)
    
    # Sort by cross-sub-question relevance
    ranked = sorted(all_docs.values(), key=lambda x: x["relevance_count"], reverse=True)
    
    return {
        "sub_questions": sub_questions,
        "results_by_subq": results_by_subq,
        "combined_results": [r["doc"] for r in ranked[:k]]
    }
```

## Query Classification for Routing

Different query types benefit from different rewriting strategies.

```python
def classify_query(query: str, llm) -> str:
    """Classify query type to select rewriting strategy."""
    
    prompt = f"""Classify this query into one of these categories:
    - factual: asking for a specific fact (e.g., "what is the fee for X?")
    - procedural: asking how to do something (e.g., "how do I apply?")
    - comparative: comparing options (e.g., "which is better, X or Y?")
    - multi_part: multiple questions in one (e.g., "what is X and how do I Y?")
    - ambiguous: unclear or too brief (e.g., "loan fees")
    
    Query: {query}
    
    Return only the category name."""
    
    return llm.generate(prompt).strip().lower()

def select_rewriting_strategy(query: str, llm) -> str:
    """Choose rewriting strategy based on query classification."""
    
    category = classify_query(query, llm)
    
    strategies = {
        "factual": "direct",           # No rewriting needed
        "procedural": "expansion",     # Expand with synonyms
        "comparative": "decomposition", # Split into separate comparisons
        "multi_part": "decomposition", # Split into sub-questions
        "ambiguous": "hyde",           # Generate hypothetical to disambiguate
    }
    
    return strategies.get(category, "expansion")
```

## Banking-Specific Query Rewriting

### Banking Terminology Expansion

```python
BANKING_SYNONYMS = {
    "fee": ["fee", "charge", "cost", "rate", "pricing"],
    "loan": ["loan", "credit", "borrowing", "financing"],
    "kyc": ["kyc", "know your customer", "customer identification", "identity verification"],
    "aml": ["aml", "anti-money laundering", "money laundering prevention"],
    "apr": ["apr", "annual percentage rate", "interest rate", "annual rate"],
    "emi": ["emi", "equated monthly installment", "monthly payment", "monthly due"],
    "overdraft": ["overdraft", "od", "negative balance", "excess withdrawal"],
    "fd": ["fd", "fixed deposit", "term deposit", "time deposit"],
    "cd": ["cd", "certificate of deposit"],
}

def expand_banking_terms(query: str) -> list[str]:
    """Expand banking abbreviations and synonyms."""
    variations = [query]
    
    for term, synonyms in BANKING_SYNONYMS.items():
        if term.lower() in query.lower():
            for synonym in synonyms:
                if synonym.lower() != term.lower():
                    variations.append(query.lower().replace(term.lower(), synonym))
    
    return list(set(variations))
```

### Regulatory Query Handling

```python
def handle_regulatory_query(query: str, llm, vectorstore) -> list:
    """Handle queries about specific regulations."""
    
    # Detect regulation references
    regulation_patterns = [
        (r'regulation\s*[- ]?e', 'Regulation E'),
        (r'regulation\s*[- ]?z', 'Regulation Z'),
        (r'regulation\s*[- ]?b', 'Regulation B'),
        (r'regulation\s*[- ]?dd', 'Regulation DD'),
        (r'bsa\b', 'Bank Secrecy Act'),
        (r'\baml\b', 'Anti-Money Laundering'),
        (r'\bkyc\b', 'Know Your Customer'),
        (r'dodd[- ]frank', 'Dodd-Frank Act'),
        (r'sar\b', 'Suspicious Activity Report'),
        (r'ctr\b', 'Currency Transaction Report'),
    ]
    
    detected_regs = []
    for pattern, name in regulation_patterns:
        if re.search(pattern, query, re.IGNORECASE):
            detected_regs.append(name)
    
    if detected_regs:
        # Expand with full regulation names
        expanded_queries = [query]
        for reg_name in detected_regs:
            expanded_queries.append(query + f" {reg_name}")
        return expanded_queries
    
    return [query]
```

## Complete Query Rewriting Pipeline

```python
class QueryRewritingPipeline:
    def __init__(self, llm, vectorstore, reranker=None):
        self.llm = llm
        self.vectorstore = vectorstore
        self.reranker = reranker
    
    def process(self, query: str, k: int = 4) -> dict:
        """Full query rewriting and retrieval pipeline."""
        
        # Step 1: Classify query
        category = classify_query(query, self.llm)
        
        # Step 2: Select and apply rewriting strategy
        if category == "multi_part":
            sub_questions = decompose_query(query, self.llm)
            all_queries = [query] + sub_questions
        elif category == "ambiguous":
            all_queries = expand_query_with_llm(query, self.llm, num_variations=5)
        elif category == "comparative":
            all_queries = decompose_query(query, self.llm)
        else:
            all_queries = expand_query_with_llm(query, self.llm, num_variations=3)
        
        # Add banking term expansion
        for q in list(all_queries):
            all_queries.extend(expand_banking_terms(q))
        
        # Step 3: Deduplicate queries
        all_queries = list(dict.fromkeys(all_queries))  # Preserve order, remove dupes
        
        # Step 4: Retrieve from each query
        all_docs = {}
        for q in all_queries:
            results = self.vectorstore.similarity_search(q, k=k * 2)
            for doc in results:
                doc_id = doc.metadata.get("id", hash(doc.page_content))
                if doc_id not in all_docs:
                    all_docs[doc_id] = {
                        "doc": doc,
                        "query_count": 0,
                        "queries": []
                    }
                all_docs[doc_id]["query_count"] += 1
                all_docs[doc_id]["queries"].append(q)
        
        # Step 5: Re-rank (if available)
        ranked_docs = sorted(
            all_docs.values(),
            key=lambda x: x["query_count"],
            reverse=True
        )
        
        # Step 6: Optional re-ranking
        if self.reranker and len(ranked_docs) > k:
            top_candidates = ranked_docs[:k * 3]
            texts = [d["doc"].page_content for d in top_candidates]
            reranked = self.reranker.rerank(query, texts, top_k=k)
            return {
                "query": query,
                "category": category,
                "expanded_queries": all_queries,
                "documents": reranked,
                "method": "query_expansion_with_reranking"
            }
        
        return {
            "query": query,
            "category": category,
            "expanded_queries": all_queries[:10],
            "documents": [d["doc"] for d in ranked_docs[:k]],
            "method": "query_expansion"
        }
```

## Evaluation of Query Rewriting

```python
def evaluate_query_rewriting(golden_queries, pipeline):
    """Compare retrieval with and without query rewriting."""
    
    baseline_scores = []
    rewritten_scores = []
    
    for gq in golden_queries:
        # Baseline: direct retrieval
        baseline_docs = pipeline.vectorstore.similarity_search(gq.query, k=4)
        baseline_precision = compute_precision(baseline_docs, gq.relevant_docs)
        baseline_scores.append(baseline_precision)
        
        # With rewriting
        result = pipeline.process(gq.query, k=4)
        rewritten_precision = compute_precision(result["documents"], gq.relevant_docs)
        rewritten_scores.append(rewritten_precision)
    
    print(f"Baseline P@4: {sum(baseline_scores)/len(baseline_scores):.3f}")
    print(f"Rewritten P@4: {sum(rewritten_scores)/len(rewritten_scores):.3f}")
    
    return {
        "baseline": sum(baseline_scores) / len(baseline_scores),
        "rewritten": sum(rewritten_scores) / len(rewritten_scores),
        "improvement": sum(rewritten_scores) / len(rewritten_scores) - sum(baseline_scores) / len(baseline_scores)
    }
```

### Typical Results

| Strategy | P@4 | MRR | Notes |
|---|---|---|---|
| Direct retrieval | 0.58 | 0.52 | Baseline |
| Query expansion (LLM) | 0.68 | 0.63 | +17% precision |
| Query decomposition | 0.72 | 0.68 | For multi-part queries |
| HyDE | 0.65 | 0.60 | Good for ambiguous queries |
| Full pipeline (classify + rewrite) | 0.75 | 0.71 | Best overall |

## Latency Considerations

Query rewriting adds LLM calls, increasing latency:

| Component | Latency | Cumulative |
|---|---|---|
| Query classification | 200ms | 200ms |
| Query expansion (3 variations) | 400ms | 600ms |
| Retrieval (4 queries x 50ms) | 200ms (parallel) | 800ms |
| Re-ranking | 150ms | 950ms |
| Total | | ~1 second |

**Optimization**: Use a small, fast model (gpt-4o-mini) for query rewriting. Cache rewritten queries for common patterns.

## When to Skip Query Rewriting

1. **Latency budget < 500ms**: Rewriting adds 400-800ms
2. **Simple, well-formed queries**: "What is the fee for X?" rarely needs rewriting
3. **Small document collection**: When vector search already achieves > 80% P@4
4. **High cost sensitivity**: Each rewrite costs 1-3 LLM API calls

**Rule of thumb**: Always use query rewriting for production banking RAG. The 15-25% improvement in retrieval precision is worth the added latency and cost.
