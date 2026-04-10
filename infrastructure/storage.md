# Storage Options

## Overview

This document covers storage options for banking GenAI workloads, including block storage, object storage, file storage, and specialized storage for vector databases and model artifacts.

---

## Storage Decision Framework

```
What are you storing?
├── Application code and configs → Block Storage (persistent volumes)
├── Documents, images, PDFs → Object Storage (S3, Blob)
├── Model artifacts (weights, adapters) → Object Storage + fast local cache
├── Vector embeddings → Vector Database (Pinecone, Milvus)
├── Relational data (users, transactions) → Database (PostgreSQL)
├── Cache data → In-memory (Redis)
├── Audit logs → Object Storage (immutable) + ELK
├── Shared files across pods → File Storage (EFS, Azure Files)
└── Training datasets → Object Storage + high-throughput data pipeline
```

---

## Block Storage

### Characteristics

- Attached to a single VM/container
- Low latency, high IOPS
- Sized independently of compute
- Supports snapshots and backups

### Block Storage Types

| Type | IOPS | Throughput | Latency | Use Case | Cost (per GB/month) |
|------|------|-----------|---------|----------|-------------------|
| **GP3 (AWS) / Premium SSD (Azure)** | 3,000-16,000 | 125-1,000 MB/s | < 1ms | General purpose, databases | £0.08-0.12 |
| **io2 Block Express (AWS)** | 256,000 | 4,000 MB/s | < 0.5ms | High-performance databases | £0.15 |
| **Ultra SSD (Azure)** | 160,000 | 4,000 MB/s | < 0.3ms | Latency-sensitive databases | £0.15 |
| **gp3 for K8s (EBS CSI)** | 3,000-16,000 | 125-1,000 MB/s | < 1ms | Kubernetes persistent volumes | £0.08 |

### Kubernetes Persistent Volumes

```yaml
# Persistent Volume Claim for database
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: postgresql-data
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: premium-ssd
  resources:
    requests:
      storage: 500Gi

# Fast local SSD for model cache (hostPath)
apiVersion: v1
kind: Pod
spec:
  containers:
    - name: triton
      volumeMounts:
        - name: model-cache
          mountPath: /models
  volumes:
    - name: model-cache
      hostPath:
        path: /mnt/local-ssd/models
        type: Directory
```

### Block Storage Best Practices

1. **Size independently of compute**: Separate storage scaling from compute scaling
2. **Use snapshots for backups**: Automated daily snapshots with 30-day retention
3. **Encrypt at rest**: All block storage encrypted with KMS/Key Vault
4. **Monitor IOPS and throughput**: Alert on sustained > 80% utilization
5. **Use Provisioned IOPS for databases**: Don't rely on burst for production databases

---

## Object Storage

### Characteristics

- Virtually unlimited capacity
- Accessed via HTTP API
- Highly durable (99.999999999%)
- Cost-effective for large datasets

### Object Storage Tiers

| Tier | Access Frequency | Cost (per GB/month) | Retrieval Time | Use Case |
|------|-----------------|---------------------|---------------|----------|
| **Hot (Standard)** | Frequent | £0.021 | Milliseconds | Active documents, model artifacts |
| **Cool** | Infrequent (30+ days) | £0.010 | Milliseconds | Archived documents |
| **Cold/Archive** | Rare (90+ days) | £0.002-0.004 | Hours | Compliance archives, old logs |

### Object Storage for GenAI

| Data Type | Storage Class | Lifecycle Policy | Encryption |
|-----------|--------------|-----------------|------------|
| Training datasets | Hot | Move to Cool after 90 days | SSE-KMS |
| Model artifacts | Hot | Versioned, never delete | SSE-KMS |
| Uploaded documents | Hot | Move to Cool after 30 days, delete after 7 years | SSE-KMS |
| Audit logs | Hot (first 30 days) -> Cold | Retain for 5 years, then delete | SSE-KMS + Object Lock |
| Embedding cache | Hot | TTL-based expiration | SSE-S3 |
| Customer documents | Hot | Per customer retention policy | SSE-KMS + CMEK |

### Object Storage Configuration

