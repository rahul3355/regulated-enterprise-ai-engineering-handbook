# Incremental Indexing

## Overview

Incremental indexing updates only the changed portions of your vector index rather than rebuilding from scratch. This is essential for production systems where:
- Documents are updated daily or hourly
- Full reindexing takes hours and is expensive
- Users need access to the latest information quickly

## When to Use Incremental vs. Full Rebuild

| Scenario | Approach | Reason |
|---|---|---|
| New document added | Incremental | Minimal change |
| Single document updated | Incremental | Localized change |
| < 5% of documents changed | Incremental | Efficient |
| Embedding model changed | Full rebuild | All embeddings need regeneration |
| Chunking strategy changed | Full rebuild | All chunks changed |
| > 30% of documents changed | Full rebuild | Incremental overhead exceeds rebuild |
| Schema change | Full rebuild | Structure changed |
| Monthly maintenance | Full rebuild | Clean up accumulated inconsistencies |

## Change Detection Methods

### Method 1: Content Hash Comparison

The most reliable method -- detect changes at the byte level.

```python
import hashlib

def compute_document_hash(content: str) -> str:
    """Compute SHA-256 hash of document content."""
    return hashlib.sha256(content.encode("utf-8")).hexdigest()

def detect_changes(current_docs: dict, indexed_metadata: dict) -> dict:
    """Detect additions, updates, and deletions."""
    
    current_ids = set(current_docs.keys())
    indexed_ids = set(indexed_metadata.keys())
    
    additions = current_ids - indexed_ids
    deletions = indexed_ids - current_ids
    common = current_ids & indexed_ids
    
    updates = set()
    for doc_id in common:
        current_hash = compute_document_hash(current_docs[doc_id]["content"])
        indexed_hash = indexed_metadata[doc_id].get("content_hash")
        
        if current_hash != indexed_hash:
            updates.add(doc_id)
    
    return {
        "additions": list(additions),
        "updates": list(updates),
        "deletions": list(deletions),
        "unchanged": list(common - updates),
        "total_changes": len(additions) + len(updates) + len(deletions)
    }
```

### Method 2: Timestamp-Based Detection

Faster but less reliable -- relies on modification timestamps.

```python
def detect_changes_by_timestamp(doc_source, last_index_time: datetime) -> list[dict]:
    """Find documents modified since last indexing."""
    
    return doc_source.query(
        "SELECT * FROM documents WHERE updated_at > %s",
        (last_index_time,)
    )
```

### Method 3: CDC (Change Data Capture)

Most accurate for database-backed documents.

```python
# Using Debezium to capture PostgreSQL changes
# Debezium streams changes from PostgreSQL WAL log

debezium_config = {
    "connector.class": "io.debezium.connector.postgresql.PostgresConnector",
    "database.hostname": "postgres-host",
    "database.port": "5432",
    "database.user": "debezium",
    "database.password": "password",
    "database.dbname": "bank_knowledge",
    "table.include.list": "public.policy_documents",
    "plugin.name": "pgoutput",
    "publication.name": "dbz_publication",
}

# Each change generates an event:
# {
#   "before": {"id": "...", "content": "...", "updated_at": "..."},
#   "after": {"id": "...", "content": "...", "updated_at": "..."},
#   "op": "u",  // c=create, u=update, d=delete
#   "ts_ms": 1234567890
# }
```

## Incremental Update Operations

### Adding New Documents

```python
def add_documents(documents: list[dict], pipeline) -> dict:
    """Add new documents to the index."""
    
    stats = {"added": 0, "errors": 0}
    
    for doc in documents:
        try:
            # Chunk
            chunks = pipeline.chunker.chunk(doc)
            
            # Embed
            chunks = pipeline.embedder.embed_batch(chunks)
            
            # Store
            pipeline.vectorstore.add_documents(chunks)
            
            # Update metadata
            pipeline.metadata_store.update(doc["id"], {
                "content_hash": compute_document_hash(doc["content"]),
                "indexed_at": datetime.utcnow().isoformat(),
                "chunk_count": len(chunks),
                "status": "indexed",
            })
            
            stats["added"] += 1
        except Exception as e:
            stats["errors"] += 1
            log_error(f"Error adding doc {doc.get('id')}: {e}")
    
    return stats
```

### Updating Existing Documents

