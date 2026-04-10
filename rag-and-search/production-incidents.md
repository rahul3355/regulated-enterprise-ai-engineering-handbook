# Production Incidents

## Why Study Incidents?

Learning from production failures is the fastest way to build robust RAG systems. This document captures real-world incidents from banking RAG deployments, their root causes, and lessons learned.

---

## Incident 1: Wrong Fee Information Due to Stale Index

**Severity**: High
**Duration**: 3 days
**Impact**: ~2,000 employees quoted incorrect loan processing fees to customers

### What Happened

The bank updated its personal loan processing fee from 1.5% to 2.0% effective July 1, 2024. The policy document was updated in the content management system, but the incremental indexing job failed silently due to a database connection timeout. The vector index still contained the old 1.5% fee information.

### Timeline

- **Day 1, 09:00**: Policy team updates fee in CMS
- **Day 1, 09:15**: CMS publishes update, triggers webhook to indexing service
- **Day 1, 09:15**: Indexing service receives webhook but database connection pool is exhausted
- **Day 1, 09:15**: Indexing job fails silently (error logged but no alert)
- **Day 1-3**: Employees use RAG assistant, get 1.5% fee info, quote to customers
- **Day 3, 14:00**: Compliance team notices discrepancy during routine audit
- **Day 3, 14:30**: Incident declared
- **Day 3, 15:00**: Manual reindexing completed
- **Day 3, 15:30**: Incident resolved

### Root Cause

1. **No retry logic**: Indexing job failed without retry
2. **Silent failure**: Error was logged but no alert was triggered
3. **No freshness monitoring**: Nobody tracked the age of indexed documents
4. **No content validation**: No check that indexed content matches source system

### Fixes Applied

```python
# Added: Retry with exponential backoff
@retry(max_attempts=5, backoff="exponential", max_delay=300)
def run_incremental_index():
    ...

# Added: Alert on indexing failure
def on_indexing_failure(error: Exception):
    pagerduty.alert(
        severity="critical",
        message=f"Incremental indexing failed: {error}",
        service="rag-indexing"
    )

# Added: Freshness monitoring
def check_index_freshness():
    oldest_index = db.query("SELECT MAX(indexed_at) FROM document_metadata")
    if datetime.utcnow() - oldest_index > timedelta(hours=2):
        alert(f"Index may be stale: last update was {oldest_index}")
```

### Lessons Learned

1. **Indexing failures must alert**: Any failure to update the index is a P1
2. **Monitor document freshness**: Track and alert on the age of indexed content
3. **Validate against source**: Periodically compare indexed content with source system
4. **Have a manual override**: Ability to force immediate reindexing
5. **Communicate policy changes**: When policy team changes something critical, the RAG team should know

---

## Incident 2: Hallucinated Regulatory Citation

**Severity**: Critical
**Duration**: 6 hours
**Impact**: Compliance team received incorrect Regulation E citation in response

### What Happened

An employee asked "What is the consumer liability limit for unauthorized electronic transfers under Regulation E?" The RAG system retrieved partially relevant documents but the LLM hallucinated a specific dollar amount ($100 instead of the correct $50) and cited a non-existent section number (1005.8 instead of 1005.6).

### Root Cause

1. **Incomplete retrieval**: The retrieved documents mentioned Regulation E but didn't contain the specific liability section
2. **Model overconfidence**: gpt-4o filled in gaps from training data with plausible but incorrect information
3. **No hallucination detection**: No post-generation verification was in place
4. **No citation validation**: Citations were not verified against source documents

### Fixes Applied

```python
# Added: NLI-based hallucination detection
def verify_response(response, context):
    claims = extract_claims(response)
    for claim in claims:
        if not claim_entailed_by(claim, context):
            return {"valid": False, "ungrounded_claim": claim}
    return {"valid": True}

# Added: Citation validation
def validate_citations(response, sources):
    cited_sections = extract_citations(response)
    for citation in cited_sections:
        if not verify_citation_exists(citation, sources):
            return {"valid": False, "hallucinated_citation": citation}
    return {"valid": True}

# Updated: Stronger system prompt
SYSTEM_PROMPT = """... If the specific section or number is not in the provided context, 
do not invent one. Say "The specific section number is not available in my current documents." ..."""
```

### Lessons Learned