```yaml
# S3 Bucket Configuration (Terraform)
resource "aws_s3_bucket" "genai_documents" {
  bucket = "genai-prod-documents"

  versioning {
    enabled = true
  }

  server_side_encryption_configuration {
    rule {
      apply_server_side_encryption_by_default {
        sse_algorithm     = "aws:kms"
        kms_master_key_id = aws_kms_key.genai.arn
      }
    }
  }

  # Object Lock for compliance (WORM)
  object_lock_configuration {
    object_lock_enabled = "Enabled"
    rule {
      default_retention {
        mode = "COMPLIANCE"
        years = 5
      }
    }
  }

  lifecycle_rule {
    id = "archive-old-documents"
    enabled = true

    transition {
      days          = 30
      storage_class = "STANDARD_IA"
    }

    transition {
      days          = 365
      storage_class = "GLACIER"
    }

    expiration {
      days = 2555  # 7 years
    }
  }
}
```

---

## File Storage

### Characteristics

- Shared access from multiple compute instances
- POSIX-compatible file system interface
- Network-attached (NFS, SMB)

### File Storage Options

| Service | Protocol | Max Throughput | Latency | Use Case | Cost |
|---------|----------|---------------|---------|----------|------|
| **EFS (AWS)** | NFSv4 | 10 GB/s | 2-5 ms | Shared configs, model sharing | £0.08/GB |
| **Azure Files** | SMB/NFS | 20 GB/s | 1-3 ms | Windows workloads, shared data | £0.07/GB |
| **FSx for Lustre** | Lustre | 1 TB/s | < 1 ms | High-performance ML training | £0.14/GB |
| **Filestore (GCP)** | NFS | 32 GB/s | < 1 ms | ML workloads | £0.09/GB |

### When to Use File Storage

- Multiple pods need access to the same files (model weights, configuration)
- Legacy applications requiring POSIX file system access
- ML training requiring high-throughput shared data access
- Home directories for developer environments

### When NOT to Use File Storage

- High IOPS workloads (use block storage)
- Large-scale object storage (use S3/Blob)
- Embedding storage (use vector database)
- Transactional data (use database)

---

## Specialized Storage for GenAI

### Vector Database Storage

| Aspect | Requirement | Recommended Solution |
|--------|------------|---------------------|
| **Embedding storage** | Low latency, high-dimensional similarity search | Pinecone, Milvus, Weaviate |
| **Metadata storage** | Structured queries, filtering | PostgreSQL with pgvector |
| **Index storage** | Fast rebuild, persistence | SSD-backed, in-memory indexes |
| **Backup** | Point-in-time recovery | Managed backup or snapshot |

### Model Artifact Storage

| Aspect | Requirement | Recommended Solution |
|--------|------------|---------------------|
| **Model weights** | Large files (1-100 GB), infrequent updates | S3/Blob with versioning |
| **LoRA adapters** | Small files (10-500 MB), frequent updates | S3/Blob with CDN |
| **Local cache** | Fast loading for inference | NVMe SSD on GPU nodes |
| **Versioning** | Rollback capability | Object storage versioning |

### Training Data Storage

| Aspect | Requirement | Recommended Solution |
|--------|------------|---------------------|
| **Raw datasets** | Large (TB-scale), sequential read | S3/Blob + high-throughput pipeline |
| **Processed datasets** | Structured, queryable | Parquet on S3 + Athena/Glue |
| **Feature store** | Low latency, point lookups | Redis, DynamoDB |
| **Training checkpoints** | Frequent writes, large size | Fast local SSD + periodic S3 sync |

---

## Data Lifecycle Management

### Document Lifecycle

```
Upload → Hot Storage (active processing)
    |
    v (30 days, no access)
Cool Storage (infrequent access)
    |
    v (1 year, no access)
Cold Storage (archive)
    |
    v (7 years total)
Secure Deletion
```

### Model Lifecycle

```
Training → Model Registry → Staging Deployment → Production
    |                                                |
    v                                                v
Checkpoint Storage                           Hot Object Storage
(S3, frequent writes)                        (versioned, immutable)
                                                  |
                                                  v (superseded by new version)
                                             Archive Storage
                                             (kept for rollback, 1 year)
```

### Audit Log Lifecycle

```
Production → Hot ELK Storage (30 days, searchable)
    |
    v (after 30 days)
Cold Object Storage (5 years, immutable)
    |
    v (after 5 years)
Secure Deletion (per regulatory requirements)
```

---

## Storage Security

