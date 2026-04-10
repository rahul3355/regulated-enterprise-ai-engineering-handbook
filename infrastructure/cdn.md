# CDN Architecture

## Overview

This document covers CDN design, caching strategies, security controls, and optimization techniques for the banking GenAI platform.

---

## CDN Purpose

A Content Delivery Network (CDN) serves content from edge locations closer to users, reducing latency and origin server load. For the GenAI platform, the CDN serves:

1. **Static Assets**: JavaScript bundles, CSS, images, fonts (React SPA)
2. **Cached API Responses**: Non-personalized, cacheable responses
3. **Document Downloads**: Customer statements, forms, guides
4. **Model Artifacts**: Public model weights for edge inference (if applicable)

---

## CDN Architecture

```
User Browser
    │
    ▼
┌─────────────────────────┐
│      Edge Location       │  (CloudFront, Azure CDN)
│  - Cache HIT: Serve     │
│  - Cache MISS: Forward  │
└────────────┬────────────┘
             │
             ▼
┌─────────────────────────┐
│      Origin Shield       │  (Mid-tier cache)
│  - Aggregate MISSes     │
│  - Single origin fetch  │
└────────────┬────────────┘
             │
             ▼
┌─────────────────────────┐
│      Origin Server       │  (S3, ALB, API Gateway)
│  - S3: Static assets    │
│  - ALB: API responses   │
└─────────────────────────┘
```

---

## CDN Configuration

### AWS CloudFront Configuration

```yaml
# CloudFront Distribution (Terraform)
resource "aws_cloudfront_distribution" "genai_cdn" {
  # Static assets origin (S3)
  origin {
    domain_name = aws_s3_bucket.genai_frontend.bucket_regional_domain_name
    origin_id   = "s3-frontend"

    s3_origin_config {
      origin_access_identity = aws_cloudfront_origin_access_identity.genai.cloudfront_access_identity_path
    }
  }

  # API origin (ALB)
  origin {
    domain_name = aws_lb.genai_alb.dns_name
    origin_id   = "alb-api"

    custom_origin_config {
      http_port              = 80
      https_port             = 443
      origin_protocol_policy = "https-only"
      origin_ssl_protocols   = ["TLSv1.2"]
    }
  }

  # Default cache behavior (SPA)
  default_cache_behavior {
    allowed_methods  = ["GET", "HEAD", "OPTIONS"]
    cached_methods   = ["GET", "HEAD"]
    target_origin_id = "s3-frontend"

    forwarded_values {
      query_string = false
      cookies {
        forward = "none"
      }
    }

    viewer_protocol_policy = "redirect-to-https"
    compress               = true

    # Cache for 1 year (immutable assets with content hash)
    min_ttl     = 31536000  # 1 year
    default_ttl = 31536000
    max_ttl     = 31536000
  }

  # API cache behavior
  ordered_cache_behavior {
    path_pattern     = "/api/*"
    allowed_methods  = ["GET", "HEAD", "OPTIONS", "POST", "PUT", "DELETE", "PATCH"]
    cached_methods   = ["GET", "HEAD"]
    target_origin_id = "alb-api"

    forwarded_values {
      query_string = true
      headers      = ["Authorization", "Content-Type"]
      cookies {
        forward = "all"
      }
    }

    viewer_protocol_policy = "https-only"
    compress               = true

    # Do NOT cache API responses by default
    min_ttl     = 0
    default_ttl = 0
    max_ttl     = 0
  }

  # Price class (only use edge locations in UK/EU for GDPR)
  price_class = "PriceClass_100"  # US, Canada, Europe

  # Security
  viewer_certificate {
    acm_certificate_arn      = aws_acm_certificate.genai.arn
    ssl_support_method       = "sni-only"
    minimum_protocol_version = "TLSv1.2_2021"
  }

  # WAF integration
  web_acl_id = aws_wafv2_web_acl.genai_cdn.arn
}
```

---

## Caching Strategy

### Cache-Control Headers

| Asset Type | Cache-Control | Max-Age | Revalidation | Rationale |
|-----------|--------------|---------|-------------|-----------|
| **JS bundles (hashed)** | `public, immutable` | 1 year | None | Content hash ensures unique URL per version |
| **CSS (hashed)** | `public, immutable` | 1 year | None | Content hash ensures unique URL per version |
| **Images (hashed)** | `public, immutable` | 1 year | None | Content hash ensures unique URL per version |
| **Fonts** | `public, immutable` | 1 year | None | Fonts rarely change |
| **HTML (index.html)** | `no-cache, no-store` | 0 | Must revalidate | Entry point, changes with each deploy |
| **API responses (personalized)** | `no-cache, private` | 0 | Must revalidate | User-specific data |
| **API responses (public)** | `public, max-age=300` | 5 minutes | Stale-while-revalidate | Non-personalized data |
| **Documents (PDFs)** | `public, max-age=86400` | 24 hours | Stale-while-revalidate | Static documents |

### Cache Invalidation

```bash
# CloudFront invalidation (after deployment)
aws cloudfront create-invalidation \
  --distribution-id E1234567890 \
  --paths "/index.html" "/manifest.json"

# Note: Only invalidate entry points, not hashed assets
# Hashed assets have new URLs, so old versions are never requested
```

### Cache Key Design

For API responses cached at the CDN:

```
Cache Key Components:
  - URL path
  - Query parameters (only whitelisted ones)
  - Authorization header (for authenticated cache)
  - Accept-Encoding header (for compression variants)

Do NOT include:
  - Cookie values (unless used for personalization)
  - X-Forwarded-For (use geo-based variation instead)
  - User-Agent (unless serving different content per device)
```

