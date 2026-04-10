# Document Freshness

## Why Freshness Matters

Banking policies, regulations, and procedures change frequently. A RAG system that returns outdated information can give employees wrong guidance, leading to compliance violations and customer harm.

**Example**: A bank updates its personal loan processing fee from 1.5% to 2.0% effective July 1, 2024. If the RAG system still returns the old policy, employees will quote the wrong fee to customers.

## Freshness Strategies

### 1. Document Versioning and Effective Dates

```python
chunk_metadata = {
    "doc_id": "POL-RTL-001",
    "doc_version": "3.2",
    "previous_version": "3.1",
    "effective_from": "2024-07-01",
    "effective_until": None,  # Still current
    "status": "active",  # vs "archived" for old versions
    "updated_at": "2024-06-20T10:30:00Z",
    "updated_by": "policy_team",
    "change_summary": "Updated processing fee from 1.5% to 2.0%",
}
```

### 2. Retrieval with Freshness Preference

```python
def freshness_aware_retrieval(query: str, vectorstore, k: int = 4):
    """Prefer current documents but don't exclude history if needed."""
    
    # Primary: search active documents only
    active_results = vectorstore.similarity_search(
        query, k=k,
        filter={"status": "active"}
    )
    
    # If not enough active results, include recently archived
    if len(active_results) < k:
        recent_archived = vectorstore.similarity_search(
            query, k=k - len(active_results),
            filter={
                "status": "archived",
                "effective_until": {"$gte": (datetime.now() - timedelta(days=90)).isoformat()}
            }
        )
        active_results.extend(recent_archived)
    
    return active_results
```

### 3. Freshness Scoring

```python
def freshness_score(doc_metadata: dict, max_age_days: int = 365) -> float:
    """Calculate freshness score (1.0 = current, 0.0 = very old)."""
    
    updated = doc_metadata.get("updated_at")
    if not updated:
        return 0.5  # Unknown age
    
    age = (datetime.now() - datetime.fromisoformat(updated)).days
    
    if age <= 30:
        return 1.0  # Less than a month old
    elif age <= max_age_days:
        return 1.0 - (age / max_age_days)
    else:
        return 0.0  # Older than max_age_days

def rerank_by_freshness_and_relevance(results: list, alpha: float = 0.7) -> list:
    """Re-rank by combining relevance and freshness."""
    scored = []
    for doc in results:
        relevance = doc.metadata.get("score", 0.5)
        freshness = freshness_score(doc.metadata)
        combined = alpha * relevance + (1 - alpha) * freshness
        scored.append((combined, doc))
    
    scored.sort(reverse=True, key=lambda x: x[0])
    return [doc for _, doc in scored]
```

## Incremental Indexing

### Detecting Changed Documents

```python
def detect_changed_documents(current_docs: dict, indexed_docs: dict) -> dict:
    """Compare current documents against indexed versions."""
    
    additions = []
    deletions = []
    updates = []
    
    # New documents
    for doc_id, doc in current_docs.items():
        if doc_id not in indexed_docs:
            additions.append(doc_id)
        elif doc["content_hash"] != indexed_docs[doc_id]["content_hash"]:
            updates.append(doc_id)
    
    # Deleted documents
    for doc_id in indexed_docs:
        if doc_id not in current_docs:
            deletions.append(doc_id)
    
    return {
        "additions": additions,
        "deletions": deletions,
        "updates": updates,
        "total_changes": len(additions) + len(deletions) + len(updates)
    }
```

### Change Detection Methods

| Method | How It Works | Accuracy | Complexity |
|---|---|---|---|
| **Content hashing** | Hash document content, compare | Exact | Low |
| **Last modified timestamp** | Compare file modification time | Approximate | Very Low |
| **Version number** | Compare version metadata | Exact | Low |
| **Database triggers** | CDC on source database | Exact | Medium |
| **Scheduled diff** | Periodic full comparison | Exact | High |

