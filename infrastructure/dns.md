# DNS Management

## Overview

This document covers DNS architecture, split-horizon DNS, failover strategies, and DNS management practices for the banking GenAI platform.

---

## DNS Architecture

### DNS Hierarchy

```
                         ┌─────────────┐
                         │  . (root)   │
                         └──────┬──────┘
                                │
                         ┌──────┴──────┐
                         │   .com /    │
                         │   .co.uk    │
                         └──────┬──────┘
                                │
                    ┌───────────┴───────────┐
                    │    bank.com           │
                    │    (Registered Domain) │
                    └───────────┬───────────┘
                                │
              ┌────────────┬────┴────┬────────────┐
              │            │         │            │
        ┌─────┴─────┐ ┌───┴──┐ ┌───┴────┐ ┌────┴────┐
        │  api.     │ │ www. │ │ genai. │ │internal.│
        │  bank.com │ │bank  │ │bank.com│ │bank.com │
        └─────┬─────┘ └───┬──┘ └───┬────┘ └────┬────┘
              │            │       │            │
        ┌─────┴─────┐      │  ┌────┴─────┐     │
        │ assistbot │      │  │assistbot │     │
        │ .api      │      │  │.internal │     │
        └───────────┘      │  └──────────┘     │
                      ┌────┴────┐         ┌────┴────┐
                      │ wealth  │         │ postgres│
                      │.internal│         │.internal│
                      └─────────┘         └─────────┘
```

### DNS Record Types

| Record Type | Purpose | Example |
|-------------|---------|---------|
| **A** | Maps name to IPv4 address | `api.bank.com → 203.0.113.10` |
| **AAAA** | Maps name to IPv6 address | `api.bank.com → 2001:db8::1` |
| **CNAME** | Alias to another name | `www.bank.com → bank.com` |
| **MX** | Mail exchange | `bank.com → mail.bank.com` |
| **TXT** | Arbitrary text (SPF, DKIM, verification) | `v=spf1 include:...` |
| **SRV** | Service location | `_https._tcp.genai.bank.com` |
| **NS** | Name server delegation | `genai.bank.com → ns1.awsdns.com` |
| **SOA** | Start of authority | Zone metadata |

---

## Split-Horizon DNS

Split-horizon DNS provides different responses based on the source of the query (internal vs. external).

### Architecture

```
┌──────────────────────────────────────────────────┐
│              External DNS (Public)                │
│  Provider: Route 53 / Azure DNS                  │
│                                                    │
│  Records visible to: Internet                     │
│  ┌─────────────────────────────────────────┐     │
│  │ api.genai.bank.com → ALB public IP      │     │
│  │ www.genai.bank.com → CDN endpoint       │     │
│  │ status.genai.bank.com → Statuspage      │     │
│  └─────────────────────────────────────────┘     │
└──────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────┐
│              Internal DNS (Private)               │
│  Provider: VPC DNS (Route 53 Private Zone)       │
│                                                    │
│  Records visible to: VPC-internal only            │
│  ┌─────────────────────────────────────────┐     │
│  │ api.genai.bank.com → Internal NLB IP    │     │
│  │ assistbot.internal → K8s service IP      │     │
│  │ postgres.internal → RDS endpoint         │     │
│  │ vector-db.internal → Milvus endpoint     │     │
│  │ grafana.internal → Monitoring endpoint   │     │
│  └─────────────────────────────────────────┘     │
└──────────────────────────────────────────────────┘
```

### Split-Horizon Configuration