---

## CDN Security

### WAF Rules at the Edge

```yaml
# WAF rules for CDN
waf_rules:
  # SQL injection protection
  - name: SQLi_protection
    action: BLOCK
    statement: sqli_match_rule

  # XSS protection
  - name: XSS_protection
    action: BLOCK
    statement: xss_match_rule

  # Rate limiting
  - name: Rate_limiting
    action: BLOCK
    statement:
      rate_based_statement:
        limit: 1000  # requests per 5 minutes per IP
        aggregate_key_type: IP

  # Geographic restriction (if required)
  - name: Geo_restriction
    action: BLOCK
    statement:
      geo_match_statement:
        country_codes: [XX, YY]  # Restricted countries

  # Bot protection
  - name: Bot_protection
    action: CHALLENGE  # CAPTCHA for suspected bots
    statement:
      managed_rule_group: aws_managed_bot_control
```

### TLS at the Edge

- Minimum TLS 1.2
- Modern cipher suites only
- HSTS header: `Strict-Transport-Security: max-age=31536000; includeSubDomains; preload`
- Certificate managed by ACM (auto-renewal)

### Origin Access Identity

For S3 origins, use CloudFront Origin Access Identity (OAI) to prevent direct S3 access:

```
Internet ──► CloudFront ──► S3
                ✓              ✗
          (OAI allowed)   (Direct access denied)
```

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowCloudFrontAccess",
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam::cloudfront:user/CloudFront Origin Access Identity E1234567890"
      },
      "Action": "s3:GetObject",
      "Resource": "arn:aws:s3:::genai-frontend/*"
    }
  ]
}
```

---

## CDN Performance Optimization

### Compression

| Content Type | Compression | Typical Reduction |
|-------------|-------------|------------------|
| JavaScript | Brotli (preferred) / Gzip | 60-70% |
| CSS | Brotli (preferred) / Gzip | 70-80% |
| HTML | Brotli (preferred) / Gzip | 70-80% |
| JSON | Brotli (preferred) / Gzip | 60-70% |
| Images | Already compressed | N/A |
| SVG | Gzip | 40-60% |

### Image Optimization

| Technique | Tool | Reduction |
|-----------|------|-----------|
| **WebP conversion** | CloudFront Functions | 25-35% vs. JPEG |
| **AVIF conversion** | Lambda@Edge | 50% vs. JPEG |
| **Responsive images** | srcset attribute | 50-80% (mobile vs. desktop) |
| **Lazy loading** | HTML `loading="lazy"` | Defers off-screen images |

### HTTP/2 and HTTP/3

- Enable HTTP/2 for multiplexing (multiple requests over single connection)
- Enable HTTP/3 (QUIC) for improved performance on lossy networks
- Server Push for critical CSS and fonts (HTTP/2)

---

## CDN Monitoring

### Key Metrics

| Metric | Alert Threshold | Description |
|--------|---------------|-------------|
| **Cache hit rate** | < 80% | Percentage of requests served from cache |
| **Error rate (4xx/5xx)** | > 1% | Error responses from CDN |
| **Origin response time** | > 500ms | Time to fetch from origin |
| **Bandwidth** | > 80% of provisioned | CDN bandwidth usage |
| **Request rate** | > 80% of limit | CDN request rate |

### Dashboard

```
CDN Dashboard
=============

Cache Performance:
  Hit Rate: 94.2% (target: > 90%) ✓
  Miss Rate: 5.8%

Traffic:
  Requests/sec: 12,500
  Bandwidth: 850 Mbps
  Unique viewers (24h): 45,000

Origin Load:
  Origin requests/sec: 720
  Origin response time (P95): 120ms
  Origin error rate: 0.02%

Errors:
  4xx: 0.5% (mostly 404 for old asset versions)
  5xx: 0.01%

Top Requests:
  1. /index.html (15%)
  2. /static/js/main.<hash>.js (12%)
  3. /static/css/main.<hash>.css (8%)
  4. /api/v1/chat (5%)
  5. /assets/logo.svg (4%)
```

---

## CDN Cost Optimization

| Strategy | Savings | Notes |
|----------|---------|-------|
| **High cache hit rate** | 60-80% origin cost reduction | Cache everything cacheable |
| **Compression** | 30-50% bandwidth cost reduction | Brotli > Gzip |
| **Image optimization** | 25-50% image bandwidth | WebP/AVIF |
| **Price class selection** | 30-50% CDN cost | Use only required edge locations |
| **Custom domain vs. CDN domain** | Varies | Custom domain has additional cost |

---

## CDN for GenAI-Specific Content

### Streaming AI Responses

For server-sent events (SSE) or WebSocket streaming from the AI:

```
CDN does NOT cache streaming responses.
Configuration:
  - Path: /v1/chat/stream
  - Cache: no-cache, no-store
  - Protocol: WebSocket or SSE
  - Connection: Long-lived (direct to origin)
```

### Cached AI Responses

For non-streaming, non-personalized AI responses:

```
Cache only when:
  - Response is the same for all users (e.g., general FAQ answers)
  - Response has a defined TTL (e.g., market commentary updated hourly)
  - No PII or personalized data in the response

Never cache:
  - Personalized financial advice
  - Account-specific information
  - Responses containing PII
```

---

## Cross-References

- [README.md](README.md) -- Infrastructure overview
- [networking.md](networking.md) -- Network architecture
- [dns.md](dns.md) -- DNS management
- [edge-computing.md](edge-computing.md) -- Edge computing
- [load-balancing.md](load-balancing.md) -- Load balancer configuration