### Hash-Based Change Detection

```python
import hashlib

def compute_content_hash(content: str) -> str:
    """Compute SHA-256 hash of document content."""
    return hashlib.sha256(content.encode("utf-8")).hexdigest()

def index_documents_with_hash(documents: list, vectorstore, metadata_store):
    """Index documents, tracking content hashes to detect changes."""
    
    for doc in documents:
        doc_id = doc["id"]
        content_hash = compute_content_hash(doc["content"])
        
        # Check if already indexed with same content
        existing = metadata_store.get(doc_id)
        if existing and existing.get("content_hash") == content_hash:
            continue  # No change, skip
        
        # Index or re-index
        if existing:
            # Update: delete old chunks, re-index
            vectorstore.delete(filter={"doc_id": doc_id})
        
        # Chunk and embed
        chunks = chunk_document(doc["content"], doc["metadata"])
        vectorstore.add_documents(chunks)
        
        # Update metadata
        metadata_store.update(doc_id, {
            "content_hash": content_hash,
            "indexed_at": datetime.utcnow().isoformat(),
            "version": doc["metadata"].get("version", "1.0")
        })
```

## Index Rebuilding Strategy

### Full vs. Incremental Rebuild

| Approach | When to Use | Time | Cost |
|---|---|---|---|
| **Full rebuild** | Schema change, model change, > 30% docs changed | Hours | High |
| **Incremental rebuild** | < 10% docs changed | Minutes | Low |
| **Real-time indexing** | Documents change continuously | Immediate | Medium |

### Scheduled Rebuild Pipeline

```python
def scheduled_rebuild(vectorstore, document_source, schedule: str = "daily"):
    """Rebuild index on schedule."""
    
    if schedule == "daily":
        # Check for changes since last rebuild
        last_rebuild = metadata_store.get("last_full_rebuild")
        changes = detect_changes_since(document_source, last_rebuild)
        
        if changes["total_changes"] > 0:
            # Incremental update
            for doc_id in changes["additions"]:
                index_document(document_source.get(doc_id), vectorstore)
            for doc_id in changes["updates"]:
                update_document(doc_id, document_source.get(doc_id), vectorstore)
            for doc_id in changes["deletions"]:
                delete_document(doc_id, vectorstore)
            
            metadata_store.update("last_incremental_update", datetime.utcnow().isoformat())
    
    elif schedule == "weekly":
        # Full rebuild weekly
        full_rebuild(document_source, vectorstore)
        metadata_store.update("last_full_rebuild", datetime.utcnow().isoformat())
```

## Freshness Monitoring

```python
def freshness_audit(vectorstore) -> dict:
    """Audit the freshness of all indexed documents."""
    
    all_docs = vectorstore.get_all_metadata()
    
    age_distribution = {"< 30d": 0, "30-90d": 0, "90-180d": 0, "180-365d": 0, "> 365d": 0}
    stale_docs = []
    
    for doc_meta in all_docs:
        age = (datetime.now() - doc_meta.get("indexed_at", datetime(2000,1,1))).days
        
        if age < 30:
            age_distribution["< 30d"] += 1
        elif age < 90:
            age_distribution["30-90d"] += 1
        elif age < 180:
            age_distribution["90-180d"] += 1
        elif age < 365:
            age_distribution["180-365d"] += 1
        else:
            age_distribution["> 365d"] += 1
            stale_docs.append(doc_meta.get("doc_id"))
    
    return {
        "total_documents": len(all_docs),
        "age_distribution": age_distribution,
        "stale_documents": stale_docs,
        "stale_percentage": len(stale_docs) / len(all_docs) * 100,
        "last_index_update": max(d.get("indexed_at", "") for d in all_docs)
    }
```

## Handling Policy Version Transitions

When a policy is updated, there may be a transition period where both old and new versions are relevant.