```hcl
# Terraform: Route 53 split-horizon DNS

# Public hosted zone
resource "aws_route53_zone" "public" {
  name = "genai.bank.com"
  comment = "Public DNS zone"
}

resource "aws_route53_record" "api_public" {
  zone_id = aws_route53_zone.public.zone_id
  name    = "api.genai.bank.com"
  type    = "A"

  alias {
    name                   = aws_lb.genai_alb.dns_name
    zone_id                = aws_lb.genai_alb.zone_id
    evaluate_target_health = true
  }
}

# Private hosted zone (VPC-internal)
resource "aws_route53_zone" "private" {
  name = "genai.bank.internal"
  comment = "Private DNS zone"
  vpc {
    vpc_id = aws_vpc.genai_prod.id
  }
}

resource "aws_route53_record" "postgres_private" {
  zone_id = aws_route53_zone.private.zone_id
  name    = "postgres.genai.bank.internal"
  type    = "CNAME"
  ttl     = 60
  records = [aws_db_instance.postgres.address]
}

resource "aws_route53_record" "vector_db_private" {
  zone_id = aws_route53_zone.private.zone_id
  name    = "vector-db.genai.bank.internal"
  type    = "CNAME"
  ttl     = 60
  records = ["milvus-prod.genai.svc.cluster.local"]
}
```

### Benefits of Split-Horizon DNS

1. **Security**: Internal service names are not exposed to the internet
2. **Performance**: Internal queries resolve to internal endpoints (lower latency, no egress cost)
3. **Flexibility**: Different endpoints for internal vs. external access
4. **Resilience**: Internal services remain resolvable even if public DNS fails

---

## DNS Failover

### Active-Passive Failover

```
Primary: api.genai.bank.com → ALB (London region)
    │
    │ Health check fails
    ▼
Failover: api.genai.bank.com → ALB (Frankfurt region)
```

### Configuration (Route 53)

```hcl
# Primary record (London)
resource "aws_route53_record" "api_primary" {
  zone_id = aws_route53_zone.public.zone_id
  name    = "api.genai.bank.com"
  type    = "A"

  failover_routing_policy {
    type = "PRIMARY"
  }

  alias {
    name                   = aws_lb.london_alb.dns_name
    zone_id                = aws_lb.london_alb.zone_id
    evaluate_target_health = true
  }

  health_check_id = aws_route53_health_check.london.id
}

# Secondary record (Frankfurt)
resource "aws_route53_record" "api_secondary" {
  zone_id = aws_route53_zone.public.zone_id
  name    = "api.genai.bank.com"
  type    = "A"

  failover_routing_policy {
    type = "SECONDARY"
  }

  alias {
    name                   = aws_lb.frankfurt_alb.dns_name
    zone_id                = aws_lb.frankfurt_alb.zone_id
    evaluate_target_health = true
  }
}

# Health check
resource "aws_route53_health_check" "london" {
  fqdn              = "api.genai.bank.com"
  port              = 443
  type              = "HTTPS"
  resource_path     = "/health"
  failure_threshold = 3
  request_interval  = 10
}
```

### Active-Active Failover (Latency-Based Routing)

```hcl
# Route users to the closest region
resource "aws_route53_record" "api_eu_west" {
  zone_id = aws_route53_zone.public.zone_id
  name    = "api.genai.bank.com"
  type    = "A"

  latency_routing_policy {
    region = "eu-west-2"  # London
  }

  set_identifier = "london"

  alias {
    name                   = aws_lb.london_alb.dns_name
    zone_id                = aws_lb.london_alb.zone_id
    evaluate_target_health = true
  }
}

resource "aws_route53_record" "api_eu_central" {
  zone_id = aws_route53_zone.public.zone_id
  name    = "api.genai.bank.com"
  type    = "A"

  latency_routing_policy {
    region = "eu-central-1"  # Frankfurt
  }

  set_identifier = "frankfurt"

  alias {
    name                   = aws_lb.frankfurt_alb.dns_name
    zone_id                = aws_lb.frankfurt_alb.zone_id
    evaluate_target_health = true
  }
}
```

### Weighted Routing (Canary Deployments)