1. **Regulatory queries need highest grounding standards**: Compliance queries should use gpt-4o with temperature=0 and strict citation requirements
2. **Verify citations**: Every citation should be checked against actual source documents
3. **NLI verification is mandatory for compliance**: Post-generation claim verification catches hallucinations
4. **Golden dataset must include regulatory queries**: Test with known regulatory Q&A before every deployment
5. **Temperature must be 0 for factual queries**: Any randomness increases hallucination risk

---

## Incident 3: Access Control Bypass in Multi-Tenant Search

**Severity**: Critical
**Duration**: 12 hours
**Impact**: 50 branch employees could view confidential corporate banking policies

### What Happened

The RAG system served multiple departments (retail, corporate, wealth management). The metadata filter for access control was incorrectly constructed -- it used `$or` instead of `$and` for combining department and clearance filters. This allowed retail banking employees to retrieve corporate banking documents.

### Root Cause

1. **Filter construction bug**: `$or` vs `$and` logic error
2. **No access control testing in golden dataset**: Golden dataset didn't include cross-department access tests
3. **No security review of retrieval pipeline**: Access control was added as an afterthought
4. **Insufficient logging**: Access violations were not logged or monitored

### Fixes Applied

```python
# Fixed: Correct filter construction
def build_access_filter(user: User) -> dict:
    """Build STRICT access control filter."""
    return {
        "$and": [  # ALL conditions must be met
            {"department": {"$in": user.allowed_departments}},
            {"min_clearance": {"$lte": user.clearance_level}},
            {"status": "active"},
        ]
    }

# Added: Access control tests in golden dataset
golden_queries.append(GoldenAccessTest(
    query="What are the corporate loan underwriting guidelines?",
    user=retail_employee_user,
    expected_result_count=0,  # Should return NOTHING
    description="Retail employee should not see corporate policies"
))

# Added: Access violation monitoring
def monitor_access_violations():
    """Detect and alert on potential access control issues."""
    violations = db.query("""
        SELECT user_id, query, retrieved_doc_id
        FROM retrieval_logs
        WHERE NOT is_authorized(user_id, retrieved_doc_id)
        AND timestamp > %s
    """, (datetime.utcnow() - timedelta(hours=1),))
    
    if violations:
        security_alert(f"Potential access control violation: {len(violations)} incidents")
```

### Lessons Learned

1. **Access control is the highest priority feature**: It must be designed, reviewed, and tested like a security system
2. **Test cross-tenant access**: Golden dataset must include queries that should NOT return results
3. **Log all access decisions**: Who accessed what, when, and was it authorized
4. **Security review for any retrieval change**: Any change to filtering logic needs security sign-off
5. **Defense in depth**: Access control at retrieval AND at response level (verify user can access each cited source)

---

## Incident 4: Embedding Model Regression After Update

**Severity**: Medium
**Duration**: 1 week
**Impact**: 15% drop in retrieval precision across all query types

### What Happened

The team updated the embedding model from text-embedding-3-small to text-embedding-3-large, expecting improved quality. However, the new model used 3072 dimensions vs. 1536, and the vector index was not rebuilt -- it contained a mix of old and new embeddings. Search quality degraded significantly because cosine similarity between different embedding dimensions is meaningless.

### Root Cause

1. **Mixed embedding dimensions**: Old chunks had 1536-dim embeddings, new ones had 3072-dim
2. **No dimension compatibility check**: System didn't validate all embeddings had same dimension
3. **No A/B test before full rollout**: Model change was deployed directly
4. **No retrieval quality monitoring**: The 15% precision drop wasn't detected for a week

### Fixes Applied

```python
# Added: Embedding dimension validation
def validate_embedding_consistency():
    """Ensure all embeddings have the same dimension."""
    dimensions = db.query("""
        SELECT DISTINCT vector_dims(embedding) as dim
        FROM document_chunks
    """)
    
    if len(dimensions) > 1:
        critical_alert(
            f"Embedding dimension mismatch: {dimensions}",
            "All embeddings must have the same dimension"
        )

# Added: Required full rebuild on model change
def change_embedding_model(new_model: str):
    """Change embedding model - requires full reindex."""
    current_model = config.get("embedding_model")
    if new_model != current_model:
        confirm = input(f"Changing model from {current_model} to {new_model}. "
                       f"This requires FULL REINDEX of {get_chunk_count()} chunks. Proceed? ")
        if confirm != "yes":
            raise AbortError("Aborted. Use 'yes' to confirm.")
        
        # Full rebuild
        full_rebuild(embedding_model=new_model)
        config.update("embedding_model", new_model)

# Added: Retrieval quality monitoring
def monitor_retrieval_quality():
    """Track retrieval quality with shadow golden dataset evaluation."""
    sample_queries = get_daily_sample_queries()  # 10 random queries from yesterday
    results = evaluate_retrieval(sample_queries)
    
    if results["precision_at_4"] < 0.60:  # Threshold
        alert(f"Retrieval precision dropped to {results['precision_at_4']:.2f}")
```

