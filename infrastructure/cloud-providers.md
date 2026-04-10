# Cloud Providers for Banking GenAI

## Overview

This document compares AWS, Azure, and GCP for banking GenAI workloads, covering compute, storage, networking, AI/ML services, security, compliance, and cost. It provides decision criteria and our recommended cloud strategy.

---

## Cloud Provider Comparison

### AI/ML and GenAI Services

| Capability | AWS | Azure | GCP |
|-----------|-----|-------|-----|
| **Managed LLM** | Bedrock (Claude, Llama, AI21) | Azure OpenAI (GPT-4, GPT-4o) | Vertex AI (Gemini, Claude) |
| **Embedding Models** | Bedrock (Titan Embeddings) | Azure OpenAI (text-embedding-ada) | Vertex AI (textembedding) |
| **GPU Instances** | p4d, p5 (A100, H100) | ND A100 v4, ND H100 v5 | A3 (H100), A2 (A100) |
| **ML Platform** | SageMaker | Azure ML | Vertex AI |
| **Vector DB (Managed)** | OpenSearch (vector), MemoryDB | Azure AI Search | Vertex AI Vector Search |
| **Feature Store** | SageMaker Feature Store | Azure ML Feature Store | Vertex AI Feature Store |
| **Model Registry** | SageMaker Model Registry | Azure ML Model Registry | Vertex AI Model Registry |

### Compute Options

| Service Type | AWS | Azure | GCP |
|-------------|-----|-------|-----|
| **VMs** | EC2 | Azure VMs | Compute Engine |
| **Containers** | ECS, EKS | AKS | GKE |
| **Serverless** | Lambda | Azure Functions | Cloud Functions |
| **GPU VMs** | p4d, p5, g5, g6 | ND series | A2, A3 |
| **Spot/Preemptible** | Spot Instances | Spot VMs | Preemptible VMs |
| **Bare Metal** | EC2 Bare Metal | Azure BareMetal | Sole-Tenant Nodes |

### Storage

| Service Type | AWS | Azure | GCP |
|-------------|-----|-------|-----|
| **Object Storage** | S3 | Blob Storage | Cloud Storage |
| **Block Storage** | EBS | Managed Disks | Persistent Disks |
| **File Storage** | EFS, FSx | Azure Files | Filestore |
| **Archive Storage** | S3 Glacier | Archive Blob Storage | Nearline/Coldline |
| **Key Management** | KMS | Key Vault | Cloud KMS |

### Networking

| Service Type | AWS | Azure | GCP |
|-------------|-----|-------|-----|
| **VPC** | VPC | VNet | VPC |
| **Load Balancer** | ALB, NLB | Application Gateway, ALB | Cloud Load Balancing |
| **CDN** | CloudFront | Azure CDN | Cloud CDN |
| **Edge Computing** | Lambda@Edge | Azure Front Door | Cloudflare (partner) |
| **Private Connectivity** | Direct Connect | ExpressRoute | Cloud Interconnect |
| **DNS** | Route 53 | Azure DNS | Cloud DNS |

---

## Compliance and Security

### Banking-Specific Compliance

| Compliance | AWS | Azure | GCP |
|------------|-----|-------|-----|
| **FCA Compliance** | FCA-approved regions, UK South | UK South, full FCA support | London region, FCA support |
| **GDPR** | Full GDPR compliance | Full GDPR compliance | Full GDPR compliance |
| **PCI DSS** | PCI DSS Level 1 | PCI DSS Level 1 | PCI DSS Level 1 |
| **SOC 1/2/3** | Certified | Certified | Certified |
| **ISO 27001** | Certified | Certified | Certified |
| **Cyber Essentials Plus** | Certified | Certified | Certified |

### Data Residency

| Requirement | AWS | Azure | GCP |
|-------------|-----|-------|-----|
| **UK Data Residency** | London (eu-west-2) | UK South, UK West | London (europe-west2) |
| **EU Data Residency** | Frankfurt, Ireland | West Europe, North Europe | Frankfurt, Belgium |
| **Data Residency Controls** | SCP policies, resource policies | Azure Policy, region lock | Organization policies |
| **Cross-Region Replication** | Configurable per service | Configurable per service | Configurable per service |

