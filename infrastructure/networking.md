# Network Architecture

## Overview

This document defines the network architecture principles, VPC design, connectivity patterns, and security controls for the banking GenAI platform. The network is the foundation upon which all security, reliability, and performance guarantees are built.

---

## Network Architecture Principles

### 1. Defense in Depth

Multiple layers of network security ensure that a single control failure does not result in a breach:
- Perimeter security (WAF, DDoS protection)
- Network segmentation (VPCs, subnets, security groups)
- Micro-segmentation (pod-level network policies)
- Application-level security (mTLS, authentication)

### 2. Least Privilege Networking

Every component only has the network access it needs:
- Private subnets for all application workloads
- No direct internet access from application pods
- Egress through controlled NAT gateway or proxy
- Ingress through load balancers and API gateways only

### 3. Zero Trust

No implicit trust based on network location:
- mTLS between all services (service mesh)
- Authentication and authorization for every request
- Network policies enforce communication patterns
- Audit all network traffic

### 4. High Availability

Network design supports multi-AZ and multi-region resilience:
- Subnets spread across at least 3 availability zones
- Cross-AZ load balancing
- Multi-region DNS failover
- Redundant network paths

---

## VPC Design

### Recommended VPC Architecture

```
┌──────────────────────────────────────────────────────────────┐
│                        VPC: genai-prod                       │
│                        CIDR: 10.0.0.0/16                    │
│                                                              │
│  ┌─────────────────────────────────────────────────────┐    │
│  │              Public Subnets (3 AZs)                  │    │
│  │  10.0.1.0/24, 10.0.2.0/24, 10.0.3.0/24             │    │
│  │                                                      │    │
│  │  - Load Balancers (ALB/NLB)                          │    │
│  │  - NAT Gateways (one per AZ)                         │    │
│  │  - Bastion Hosts (for admin access)                  │    │
│  └──────────────────────┬──────────────────────────────┘    │
│                         │                                   │
│  ┌──────────────────────┴──────────────────────────────┐    │
│  │            Application Subnets (3 AZs)               │    │
│  │  10.0.10.0/24, 10.0.11.0/24, 10.0.12.0/24          │    │
│  │                                                      │    │
│  │  - Kubernetes worker nodes (AKS/EKS)                 │    │
│  │  - Application pods (AssistBot, WealthAdvisor, etc.)│    │
│  │  - API Gateway                                       │    │
│  └──────────────────────┬──────────────────────────────┘    │
│                         │                                   │
│  ┌──────────────────────┴──────────────────────────────┐    │
│  │             Data Subnets (3 AZs)                     │    │
│  │  10.0.20.0/24, 10.0.21.0/24, 10.0.22.0/24          │    │
│  │                                                      │    │
│  │  - Databases (PostgreSQL, Redis)                     │    │
│  │  - Vector databases (Pinecone, Milvus)               │    │
│  │  - Cache (Redis, Memcached)                          │    │
│  └──────────────────────┬──────────────────────────────┘    │
│                         │                                   │
│  ┌──────────────────────┴──────────────────────────────┐    │
│  │            GPU Subnets (3 AZs)                       │    │
│  │  10.0.30.0/24, 10.0.31.0/24, 10.0.32.0/24          │    │
│  │                                                      │    │
│  │  - GPU inference nodes (Triton, vLLM)                │    │
│  │  - GPU training nodes                                │    │
│  └─────────────────────────────────────────────────────┘    │
│                                                              │
│  ┌─────────────────────────────────────────────────────┐    │
│  │           Management Subnets (3 AZs)                 │    │
│  │  10.0.40.0/24, 10.0.41.0/24, 10.0.42.0/24          │    │
│  │                                                      │    │
│  │  - Monitoring (Prometheus, Grafana)                  │    │
│  │  - Logging (ELK)                                     │    │
│  │  - CI/CD runners                                     │    │
│  └─────────────────────────────────────────────────────┘    │
└──────────────────────────────────────────────────────────────┘
```

### Subnet Sizing

| Subnet Type | CIDR | IPs per AZ | Purpose |
|-------------|------|-----------|---------|
| Public | /24 | 251 | Load balancers, NAT, bastions |
| Application | /24 | 251 | Kubernetes worker nodes |
| Data | /24 | 251 | Databases, vector DBs, cache |
| GPU | /24 | 251 | GPU inference and training nodes |
| Management | /24 | 251 | Monitoring, logging, CI/CD |

