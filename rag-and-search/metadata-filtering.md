# Metadata Filtering

## Why Metadata Matters

Metadata is structured information attached to each chunk that enables precise filtering during retrieval. Without metadata, you can only search by semantic similarity -- which is powerful but insufficient for real-world applications.

**Banking example**: A user asks "What is the processing fee for personal loans?" The knowledge base has:
- Personal Loan Policy (2024, Retail Banking, Active)
- Personal Loan Policy (2023, Retail Banking, Archived)
- Business Loan Policy (2024, Corporate Banking, Active)

Without metadata filtering, all three may rank highly. With metadata, you can filter to `department="Retail Banking" AND status="Active"` before the semantic search, ensuring only the current retail policy is considered.

## Common Metadata Fields

```python
# Metadata for each chunk in a banking RAG system
chunk_metadata = {
    # Document identity
    "doc_id": "policy-retail-loan-2024",
    "doc_title": "Retail Personal Loan Policy",
    "doc_version": "3.2",
    "doc_type": "policy",           # policy, procedure, faq, training, report
    
    # Organizational context
    "department": "retail_banking",  # retail, corporate, wealth, compliance
    "business_unit": "lending",
    "region": "us",                  # us, uk, eu, apac, global
    
    # Temporal information
    "created_at": "2024-01-15",
    "updated_at": "2024-06-20",
    "effective_from": "2024-07-01",
    "effective_until": None,         # None = still active
    "status": "active",              # active, archived, draft
    
    # Access control
    "min_clearance": 2,              # 1-5 clearance level
    "allowed_roles": ["loan_officer", "branch_manager", "compliance"],
    "restricted_countries": [],
    
    # Content classification
    "language": "en",
    "contains_pii": False,
    "regulatory_framework": "reg_e", # reg_e, reg_z, bsa, aml
    "tags": ["loan", "personal", "fees", "processing"],
    
    # Chunk-specific
    "chunk_index": 3,
    "total_chunks": 12,
    "section_title": "Processing Fees",
    "chunk_type": "text",           # text, table, image, list
}
```

## Filtering Strategies

### Pre-Filtering (Filter Before Search)

Apply metadata filter before vector search. This is the default and recommended approach.

```python
def pre_filter_search(query: str, user: User, vectorstore, k: int = 4):
    """Filter first, then search within filtered results."""
    
    filter_dict = {
        "department": {"$in": user.allowed_departments},
        "status": "active",
        "min_clearance": {"$lte": user.clearance_level}
    }
    
    return vectorstore.similarity_search(
        query, k=k, filter=filter_dict
    )
```

**Pros**: Most secure (inaccessible docs never searched); efficient
**Cons**: May return fewer results if filter is too restrictive

### Post-Filtering (Search Then Filter)

Search broadly, then filter results by metadata.

```python
def post_filter_search(query: str, user: User, vectorstore, k: int = 4):
    """Search first, then filter."""
    
    # Search without access filter (broader)
    results = vectorstore.similarity_search(query, k=k * 3)
    
    # Filter by access
    filtered = [
        doc for doc in results
        if doc.metadata["department"] in user.allowed_departments
        and doc.metadata["status"] == "active"
        and doc.metadata["min_clearance"] <= user.clearance_level
    ]
    
    return filtered[:k]
```

**Pros**: More results available; better recall
**Cons**: Security risk (metadata leakage); less efficient
**When to use**: When pre-filter returns too few results; non-security-critical filters

### Hybrid Pre/Post Filtering

```python
def hybrid_filter_search(query: str, user: User, vectorstore, k: int = 4):
    """Pre-filter for security, post-filter for quality."""
    
    # Pre-filter: Security-critical (access control, status)
    secure_filter = {
        "min_clearance": {"$lte": user.clearance_level},
        "status": "active"
    }
    
    results = vectorstore.similarity_search(query, k=k * 3, filter=secure_filter)
    
    # Post-filter: Quality filters (department, region)
    filtered = [
        doc for doc in results
        if doc.metadata.get("department") in user.allowed_departments
    ]
    
    # If too few results, relax department filter
    if len(filtered) < k:
        relaxed = vectorstore.similarity_search(
            query, k=k, filter=secure_filter
        )
        filtered.extend(relaxed)
        filtered = list({d.metadata["doc_id"]: d for d in filtered}.values())
    
    return filtered[:k]
```