### Security Features

| Feature | AWS | Azure | GCP |
|---------|-----|-------|-----|
| **IAM** | IAM (fine-grained) | Azure AD + RBAC | Cloud IAM |
| **Encryption at Rest** | SSE-S3, SSE-KMS | Storage Service Encryption | CMEK, CSEK |
| **Encryption in Transit** | TLS 1.2+ | TLS 1.2+ | TLS 1.2+ |
| **Secret Management** | Secrets Manager | Key Vault | Secret Manager |
| **Network Security** | Security Groups, NACLs | NSGs, Application Security Groups | Firewall rules |
| **WAF** | AWS WAF | Azure WAF | Cloud Armor |
| **DDoS Protection** | AWS Shield | Azure DDoS Protection | Google Cloud Armor |

---

## Cost Comparison

### GPU Instance Pricing (On-Demand, Hourly)

| Instance | GPU | AWS | Azure | GCP |
|----------|-----|-----|-------|-----|
| **A100 80GB (single)** | 1x A100 80GB | ~$3.50 (p4de) | ~$3.70 (ND96amsr) | ~$3.30 (a2-ultragpu-1g) |
| **A100 80GB (8x)** | 8x A100 80GB | ~$28.00 (p4de.24xlarge) | ~$29.60 (ND96amsr) | ~$26.40 (a2-ultragpu-8g) |
| **H100 80GB (single)** | 1x H100 80GB | ~$5.00 (p5) | ~$5.50 (ND H100) | ~$4.80 (a3-highgpu-1g) |
| **L40S (single)** | 1x L40S | ~$1.80 (g6) | ~$2.00 (NC L40S) | N/A |

### Savings Options

| Option | AWS | Azure | GCP |
|--------|-----|-------|-----|
| **1-Year Reserved** | ~40% discount | ~40% discount | ~37% discount |
| **3-Year Reserved** | ~60% discount | ~60% discount | ~55% discount |
| **Spot/Preemptible** | 60-90% discount | 60-90% discount | 60-80% discount |
| **Savings Plans** | 17-54% discount | N/A | Committed use discounts |

### LLM API Pricing

| Model | Azure OpenAI | AWS Bedrock | GCP Vertex AI |
|-------|-------------|-------------|---------------|
| **GPT-4** | £0.03/1K input, £0.09/1K output | N/A | N/A |
| **GPT-4-Turbo** | £0.01/1K input, £0.03/1K output | N/A | N/A |
| **Claude 3 Opus** | N/A | £0.015/1K input, £0.075/1K output | N/A |
| **Claude 3 Sonnet** | N/A | £0.003/1K input, £0.015/1K output | N/A |
| **Gemini Pro** | N/A | N/A | £0.00125/1K input, £0.00375/1K output |
| **Llama 3 70B** | N/A | £0.00265/1K input, £0.0035/1K output | N/A |

---

## Our Cloud Strategy

### Primary Cloud: Azure

**Rationale:**
- Azure OpenAI provides native access to GPT-4 and GPT-4-Turbo
- Strongest enterprise integration (Active Directory, Microsoft ecosystem)
- UK South region for data residency
- Comprehensive compliance certifications for banking
- Existing Microsoft relationship and enterprise agreement

### Secondary Cloud: AWS

**Rationale:**
- Bedrock provides access to Claude, Llama, and other models
- Strongest GPU instance availability (p5 with H100)
- Mature Kubernetes offering (EKS)
- Multi-cloud risk mitigation
- Used for self-hosted model inference and training

### Multi-Cloud Strategy

```
┌─────────────────────────────────────────────┐
│              Multi-Cloud Architecture         │
│                                              │
│  ┌──────────────┐    ┌──────────────┐       │
│  │    Azure      │    │     AWS      │       │
│  │               │    │              │       │
│  │ - Azure OpenAI│    │ - Bedrock    │       │
│  │ - AKS         │    │ - EKS        │       │
│  │ - Azure AD    │    │ - IAM        │       │
│  │ - Blob Storage│    │ - S3         │       │
│  │ - UK South    │    │ - London     │       │
│  └───────┬───────┘    └──────┬───────┘       │
│          │                   │               │
│          └────────┬──────────┘               │
│                   │                          │
│          ┌────────┴────────┐                 │
│          │  Cloud Abstraction               │
│          │  (Terraform + K8s)               │
│          └─────────────────┘                 │
└─────────────────────────────────────────────┘
```