```hcl
# 90% traffic to v2, 10% to v1
resource "aws_route53_record" "api_v2" {
  zone_id = aws_route53_zone.public.zone_id
  name    = "api.genai.bank.com"
  type    = "A"

  weighted_routing_policy {
    weight = 90
  }

  set_identifier = "v2"

  alias {
    name                   = aws_lb.alb_v2.dns_name
    zone_id                = aws_lb.alb_v2.zone_id
    evaluate_target_health = true
  }
}

resource "aws_route53_record" "api_v1" {
  zone_id = aws_route53_zone.public.zone_id
  name    = "api.genai.bank.com"
  type    = "A"

  weighted_routing_policy {
    weight = 10
  }

  set_identifier = "v1"

  alias {
    name                   = aws_lb.alb_v1.dns_name
    zone_id                = aws_lb.alb_v1.zone_id
    evaluate_target_health = true
  }
}
```

---

## DNS Security

### DNSSEC

- Enable DNSSEC signing for public hosted zones
- Prevents DNS spoofing and cache poisoning
- Required for high-security banking domains

### DNS-over-HTTPS (DoH) / DNS-over-TLS (DoT)

- Encrypt DNS queries to prevent eavesdropping
- Use DoH/DoT for internal service resolution where supported
- VPC DNS resolver automatically encrypts queries

### DNS Monitoring and Alerting

| Metric | Alert Threshold | Description |
|--------|---------------|-------------|
| **DNS query volume** | > 2x baseline | Unusual DNS activity |
| **NXDOMAIN rate** | > 5% of queries | Possible DNS tunneling or misconfiguration |
| **Health check failures** | Any failure | Target service unreachable |
| **DNS resolution latency** | > 100ms | DNS resolver performance issue |
| **Certificate expiry** | < 30 days | TLS certificate approaching expiry |

---

## DNS Management Best Practices

### TTL Strategy

| Record Type | TTL | Rationale |
|-------------|-----|-----------|
| **Stable records** (A, AAAA to fixed IPs) | 3600s (1 hour) | Reduce DNS query load |
| **Alias records** (to ALB, CloudFront) | 60s (managed by provider) | Provider-managed |
| **Failover records** | 60s | Quick failover response |
| **Canary/weighted records** | 60s | Quick traffic shift |
| **Internal service records** | 60s | Internal changes propagate quickly |

### DNS Change Management

1. **Test in staging**: All DNS changes tested in staging environment first
2. **Lower TTL before changes**: Reduce TTL to 60s at least 24 hours before planned changes
3. **Monitor after changes**: Verify resolution after DNS changes are deployed
4. **Document changes**: All DNS changes logged with reason and approval

### DNS Naming Convention

```
<service>.<environment>.<domain>

Examples:
  assistbot.prod.genai.bank.internal     # Production AssistBot service
  wealth-advisor.staging.genai.bank.internal  # Staging Wealth Advisor
  api.genai.bank.com                     # Public API endpoint
  status.genai.bank.com                  # Public status page
  grafana.internal.genai.bank.internal   # Internal Grafana
```

---

## DNS Troubleshooting

### Common Issues

| Issue | Symptom | Resolution |
|-------|---------|------------|
| **Stale DNS cache** | Service unreachable, nslookup shows old IP | Wait for TTL expiry, flush cache |
| **Split-horizon misconfiguration** | Internal service resolves to public IP | Check VPC DNS association |
| **Health check failure** | DNS failover not triggering | Check health check endpoint |
| **DNS resolution timeout** | All services unreachable | Check DNS resolver health |
| **Certificate mismatch** | TLS errors after DNS change | Verify certificate covers new name |

### Troubleshooting Commands

```bash
# Check DNS resolution
dig api.genai.bank.com
nslookup api.genai.bank.com

# Check specific DNS server
dig @8.8.8.8 api.genai.bank.com

# Check DNSSEC
dig +dnssec api.genai.bank.com

# Trace DNS resolution path
dig +trace api.genai.bank.com

# Check split-horizon (from inside VPC)
dig postgres.genai.bank.internal

# Monitor DNS resolution time
time dig api.genai.bank.com
```

---

## Cross-References

- [README.md](README.md) -- Infrastructure overview
- [networking.md](networking.md) -- Network architecture
- [load-balancing.md](load-balancing.md) -- Load balancer configuration
- [cdn.md](cdn.md) -- CDN architecture