## Access Control in Retrieval

### Role-Based Access Control (RBAC)

```python
from typing import Optional

class AccessControlFilter:
    """Enforce RBAC on document retrieval."""
    
    def __init__(self, role_permissions: dict, doc_roles: dict):
        # role_permissions: {"loan_officer": ["retail_loans", "customer_data"], ...}
        self.role_permissions = role_permissions
        # doc_categories: which permission is needed for each doc category
        self.doc_roles = doc_roles
    
    def build_filter(self, user_roles: list[str]) -> dict:
        """Build vector DB filter based on user roles."""
        
        # Get all permissions for user's roles
        permissions = set()
        for role in user_roles:
            permissions.update(self.role_permissions.get(role, []))
        
        # Build filter: document must require a permission the user has
        return {
            "required_permission": {"$in": list(permissions)}
        }
    
    def can_access(self, doc_metadata: dict, user_roles: list[str]) -> bool:
        """Check if user can access a specific document."""
        doc_permission = doc_metadata.get("required_permission")
        if not doc_permission:
            return True  # Public document
        
        for role in user_roles:
            if doc_permission in self.role_permissions.get(role, []):
                return True
        return False
```

### Attribute-Based Access Control (ABAC)

More granular than RBAC -- access based on user attributes, document attributes, and environmental conditions.

```python
def abac_filter(user: dict, doc: dict, context: dict) -> bool:
    """Attribute-based access control."""
    
    # Check department match
    if user["department"] not in doc.get("allowed_departments", []):
        return False
    
    # Check clearance level
    if user["clearance"] < doc.get("min_clearance", 0):
        return False
    
    # Check region
    if user["region"] != doc.get("region") and doc.get("region") != "global":
        return False
    
    # Check effective date (user can only see currently effective policies)
    today = context.get("current_date", datetime.now())
    effective_from = doc.get("effective_from")
    effective_until = doc.get("effective_until")
    
    if effective_from and today < effective_from:
        return False  # Policy not yet effective
    if effective_until and today > effective_until:
        return False  # Policy expired
    
    # Check time-based access (compliance docs only during business hours)
    if doc.get("time_restricted") and not is_business_hours():
        return False
    
    return True
```

### Hierarchical Access Control

Banking organizations have hierarchies: branch -> region -> country -> global.

```python
def hierarchical_filter(user_branch: str, user_region: str, 
                        user_country: str, vectorstore, query: str, k: int = 4):
    """Filter documents by organizational hierarchy."""
    
    # Build hierarchical filter (OR across levels)
    filter_dict = {
        "$or": [
            {"scope": "global"},
            {"scope": "country", "country": user_country},
            {"scope": "region", "region": user_region},
            {"scope": "branch", "branch": user_branch},
        ]
    }
    
    return vectorstore.similarity_search(query, k=k, filter=filter_dict)
```

## Hybrid Search with Metadata

Combining BM25, vector search, and metadata filtering:

```python
def full_hybrid_with_metadata(query: str, user: User, vectorstore, k: int = 4):
    """Complete retrieval: metadata filter + hybrid search + re-ranking."""
    
    # Step 1: Build metadata filter
    metadata_filter = {
        "$and": [
            {"status": "active"},
            {"department": {"$in": user.allowed_departments}},
            {"min_clearance": {"$lte": user.clearance_level}},
        ]
    }
    
    # Step 2: Hybrid search with filter
    # BM25 component
    bm25_results = bm25_search(query, k=k*3, filter=metadata_filter)
    
    # Vector component  
    vector_results = vector_search(query, k=k*3, filter=metadata_filter)
    
    # Combine
    combined = combine_scores(bm25_results, vector_results, alpha=0.3)
    
    # Step 3: Re-rank (if re-ranker available)
    if reranker:
        top_candidates = combined[:20]
        reranked = reranker.rerank(query, top_candidates, top_k=k)
        return reranked
    
    return combined[:k]
```