### Decision Criteria for Workload Placement

| Workload | Primary Placement | Rationale |
|----------|------------------|-----------|
| **LLM API (GPT-4)** | Azure OpenAI | Native GPT-4 access |
| **LLM API (Claude)** | AWS Bedrock | Native Claude access |
| **Self-hosted LLM Inference** | AWS (GPU instances) | Better GPU availability and pricing |
| **Training Jobs** | AWS (p5/H100) | Best GPU training instances |
| **Vector Database** | Azure (AI Search) or self-hosted | Integration with Azure ecosystem |
| **Application Services** | Azure (AKS) | Primary cloud, AD integration |
| **Data Storage** | Azure (Blob, UK South) | Data residency, compliance |
| **Audit Logs** | Both clouds (immutable storage) | Redundancy, compliance |

---

## Cloud Cost Optimization

### Reserved Capacity Strategy

1. **Azure Reserved VM Instances**: 3-year reservation for baseline AKS nodes (60% savings)
2. **AWS Savings Plans**: 1-year commitment for baseline compute (30-50% savings)
3. **LLM API Committed Spend**: Azure OpenAI committed tier for volume discounts

### Spot/Preemptible Usage

1. **Training Jobs**: Run on spot instances (70% savings, checkpointing handles preemption)
2. **Batch Processing**: Embedding pipeline on spot instances
3. **Development Environments**: Spot instances for non-production

### Auto-Scaling

1. **GPU Instances**: Scale based on inference queue depth
2. **CPU Instances**: Scale based on request rate
3. **Serverless**: Use serverless for variable workloads (event processing)

### Right-Sizing

1. **Monthly Review**: Analyze utilization and right-size instances
2. **GPU Utilization Target**: > 70% utilization for production GPU instances
3. **Storage Lifecycle**: Move infrequently accessed data to cheaper tiers

---

## Cloud Risk Management

### Vendor Lock-In Mitigation

1. **Abstraction Layers**: Terraform for infrastructure, Kubernetes for workloads
2. **Multi-Provider LLM Routing**: Abstract LLM provider selection from application logic
3. **Data Portability**: Regular data exports in standard formats
4. **Exit Plan**: Documented cloud exit strategy with estimated timeline and cost

### Availability Risk Mitigation

1. **Multi-Region**: Critical services deployed in multiple regions
2. **Multi-Cloud**: Critical workloads can fail over between clouds
3. **Regular Testing**: Quarterly failover testing between regions and clouds

### Compliance Risk Mitigation

1. **Policy as Code**: OPA/Gatekeeper for compliance enforcement
2. **Automated Auditing**: Continuous compliance scanning
3. **Regular Reviews**: Quarterly compliance reviews with legal and compliance teams

---

## Cloud Provider Evaluation Checklist

When evaluating a new cloud service:

- [ ] **Compliance**: Does the service meet our regulatory requirements (FCA, GDPR, PCI DSS)?
- [ ] **Data Residency**: Can data be kept in required regions?
- [ ] **Security**: Does the service support encryption, IAM, and audit logging?
- [ ] **Performance**: Does the service meet our latency and throughput requirements?
- [ ] **Availability**: What is the SLA? Does it meet our SLO?
- [ ] **Cost**: What is the cost model? Can we optimize with reservations?
- [ ] **Integration**: Does it integrate with our existing tools and processes?
- [ ] **Lock-In**: What is the migration cost if we need to move away?
- [ ] **Support**: What support levels are available?
- [ ] **Roadmap**: What is the provider's investment in this service?

---

## Cross-References

- [README.md](README.md) -- Infrastructure overview
- [networking.md](networking.md) -- Network architecture
- [compute.md](compute.md) -- Compute options
- [storage.md](storage.md) -- Storage options
- [infrastructure-cost-management.md](infrastructure-cost-management.md) -- Cost management
- [../security/README.md](../security/README.md) -- Security practices
- [../kubernetes-openshift/README.md](../kubernetes-openshift/README.md) -- Kubernetes operations