```python
def update_documents(documents: list[dict], pipeline) -> dict:
    """Update documents: delete old chunks, add new ones."""
    
    stats = {"updated": 0, "errors": 0, "old_chunks_deleted": 0, "new_chunks_added": 0}
    
    for doc in documents:
        try:
            doc_id = doc["id"]
            
            # Step 1: Delete old chunks for this document
            old_chunk_count = pipeline.vectorstore.delete(filter={"doc_id": doc_id})
            stats["old_chunks_deleted"] += old_chunk_count
            
            # Step 2: Chunk and embed new version
            chunks = pipeline.chunker.chunk(doc)
            chunks = pipeline.embedder.embed_batch(chunks)
            
            # Step 3: Store new chunks
            pipeline.vectorstore.add_documents(chunks)
            stats["new_chunks_added"] += len(chunks)
            
            # Step 4: Update metadata
            pipeline.metadata_store.update(doc_id, {
                "content_hash": compute_document_hash(doc["content"]),
                "indexed_at": datetime.utcnow().isoformat(),
                "chunk_count": len(chunks),
                "status": "indexed",
                "previous_version": pipeline.metadata_store.get(doc_id, {}).get("version"),
                "version": doc.get("metadata", {}).get("version", "unknown"),
            })
            
            stats["updated"] += 1
        except Exception as e:
            stats["errors"] += 1
            log_error(f"Error updating doc {doc.get('id')}: {e}")
    
    return stats
```

### Deleting Documents

```python
def delete_documents(doc_ids: list[str], vectorstore, metadata_store) -> dict:
    """Remove documents from the index."""
    
    stats = {"deleted": 0, "errors": 0}
    
    for doc_id in doc_ids:
        try:
            # Delete chunks from vector store
            deleted_count = vectorstore.delete(filter={"doc_id": doc_id})
            
            # Remove from metadata
            metadata_store.delete(doc_id)
            
            stats["deleted"] += 1
        except Exception as e:
            stats["errors"] += 1
            log_error(f"Error deleting doc {doc_id}: {e}")
    
    return stats
```

## Full Incremental Indexing Pipeline

```python
class IncrementalIndexer:
    def __init__(self, pipeline, document_source, metadata_store):
        self.pipeline = pipeline
        self.document_source = document_source
        self.metadata_store = metadata_store
    
    def run_incremental(self) -> dict:
        """Run incremental indexing."""
        
        last_update = self.metadata_store.get("last_incremental_update")
        if last_update:
            last_update = datetime.fromisoformat(last_update)
        else:
            # First run, do full indexing
            return self.run_full()
        
        # Step 1: Detect changes
        current_docs = self.document_source.get_all()
        indexed_metadata = self.metadata_store.get_all_document_metadata()
        
        changes = detect_changes(current_docs, indexed_metadata)
        
        print(f"Changes detected: {len(changes['additions'])} additions, "
              f"{len(changes['updates'])} updates, {len(changes['deletions'])} deletions")
        
        # Step 2: Apply changes
        total_stats = {"additions": {}, "updates": {}, "deletions": {}}
        
        if changes["additions"]:
            new_docs = [current_docs[doc_id] for doc_id in changes["additions"]]
            total_stats["additions"] = add_documents(new_docs, self.pipeline)
        
        if changes["updates"]:
            updated_docs = [current_docs[doc_id] for doc_id in changes["updates"]]
            total_stats["updates"] = update_documents(updated_docs, self.pipeline)
        
        if changes["deletions"]:
            total_stats["deletions"] = delete_documents(
                changes["deletions"],
                self.pipeline.vectorstore,
                self.metadata_store
            )
        
        # Step 3: Update tracking
        self.metadata_store.update("last_incremental_update", datetime.utcnow().isoformat())
        
        return {
            "changes": changes,
            "stats": total_stats,
            "timestamp": datetime.utcnow().isoformat(),
            "type": "incremental"
        }
    
    def run_full(self) -> dict:
        """Full reindexing."""
        
        print("Running full reindex...")
        
        # Clear existing index
        self.pipeline.vectorstore.clear()
        
        # Run pipeline on all documents
        all_docs = list(self.document_source.get_all().values())
        stats = self.pipeline.run(iter(all_docs))
        
        self.metadata_store.update("last_full_rebuild", datetime.utcnow().isoformat())
        self.metadata_store.update("last_incremental_update", datetime.utcnow().isoformat())
        
        return {**stats, "type": "full"}
    
    def should_full_rebuild(self, changes: dict, force: bool = False) -> bool:
        """Decide whether to do a full rebuild instead of incremental."""
        
        if force:
            return True
        
        total_docs = self.metadata_store.get_document_count()
        change_ratio = changes["total_changes"] / max(total_docs, 1)
        
        # Full rebuild if > 30% changed
        if change_ratio > 0.30:
            return True
        
        # Full rebuild on first run
        if total_docs == 0:
            return True
        
        return False
```