### Encryption

| Storage Type | At Rest | In Transit | Key Management |
|-------------|---------|-----------|---------------|
| Block Storage | AES-256 | TLS 1.2+ | KMS/Key Vault |
| Object Storage | SSE-S3, SSE-KMS | TLS 1.2+ | KMS/Key Vault |
| File Storage | AES-256 | TLS 1.2+ (in-transit encryption) | KMS/Key Vault |
| Vector DB | Provider encryption or self-managed | TLS 1.2+ | Provider or KMS |
| Database | TDE (Transparent Data Encryption) | TLS 1.2+ | KMS/Key Vault |

### Access Control

| Storage Type | Access Control | Authentication |
|-------------|---------------|---------------|
| S3/Blob | IAM policies, bucket policies | IAM roles, SAS tokens |
| EBS/Managed Disks | IAM, resource policies | IAM roles |
| EFS/Azure Files | POSIX permissions, IAM | IAM, AD integration |
| Vector DB | API keys, RBAC | API keys, OAuth |
| Database | Database roles, row-level security | IAM database auth, passwords |

### Immutable Storage for Compliance

For regulatory audit trails:
- S3 Object Lock (Governance or Compliance mode)
- Azure Blob immutable storage (time-based retention)
- WORM (Write Once, Read Many) enforcement
- Legal hold capability for litigation

```yaml
# Immutable storage configuration
object_lock:
  enabled: true
  mode: COMPLIANCE  # Cannot be overridden, even by root
  retention_period: 5 years

legal_hold:
  enabled: true
  authorized_roles:
    - legal-counsel
    - compliance-officer
```

---

## Storage Performance Optimization

### Caching Strategy

```
┌─────────────────────────────────────────────────────┐
│                Storage Caching Layers                │
│                                                      │
│  Application Request                                 │
│       │                                              │
│       v                                              │
│  ┌─────────────┐     Cache Miss                     │
│  │  L1 Cache   │ ──────────────────────────────────►│
│  │ (In-memory) │                                     │
│  │  Redis      │                                     │
│  └──────┬──────┘                                     │
│         │ Cache Miss                                  │
│         v                                            │
│  ┌─────────────┐                                     │
│  │  L2 Cache   │ ──────────────────────────────────►│
│  │ (Local SSD) │                                     │
│  │  NVMe       │                                     │
│  └──────┬──────┘                                     │
│         │ Cache Miss                                  │
│         v                                            │
│  ┌─────────────┐                                     │
│  │  L3 Cache   │ ──────────────────────────────────►│
│  │ (Object     │                                     │
│  │  Storage)   │                                     │
│  └─────────────┘                                     │
└─────────────────────────────────────────────────────┘
```

### Performance Benchmarks

| Storage Type | Read Latency | Write Latency | IOPS | Throughput |
|-------------|-------------|--------------|------|------------|
| Redis (in-memory) | < 1ms | < 1ms | 100K+ | 10 GB/s |
| NVMe SSD | < 0.1ms | < 0.1ms | 500K+ | 7 GB/s |
| Premium SSD | < 1ms | < 1ms | 20K-80K | 1 GB/s |
| S3/Blob | 50-150ms | 50-150ms | N/A | Unlimited |
| Vector DB (ANN search) | 10-50ms | N/A | Query-dependent | Query-dependent |

---

## Storage Cost Optimization

| Strategy | Savings | Notes |
|----------|---------|-------|
| **Lifecycle policies** | 40-70% on storage costs | Automate hot -> cool -> cold transitions |
| **Intelligent tiering** | 20-40% | Provider automatically moves data based on access patterns |
| **Compression** | 50-80% on storage size | Compress documents before storage |
| **Deduplication** | 30-60% | Remove duplicate documents |
| **Right-sizing volumes** | 20-40% | Monitor utilization and resize |
| **Delete unused data** | 10-30% | Regular data cleanup |

---

## Cross-References

- [README.md](README.md) -- Infrastructure overview
- [cloud-providers.md](cloud-providers.md) -- Cloud provider selection
- [compute.md](compute.md) -- Compute options
- [../databases/README.md](../databases/README.md) -- Database practices
- [../databases/vector-databases.md](../databases/vector-databases.md) -- Vector database details
- [../security/README.md](../security/README.md) -- Security practices
