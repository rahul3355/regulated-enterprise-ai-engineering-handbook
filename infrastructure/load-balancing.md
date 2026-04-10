# Load Balancing

## Overview

This document covers load balancer types, configuration patterns, health check design, and best practices for the banking GenAI platform. Load balancing is critical for availability, scalability, and resilience.

---

## Load Balancer Types

### Layer 4 (Transport Layer) Load Balancer

**Protocols**: TCP, UDP
**Examples**: AWS NLB, Azure Standard Load Balancer, GCP Internal TCP/UDP LB

**When to Use:**
- High-performance, low-latency requirements
- Non-HTTP protocols (gRPC, custom protocols)
- TLS termination handled by backend
- GPU inference services (Triton gRPC)

**Characteristics:**
- No content-based routing
- Faster processing (no HTTP parsing)
- Source IP preserved
- Supports millions of requests per second

### Layer 7 (Application Layer) Load Balancer

**Protocols**: HTTP/HTTPS, HTTP/2, WebSocket
**Examples**: AWS ALB, Azure Application Gateway, GCP HTTP(S) Load Balancing

**When to Use:**
- HTTP/HTTPS traffic
- Content-based routing (path, host, headers)
- SSL/TLS termination at the edge
- WAF integration required
- Cookie-based session affinity

**Characteristics:**
- Content-aware routing
- SSL/TLS termination
- Request inspection and modification
- Slightly higher latency than L4

---

## Load Balancer Architecture

### Production Load Balancing Hierarchy

```
Internet
    │
    ▼
┌─────────────────────────┐
│   WAF / DDoS Protection │  (AWS WAF, Azure WAF)
└────────────┬────────────┘
             │
             ▼
┌─────────────────────────┐
│   CDN (CloudFront,      │  Static assets, cached responses
│   Azure CDN)            │
└────────────┬────────────┘
             │
             ▼
┌─────────────────────────┐
│   L7 Load Balancer      │  (ALB, Application Gateway)
│   - Path-based routing  │
│   - SSL termination     │
│   - Host-based routing  │
└──────┬──────────┬───────┘
       │          │
       ▼          ▼
┌──────────┐ ┌──────────┐
│ AssistBot │ │WealthAd. │   Backend service pools
│  Pods     │ │  Pods    │   (Kubernetes services)
└─────┬────┘ └─────┬────┘
      │            │
      ▼            ▼
┌─────────────────────────┐
│   L4 Load Balancer      │  (NLB, Standard LB)
│   - gRPC to Triton      │
│   - TCP to databases     │
└──────────┬──────────────┘
           │
           ▼
┌─────────────────────────┐
│   GPU Inference Pods    │  (Triton, vLLM)
└─────────────────────────┘
```

---

## Load Balancer Configuration

### AWS ALB Configuration

```yaml
# Ingress configuration for Kubernetes (AWS ALB Controller)
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: genai-ingress
  annotations:
    alb.ingress.kubernetes.io/scheme: internet-facing
    alb.ingress.kubernetes.io/target-type: ip
    alb.ingress.kubernetes.io/ssl-redirect: "443"
    alb.ingress.kubernetes.io/certificate-arn: arn:aws:acm:...
    alb.ingress.kubernetes.io/healthcheck-path: /health
    alb.ingress.kubernetes.io/healthcheck-interval-seconds: "10"
    alb.ingress.kubernetes.io/healthcheck-timeout-seconds: "5"
    alb.ingress.kubernetes.io/healthy-threshold-count: "2"
    alb.ingress.kubernetes.io/unhealthy-threshold-count: "3"
    alb.ingress.kubernetes.io/load-balancer-attributes: |
      idle_timeout.timeout_seconds=60
      routing.http2.enabled=true
      deletion_protection.enabled=true
spec:
  ingressClassName: alb
  tls:
    - hosts:
        - api.genai.bank.com
      secretName: genai-tls-secret
  rules:
    - host: api.genai.bank.com
      http:
        paths:
          - path: /v1/chat
            pathType: Prefix
            backend:
              service:
                name: assistbot-service
                port:
                  number: 80
          - path: /v1/advisory
            pathType: Prefix
            backend:
              service:
                name: wealth-advisor-service
                port:
                  number: 80
          - path: /v1/compliance
            pathType: Prefix
            backend:
              service:
                name: compliance-bot-service
                port:
                  number: 80
```