## Scheduled Incremental Indexing

```python
import schedule
import time

def setup_indexing_schedule(indexer):
    """Set up scheduled incremental indexing."""
    
    # Incremental every 15 minutes
    schedule.every(15).minutes.do(lambda: indexer.run_incremental())
    
    # Full rebuild weekly (Sunday at 2 AM)
    schedule.every().sunday.at("02:00").do(lambda: indexer.run_full())
    
    # Full rebuild monthly on the 1st
    schedule.every().month.do(lambda: indexer.run_full())
    
    while True:
        schedule.run_pending()
        time.sleep(60)
```

## Handling Indexing Failures

```python
class ResumableIndexer:
    """Indexer that can resume from failures."""
    
    def __init__(self, pipeline, state_store):
        self.pipeline = pipeline
        self.state_store = state_store
    
    def run_with_checkpointing(self, documents: list[dict]) -> dict:
        """Run indexing with checkpoint recovery."""
        
        # Load checkpoint
        checkpoint = self.state_store.get("indexing_checkpoint", {})
        processed_ids = set(checkpoint.get("processed_ids", []))
        
        stats = {"processed": len(processed_ids), "errors": 0, "skipped": 0}
        
        for i, doc in enumerate(documents):
            if doc["id"] in processed_ids:
                stats["skipped"] += 1
                continue
            
            try:
                # Process document
                chunks = self.pipeline.chunker.chunk(doc)
                chunks = self.pipeline.embedder.embed_batch(chunks)
                self.pipeline.vectorstore.add_documents(chunks)
                
                # Update checkpoint
                processed_ids.add(doc["id"])
                self.state_store.update("indexing_checkpoint", {
                    "processed_ids": list(processed_ids),
                    "last_processed_index": i,
                    "timestamp": datetime.utcnow().isoformat(),
                })
                
                stats["processed"] += 1
                
            except Exception as e:
                stats["errors"] += 1
                log_error(f"Error at doc {i} ({doc.get('id')}): {e}")
                # Continue with next document (don't abort entire batch)
        
        # Clear checkpoint on successful completion
        self.state_store.delete("indexing_checkpoint")
        
        return stats
```

## Monitoring Incremental Indexing

```python
def indexing_health_check(metadata_store, vectorstore) -> dict:
    """Check health of the indexing system."""
    
    doc_metadata = metadata_store.get_all_document_metadata()
    vector_count = vectorstore.count()
    
    # Expected chunks: sum of chunk_count across all documents
    expected_chunks = sum(m.get("chunk_count", 0) for m in doc_metadata.values())
    
    checks = {
        "metadata_doc_count": len(doc_metadata),
        "vector_store_count": vector_count,
        "expected_chunk_count": expected_chunks,
        "vector_count_matches": abs(vector_count - expected_chunks) < expected_chunks * 0.05,
        "last_incremental_update": metadata_store.get("last_incremental_update"),
        "last_full_rebuild": metadata_store.get("last_full_rebuild"),
    }
    
    # Check for stale documents
    stale_threshold = datetime.utcnow() - timedelta(days=7)
    stale_docs = [
        doc_id for doc_id, meta in doc_metadata.items()
        if datetime.fromisoformat(meta.get("indexed_at", "2000-01-01")) < stale_threshold
    ]
    checks["stale_documents"] = stale_docs
    
    # Check for orphaned vectors (vectors without metadata)
    checks["possible_orphaned_vectors"] = max(0, vector_count - expected_chunks)
    
    return checks
```

## Best Practices

1. **Always use content hashes**: Timestamps can be unreliable
2. **Checkpoint frequently**: Save progress every 100 documents
3. **Handle failures gracefully**: One bad document shouldn't stop the pipeline
4. **Monitor drift**: Track the gap between vector store and source system
5. **Schedule full rebuilds**: Weekly or monthly to clean up accumulated errors
6. **Log all operations**: For audit and debugging
7. **Test incremental on staging**: Verify it produces same results as full rebuild
8. **Version your pipeline**: Track which pipeline version indexed documents
9. **Alert on anomalies**: Sudden spike in changes may indicate a bug
10. **Document the process**: So other engineers understand the indexing strategy