### Lessons Learned

1. **Embedding model changes require full reindex**: No partial updates
2. **Validate embedding consistency**: Check dimensions regularly
3. **Monitor retrieval quality continuously**: Not just at evaluation time
4. **A/B test model changes**: Run old and new models in parallel for a week
5. **Document model versions**: Track which model version indexed which documents

---

## Incident 5: Query Rewriting Infinite Loop

**Severity**: Medium
**Duration**: 4 hours
**Impact**: 30% increase in LLM costs, degraded latency for affected queries

### What Happened

A query rewriting bug caused certain queries to be expanded into variations that, when retrieved and answered, triggered follow-up queries that were similar to the original. The caching system saw these as "new" queries and rewrote them again, creating a loop of increasingly similar queries that never matched the cache.

### Root Cause

1. **Cache key was too specific**: Included full query text, not normalized form
2. **No query deduplication**: Similar queries were treated as distinct
3. **No rate limiting on query rewriting**: Unlimited rewrites per session
4. **No cost monitoring**: Cost spike wasn't detected for 4 hours

### Fixes Applied

```python
# Fixed: Normalized cache keys
def normalize_query_for_cache(query: str) -> str:
    """Normalize query for cache key matching."""
    normalized = query.lower().strip()
    normalized = re.sub(r'\s+', ' ', normalized)  # Normalize whitespace
    normalized = re.sub(r'[?!.]+$', '', normalized)  # Remove trailing punctuation
    normalized = re.sub(r'\b(a|an|the|is|are|do|does)\b', '', normalized)  # Remove stop words
    return normalized

# Added: Query similarity check
def is_similar_to_recent_query(query: str, session_id: str, 
                               threshold: float = 0.8) -> bool:
    """Check if this query is similar to recent queries in session."""
    recent_queries = get_recent_queries(session_id, window_minutes=5)
    query_emb = embed(query)
    
    for recent in recent_queries:
        recent_emb = embed(recent)
        similarity = cosine_similarity(query_emb, recent_emb)
        if similarity > threshold:
            return True
    return False

# Added: Cost monitoring
def monitor_hourly_cost():
    hourly_cost = get_costs_for_hour()
    avg_hourly_cost = get_avg_hourly_cost(last_7_days=168)
    
    if hourly_cost > avg_hourly_cost * 3:
        critical_alert(f"Cost spike: ${hourly_cost:.2f}/hr vs ${avg_hourly_cost:.2f}/hr avg")
```

### Lessons Learned

1. **Normalize cache keys**: Use semantic similarity, not exact string matching
2. **Rate limit query rewriting**: Maximum 3 rewrites per query per session
3. **Monitor costs in real-time**: Hourly cost alerts with 3x threshold
4. **Detect query loops**: Track query similarity within sessions
5. **Set budget caps**: Maximum daily/monthly spend with hard stops

---

## Incident 6: PDF Parsing Lost Critical Table Data

**Severity**: High
**Duration**: 2 weeks (discovered during quarterly review)
**Impact**: Fee schedule tables were completely missing from indexed content

### What Happened

A batch of banking policy PDFs contained critical fee schedules as embedded tables. The PDF parser extracted text content but did not detect or extract tables. As a result, queries about specific fees returned "I don't have that information" because the table data was never indexed.

### Root Cause

1. **No table extraction in PDF pipeline**: Parser only extracted text, ignored tables
2. **No content validation**: Nobody verified that key information (fee amounts) was indexed
3. **Golden dataset gap**: No test queries specifically asked about table content
4. **No PDF type detection**: Treated all PDFs the same regardless of content type

### Fixes Applied