### Network ACLs

Network ACLs provide stateless, subnet-level filtering:

```
# Public Subnet NACL
Inbound:
  Rule 100: ALLOW TCP 443 from 0.0.0.0/0 (HTTPS in)
  Rule 110: ALLOW TCP 80 from 0.0.0.0/0 (HTTP redirect)
  Rule 120: ALLOW TCP 1024-65535 from 10.0.0.0/16 (Ephemeral return)
  Rule *: DENY ALL

Outbound:
  Rule 100: ALLOW TCP 1024-65535 from 0.0.0.0/0 (Ephemeral out)
  Rule 110: ALLOW TCP 443 to 10.0.0.0/16 (To private subnets)
  Rule *: DENY ALL
```

### Security Groups

Security groups provide stateful, instance-level filtering:

```
# Application Node Security Group
Inbound:
  - TCP 443 from ALB Security Group
  - TCP 6443 from Management CIDR (Kubernetes API)
  - TCP 22 from Bastion Security Group (SSH, jump only)

Outbound:
  - TCP 443 to Data Subnet CIDR (databases)
  - TCP 443 to NAT Gateway (external APIs)
  - TCP 53, UDP 53 to VPC DNS
  - TCP 443 to LLM API endpoints (Azure OpenAI, AWS Bedrock)
```

---

## Connectivity Patterns

### Internet Access

Application pods do NOT have direct internet access. All egress flows through:

```
Application Pod
    |
    v
NAT Gateway (public subnet)
    |
    v
Egress Proxy (optional, for inspection and filtering)
    |
    v
Internet
```

### Egress Filtering

Approved egress destinations only:

```yaml
# Egress policy (Kubernetes NetworkPolicy + proxy allowlist)
egress_rules:
  allowed_domains:
    # LLM APIs
    - "*.openai.azure.com"
    - "*.bedrock.amazonaws.com"
    - "*.anthropic.com"

    # Cloud provider services
    - "*.blob.core.windows.net"
    - "*.s3.amazonaws.com"
    - "*.ecr.amazonaws.com"

    # Package registries (CI/CD only)
    - "pypi.org"
    - "registry.npmjs.org"
    - "hub.docker.com"

    # Monitoring
    - "*.monitoring.azure.com"
    - "*.cloudwatch.amazonaws.com"

  denied_domains:
    - "*"  # Everything else is denied by default
```

### VPC Peering

For multi-VPC architectures (production, staging, development):

```
┌─────────────┐     VPC Peering     ┌─────────────┐
│  VPC: Prod  │ ◄─────────────────► │ VPC: Staging│
│  10.0.0.0/16│                     │10.1.0.0/16  │
└──────┬──────┘                     └──────┬──────┘
       │                                   │
       │         VPC Peering               │
       └─────────────────┬─────────────────┘
                         │
                   ┌─────┴─────┐
                   │VPC: Dev   │
                   │10.2.0.0/16│
                   └───────────┘
```

**Rules:**
- Prod <-> Staging: Peered (for promotion testing)
- Prod <-> Dev: NOT peered (isolation)
- Staging <-> Dev: Peered (for development workflow)

### PrivateLink / VPC Endpoints

For cloud provider services, use private endpoints instead of internet routing:

| Service | Connection Method |
|---------|------------------|
| Azure OpenAI | Private Endpoint |
| Azure Blob Storage | Private Endpoint |
| Azure Container Registry | Private Endpoint |
| AWS S3 | VPC Endpoint (Gateway) |
| AWS ECR | VPC Endpoint (Interface) |
| AWS Bedrock | VPC Endpoint (Interface) |

---

## Network Security

### TLS Everywhere

- **Ingress**: TLS 1.2+ at the load balancer
- **Service-to-Service**: mTLS via service mesh (Istio/Linkerd)
- **Database Connections**: TLS for all database connections
- **External APIs**: TLS for all external API calls

### WAF (Web Application Firewall)

WAF rules at the edge:

```
WAF Rules:
  - SQL injection protection
  - XSS protection
  - Request size limits (max 10MB)
  - Rate limiting per IP (100 req/min)
  - Geographic blocking (if required)
  - Bot detection
  - Custom rules for prompt injection patterns
```