## Metadata Filtering in Different Vector Databases

### pgvector (SQL-based)

```sql
-- Metadata stored as JSONB in the same table
SELECT id, content, metadata, embedding <=> $1 as distance
FROM documents
WHERE metadata->>'status' = 'active'
  AND metadata->>'department' = ANY($2)
  AND (metadata->>'min_clearance')::int <= $3
ORDER BY distance
LIMIT $4;
```

### Pinecone (Metadata Filter)

```python
index.query(
    vector=query_embedding,
    top_k=20,
    filter={
        "status": {"$eq": "active"},
        "department": {"$in": ["retail", "wealth"]},
        "min_clearance": {"$lte": 3}
    }
)
```

### Milvus (Expression Filter)

```python
results = collection.search(
    data=[query_embedding],
    anns_field="embedding",
    param={"metric_type": "L2", "params": {"nprobe": 16}},
    limit=20,
    expr='status == "active" and department in ["retail", "wealth"] and min_clearance <= 3'
)
```

### Weaviate (Where Filter)

```python
client.query.get("Documents", ["content", "metadata"])
    .with_near_vector({"vector": query_embedding})
    .with_where({
        "operator": "And",
        "operands": [
            {"path": ["status"], "operator": "Equal", "valueString": "active"},
            {"path": ["department"], "operator": "ContainsAny", 
             "valueStringArray": ["retail", "wealth"]}
        ]
    })
    .with_limit(20)
    .do()
```

## Debugging Metadata Filters

### Common Issues

1. **Filter returns zero results**: Filter is too restrictive. Log the filter criteria and document counts at each level.

```python
def debug_filter_counts(vectorstore, base_filter: dict) -> dict:
    """Show how many docs match each filter condition."""
    counts = {}
    
    # Base count without filter
    counts["no_filter"] = vectorstore.total_count()
    
    # Count for each individual condition
    for key, value in base_filter.items():
        counts[f"filter_{key}"] = vectorstore.count(filter={key: value})
    
    # Count for combined filter
    counts["combined"] = vectorstore.count(filter=base_filter)
    
    return counts
```

2. **Type mismatches**: `min_clearance` stored as string "3" vs integer 3. Ensure consistent types.

3. **Missing metadata**: New ingestion pipeline adds fields that old documents don't have. Use `metadata.get("field", default)` pattern.

4. **Array field filtering**: Filtering on `allowed_roles` array requires `$in` or `CONTAINS` syntax depending on the database.

## Best Practices

1. **Always filter by status**: Exclude archived and draft documents from retrieval
2. **Filter by effective date**: Only return currently effective policies
3. **Filter by access level before search**: Never retrieve restricted docs and filter after
4. **Use metadata for routing**: Route queries to the right department collection
5. **Log filter criteria**: For debugging and audit compliance
6. **Version metadata**: Track which metadata schema version was used
7. **Index metadata**: Ensure frequently-filtered fields are indexed

## Metadata Schema Versioning

As your system evolves, metadata schemas change. Handle this with versioning:

```python
# Metadata with schema version
{
    "schema_version": "2.0",
    "doc_id": "...",
    "department": "retail",  # was "dept" in v1
    "status": "active",      # was "is_active": true in v1
    # ... other fields
}

# Migration query
def migrate_metadata_v1_to_v2():
    """Update all v1 metadata to v2 schema."""
    for doc in vectorstore.get_all():
        meta = doc.metadata
        if meta.get("schema_version") == "1.0":
            meta["department"] = meta.pop("dept")
            meta["status"] = "active" if meta.pop("is_active") else "archived"
            meta["schema_version"] = "2.0"
            vectorstore.update_metadata(doc.id, meta)
```