### Azure Application Gateway Configuration

```json
{
  "applicationGateway": {
    "sku": {
      "name": "WAF_v2",
      "tier": "WAF_v2"
    },
    "sslPolicy": {
      "policyType": "Predefined",
      "policyName": "AppGwSslPolicy20220101"
    },
    "frontendPorts": [
      {"name": "port-443", "port": 443},
      {"name": "port-80", "port": 80}
    ],
    "backendHttpSettingsCollection": [
      {
        "name": "genai-backend",
        "port": 80,
        "protocol": "Http",
        "requestTimeout": 60,
        "probe": {
          "id": "[resourceId('Microsoft.Network/applicationGateways/probes', 'genai-health-probe')]"
        }
      }
    ],
    "httpListeners": [
      {
        "name": "genai-https-listener",
        "frontendIPConfiguration": "frontend-ip",
        "frontendPort": "port-443",
        "protocol": "Https",
        "sslCertificate": "genai-tls-cert"
      }
    ]
  }
}
```

---

## Health Check Design

### Health Check Types

| Type | Endpoint | Purpose | Check Interval |
|------|---------|---------|---------------|
| **Liveness Probe** | `/healthz` | Is the process alive? | 10 seconds |
| **Readiness Probe** | `/ready` | Is the service ready for traffic? | 5 seconds |
| **Deep Health Check** | `/health/deep` | Are all dependencies healthy? | 30 seconds |

### Kubernetes Health Probes

```yaml
livenessProbe:
  httpGet:
    path: /healthz
    port: 8080
  initialDelaySeconds: 15
  periodSeconds: 10
  timeoutSeconds: 3
  failureThreshold: 3

readinessProbe:
  httpGet:
    path: /ready
    port: 8080
  initialDelaySeconds: 5
  periodSeconds: 5
  timeoutSeconds: 3
  failureThreshold: 3

# Deep health check (not used by K8s, used by load balancer)
# /health/deep checks:
#   - Database connectivity
#   - Vector DB connectivity
#   - LLM API connectivity
#   - Cache connectivity
#   - GPU availability (for inference pods)
```

### Load Balancer Health Check Configuration

```yaml
# ALB Health Check
healthCheck:
  path: /health
  port: 8080
  protocol: HTTP
  intervalSeconds: 10
  timeoutSeconds: 5
  healthyThresholdCount: 2
  unhealthyThresholdCount: 3

# What makes a target "unhealthy":
# 1. HTTP response code != 200
# 2. Response timeout
# 3. Connection refused
# 4. 3 consecutive failures
```

### Health Check Best Practices

1. **Separate liveness and readiness**: Liveness checks if the process is alive. Readiness checks if it can serve traffic.

2. **Lightweight checks**: Health checks should be fast and resource-light. Do not run expensive queries in health checks.

3. **Dependency-aware readiness**: Readiness should check critical dependencies (database, vector DB), but not all dependencies (a non-critical cache being down should not make the service unhealthy).

4. **Avoid cascading failures**: If the health check depends on a shared resource that is down, all pods become unhealthy simultaneously, causing the load balancer to remove all targets.

5. **Startup probes**: Use startup probes for slow-starting services (model loading can take 30-60 seconds).

```yaml
startupProbe:
  httpGet:
    path: /healthz
    port: 8080
  initialDelaySeconds: 0
  periodSeconds: 5
  timeoutSeconds: 3
  failureThreshold: 30  # Allow up to 150 seconds for startup
```

---

## Load Balancing Algorithms

### Algorithm Selection