### DDoS Protection

| Layer | Protection |
|-------|-----------|
| **L3/L4** | Cloud provider DDoS protection (AWS Shield, Azure DDoS) |
| **L7** | WAF rate limiting, CAPTCHA for suspicious traffic |
| **Application** | Request validation, authentication, circuit breakers |

### Network Monitoring

| Tool | Purpose |
|------|---------|
| VPC Flow Logs | All network traffic logging |
| Network Watcher / VPC Reachability Analyzer | Network path analysis |
| IDS/IPS | Intrusion detection and prevention |
| Anomaly Detection | Unusual traffic pattern detection |

---

## Hybrid Connectivity

### Site-to-Site VPN

For connecting on-premises data centers to cloud VPCs:
- IPsec VPN tunnels (redundant, active-passive)
- Minimum 1 Gbps throughput
- BGP for dynamic routing
- Monitor tunnel health and failover

### Direct Connect / ExpressRoute

For higher bandwidth and lower latency:
- AWS Direct Connect or Azure ExpressRoute
- 10 Gbps minimum for production
- Redundant connections (different physical paths)
- Private peering (no internet transit)

### Use Cases

| Use Case | Connection Type | Bandwidth |
|----------|----------------|-----------|
| Cloud to on-premises data sync | Direct Connect / ExpressRoute | 10 Gbps |
| Cloud to on-premises AD authentication | Site-to-Site VPN | 100 Mbps |
| Cloud to co-location (GPU cluster) | Direct Connect / ExpressRoute | 40 Gbps |

---

## DNS Architecture

See [dns.md](dns.md) for detailed DNS management.

High-level design:
- **External DNS**: Route 53 / Azure DNS for public domains
- **Internal DNS**: VPC DNS for service discovery
- **Split-Horizon**: Different records for internal vs. external resolution
- **Private Hosted Zones**: Internal service names

---

## Network Capacity Planning

### Bandwidth Requirements

| Traffic Pattern | Estimated Bandwidth | Notes |
|----------------|--------------------|------|
| Customer -> API Gateway | 10 Gbps peak Based on 10K concurrent users |
| API Gateway -> Application | 10 Gbps | Internal traffic |
| Application -> LLM API | 1 Gbps | External API calls |
| Application -> Vector DB | 5 Gbps | Embedding queries |
| Application -> Database | 2 Gbps | Standard DB queries |
| GPU Node Interconnect | 100 Gbps | Distributed inference/training |
| Data Replication (cross-region) | 1 Gbps | DR replication |

### Scaling Considerations

- NAT Gateway: 5 Gbps per NAT GW. Deploy multiple for higher throughput.
- Load Balancer: Auto-scales, but monitor connection limits.
- VPC Endpoint: Monitor throughput and scale if needed.

---

## Network Design Checklist

### VPC Design
- [ ] CIDR blocks planned (no overlap between VPCs)
- [ ] Subnets sized appropriately
- [ ] Subnets spread across 3+ AZs
- [ ] Public and private subnets separated
- [ ] Network ACLs configured
- [ ] Security groups defined per workload type

### Connectivity
- [ ] NAT Gateways deployed (one per AZ)
- [ ] VPC endpoints for cloud services
- [ ] VPC peering configured (where needed)
- [ ] Site-to-site VPN / Direct Connect (if hybrid)
- [ ] Egress filtering and proxy configured

### Security
- [ ] WAF configured and rules tested
- [ ] DDoS protection enabled
- [ ] TLS certificates managed (ACM, Key Vault)
- [ ] mTLS between services (service mesh)
- [ ] Network policies enforced (Kubernetes)
- [ ] VPC Flow Logs enabled

### Monitoring
- [ ] Network metrics dashboards
- [ ] Anomaly detection configured
- [ ] Alert thresholds set
- [ ] IDS/IPS deployed
- [ ] Regular network path testing

---

## Cross-References

- [README.md](README.md) -- Infrastructure overview
- [cloud-providers.md](cloud-providers.md) -- Cloud provider selection
- [dns.md](dns.md) -- DNS management
- [load-balancing.md](load-balancing.md) -- Load balancer configuration
- [cdn.md](cdn.md) -- CDN architecture
- [edge-computing.md](edge-computing.md) -- Edge computing
- [../security/README.md](../security/README.md) -- Security practices