```python
def handle_policy_transition(old_version: dict, new_version: dict, 
                              vectorstore, transition_days: int = 30):
    """Handle policy version transition gracefully."""
    
    effective_date = new_version["effective_from"]
    transition_end = effective_date + timedelta(days=transition_days)
    
    # Index new version
    index_document(new_version, vectorstore)
    
    # Keep old version but mark as "transitioning"
    vectorstore.update_metadata(
        filter={"doc_id": old_version["id"]},
        update={"status": "transitioning", "superseded_by": new_version["id"]}
    )
    
    # After transition, archive old version
    if datetime.now() > transition_end:
        vectorstore.update_metadata(
            filter={"doc_id": old_version["id"]},
            update={"status": "archived"}
        )

def retrieval_with_transition(query: str, vectorstore, k: int = 4):
    """During transition, prefer new policy but mention old one if relevant."""
    
    results = vectorstore.similarity_search(query, k=k, filter={"status": "active"})
    
    # Check if any transitioning documents are relevant
    transitioning = vectorstore.similarity_search(query, k=1, filter={"status": "transitioning"})
    
    if transitioning and len(results) < k:
        results.extend(transitioning)
    
    return results
```

## Change Detection in Source Systems

### Method 1: Webhook-Based

```python
from flask import Flask, request

app = Flask(__name__)

@app.route("/webhook/document-change", methods=["POST"])
def handle_document_change():
    """Receive document change webhook from source system."""
    
    event = request.json
    doc_id = event["document_id"]
    change_type = event["change_type"]  # created, updated, deleted
    
    if change_type == "deleted":
        vectorstore.delete(filter={"doc_id": doc_id})
    elif change_type in ("created", "updated"):
        doc = document_service.get_document(doc_id)
        if change_type == "updated":
            vectorstore.delete(filter={"doc_id": doc_id})
        index_document(doc, vectorstore)
    
    return {"status": "ok"}
```

### Method 2: Database CDC (Change Data Capture)

```python
# Using Debezium or similar CDC tool
# Listen to PostgreSQL WAL (Write-Ahead Log)

from pglogical import ReplicationSlot

def listen_to_document_changes():
    """Listen to PostgreSQL changes on the documents table."""
    
    slot = ReplicationSlot("document_changes")
    
    for change in slot.changes():
        if change.table == "policy_documents":
            if change.type == "INSERT":
                index_document(change.new_data, vectorstore)
            elif change.type == "UPDATE":
                vectorstore.delete(filter={"doc_id": change.new_data["id"]})
                index_document(change.new_data, vectorstore)
            elif change.type == "DELETE":
                vectorstore.delete(filter={"doc_id": change.old_data["id"]})
```

### Method 3: Scheduled Polling

```python
def poll_for_changes(document_source, vectorstore, interval_minutes: int = 15):
    """Poll source system for document changes."""
    
    last_check = metadata_store.get("last_change_check", datetime.min)
    
    # Get documents modified since last check
    changed_docs = document_source.get_modified_since(last_check)
    
    for doc in changed_docs:
        if doc["status"] == "deleted":
            vectorstore.delete(filter={"doc_id": doc["id"]})
        else:
            # Delete old chunks and re-index
            vectorstore.delete(filter={"doc_id": doc["id"]})
            index_document(doc, vectorstore)
    
    metadata_store.update("last_change_check", datetime.utcnow())
```

## Best Practices

1. **Always track document version**: Every chunk must have version metadata
2. **Use content hashing**: Detect changes at the byte level, not just timestamps
3. **Prefer incremental updates**: Only re-index changed documents
4. **Full rebuild weekly/monthly**: Clean up any accumulated errors
5. **Monitor freshness**: Alert when documents are older than expected
6. **Handle transitions gracefully**: During policy changes, provide both old and new info
7. **Log all indexing events**: For audit trail and debugging
8. **Set freshness SLAs**: Critical policies should be indexed within 1 hour of change
9. **Test with stale data**: Regularly verify that old versions are properly excluded