| Algorithm | When to Use | Notes |
|-----------|-------------|-------|
| **Round Robin** | Equal-capacity backends, stateless services | Default for most LBs |
| **Weighted Round Robin** | Different-capacity backends | Assign weights based on capacity |
| **Least Connections** | Long-lived connections (WebSocket, gRPC) | Routes to the least busy backend |
| **IP Hash** | Session affinity requirements | Same client IP always goes to same backend |
| **Random** | High-traffic, homogeneous backends | Surprisingly effective at scale |

### GenAI-Specific Load Balancing

For GPU inference services, consider:

| Strategy | Description | When to Use |
|----------|-------------|-------------|
| **Queue-based** | Route to the pod with the shortest queue | High-throughput inference |
| **Load-aware** | Route based on current GPU utilization | Heterogeneous GPU instances |
| **Model-aware** | Route based on which models are loaded on each pod | Multi-model serving |
| **Affinity-based** | Route same customer to same pod (for caching) | Customer-specific context caching |

---

## Session Affinity

### When Session Affinity Is Needed

- Customer context cached on specific pods
- WebSocket connections requiring stateful sessions
- gRPC streaming connections

### When to Avoid Session Affinity

- Stateless services
- Services where any pod can serve any request
- When uneven load distribution is a concern

### Configuration

```yaml
# Kubernetes session affinity (cookie-based)
apiVersion: v1
kind: Service
metadata:
  name: assistbot-service
spec:
  sessionAffinity: ClientIP
  sessionAffinityConfig:
    clientIP:
      timeoutSeconds: 3600  # 1 hour
```

---

## TLS Configuration

### TLS Termination Options

| Option | Where TLS Ends | Backend Protocol | Security |
|--------|---------------|-----------------|----------|
| **TLS at LB** | Load balancer | HTTP to backend | Good (LB is in private network) |
| **End-to-End TLS** | Backend service | HTTPS to backend | Best (defense in depth) |
| **TLS with mTLS** | Backend service + service mesh | mTLS between all services | Best (zero trust) |

### TLS Certificate Management

```yaml
# cert-manager configuration for automatic certificate renewal
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: genai-tls-cert
spec:
  secretName: genai-tls-secret
  issuerRef:
    name: letsencrypt-prod
    kind: ClusterIssuer
  dnsNames:
    - api.genai.bank.com
    - assistbot.genai.bank.com
  renewBefore: 720h  # 30 days before expiry
```

### TLS Security Requirements

- Minimum TLS 1.2
- Strong cipher suites only
- HSTS headers enabled
- Certificate expiry monitoring and alerting
- Automated certificate renewal (cert-manager, ACM)

---

## Load Balancer Monitoring

### Key Metrics

| Metric | Alert Threshold | Description |
|--------|---------------|-------------|
| **Healthy host count** | < 80% of expected | Number of healthy backends |
| **Request count** | > 80% of capacity | Total requests per second |
| **Latency (P95)** | > 500ms | Backend response latency |
| **Error rate (5xx)** | > 1% | Backend error responses |
| **Active connections** | > 80% of limit | Concurrent connections |
| **Unhealthy host count** | > 0 for > 5 minutes | Backends failing health checks |

### Dashboard Configuration

```yaml
# Grafana dashboard panels for load balancer
panels:
  - title: "Healthy Hosts"
    query: "aws_alb_healthy_host_count{load_balancer='genai-prod'}"
    alert:
      condition: "< 3"
      severity: "warning"

  - title: "Request Rate"
    query: "rate(aws_alb_request_count_sum{load_balancer='genai-prod'}[5m])"

  - title: "Backend Latency (P95)"
    query: "histogram_quantile(0.95, aws_alb_target_response_time_sum)"
    alert:
      condition: "> 0.5"
      severity: "warning"

  - title: "5xx Error Rate"
    query: "sum(rate(aws_alb_http_code_5xx[5m])) / sum(rate(aws_alb_request_count[5m]))"
    alert:
      condition: "> 0.01"
      severity: "critical"
```

---

## Cross-References

- [README.md](README.md) -- Infrastructure overview
- [networking.md](networking.md) -- Network architecture
- [dns.md](dns.md) -- DNS management
- [cdn.md](cdn.md) -- CDN architecture
- [compute.md](compute.md) -- Compute options