```python
# Added: Table extraction to PDF pipeline
def parse_pdf_with_tables(pdf_path: str) -> dict:
    """Parse PDF extracting both text and tables."""
    import camelot
    
    # Extract text
    text = extract_text(pdf_path)
    
    # Extract tables
    tables = camelot.read_pdf(pdf_path, pages="all")
    
    result = {"text": text, "tables": []}
    
    for table in tables:
        if table.accuracy > 70:  # Only use tables with reasonable accuracy
            markdown = table.df.to_markdown()
            result["tables"].append({
                "page": table.parsing_report["page"],
                "content": markdown,
                "accuracy": table.parsing_report["accuracy"],
            })
    
    return result

# Added: Golden dataset includes table-based queries
golden_queries.extend([
    GoldenQuery(
        query="What is the wire transfer fee for domestic transfers?",
        relevant_doc_ids=["FEE-SCHEDULE-001"],
        expected_answer="$25 for domestic wire transfers",
        answer_type="single_fact",
        difficulty="medium",  # Medium because it's in a table
    ),
])
```

### Lessons Learned

1. **Tables require special handling**: They are not captured by text extraction
2. **Validate content coverage**: Verify that critical information types are indexed
3. **Test table queries specifically**: Include table-based Q&A in golden dataset
4. **Use proper table extraction tools**: Camelot, Textract, or Document AI
5. **Quarterly content audits**: Manually verify that all critical info is searchable

---

## Incident 7: Vector Database Index Corruption After OOM

**Severity**: Critical
**Duration**: 8 hours
**Impact**: Complete search outage for 8 hours

### What Happened

The pgvector HNSW index became corrupted after an out-of-memory event during a bulk indexing operation. The index was partially written when the server ran out of memory and restarted. After restart, queries returned incorrect results or timed out. The team had to rebuild the entire index from scratch, which took 8 hours for 5M chunks.

### Root Cause

1. **No memory limits on indexing**: Bulk indexing consumed all available memory
2. **No index integrity check**: Corruption was not detected until users complained
3. **No fallback**: No secondary index or degraded mode
4. **Long rebuild time**: 5M chunks took 8 hours to re-embed and re-index

### Fixes Applied

```sql
-- Added: Memory limits for indexing operations
-- In postgresql.conf
maintenance_work_mem = '2GB'  -- Cap memory for index building
max_worker_processes = 8      -- Limit parallel workers

-- Added: Index integrity check
SELECT * FROM pgstatprogress_create_index 
WHERE index_relid = 'document_chunks_embedding_idx'::regclass;

-- Added: Batched indexing with memory monitoring
def batched_index_with_memory_monitor(chunks, batch_size=500):
    for i in range(0, len(chunks), batch_size):
        batch = chunks[i:i+batch_size]
        
        # Check memory before indexing
        mem_usage = get_memory_usage()
        if mem_usage > 0.8:  # 80% of available
            log_warning(f"High memory usage: {mem_usage:.0%}")
            time.sleep(60)  # Wait for memory to free up
        
        index_batch(batch)
        
        # Verify index after each batch
        verify_index_integrity()
```

### Lessons Learned

1. **Batch indexing with memory monitoring**: Never bulk-index without memory limits
2. **Verify index integrity**: Regular integrity checks, especially after OOM events
3. **Have a read replica**: Secondary index for failover
4. **Optimize rebuild time**: Pre-compute embeddings so rebuild only needs index construction
5. **Store embeddings separately**: Keep raw embeddings so you don't need to re-embed during rebuild

---

## Common Patterns Across Incidents

| Pattern | Frequency | Severity | Prevention |
|---|---|---|---|
| Silent failures | 3/7 | Critical | Alerting on all failures |
| Missing validation | 4/7 | High | Input/output validation at every stage |
| No monitoring gap | 5/7 | High | Monitor all critical metrics with alerts |
| Golden dataset gap | 3/7 | Medium-High | Comprehensive, regularly updated golden dataset |
| No rollback plan | 2/7 | High | Every change must be reversible |
| Insufficient testing | 4/7 | High | Test edge cases, not just happy path |

## Incident Response Runbook

1. **Detect**: Monitoring alerts or user reports
2. **Acknowledge**: On-call engineer acknowledges within 15 minutes
3. **Assess**: Determine severity and impact within 30 minutes
4. **Mitigate**: Apply immediate fix (rollback, disable feature, manual override)
5. **Communicate**: Notify affected teams and stakeholders
6. **Root cause**: Investigate and identify root cause
7. **Fix**: Implement permanent fix
8. **Verify**: Test fix against golden dataset and production traffic
9. **Post-mortem**: Write incident report with lessons learned
10. **Prevent**: Implement preventive measures (tests, monitoring, process changes)
