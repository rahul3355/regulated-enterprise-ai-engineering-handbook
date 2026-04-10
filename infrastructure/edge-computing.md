# Edge Computing

## Overview

This document covers edge computing architectures for low-latency AI inference, content personalization, and request preprocessing at CDN edge locations.

---

## Edge Computing Use Cases

### 1. Low-Latency AI Inference

Running lightweight ML models at the edge for ultra-low-latency predictions:

| Use Case | Model Type | Latency Target | Edge Platform |
|----------|-----------|---------------|---------------|
| Intent classification (pre-routing) | Tiny BERT, < 50MB | < 10ms | Cloudflare Workers AI |
| Sentiment analysis | DistilBERT, < 100MB | < 20ms | Lambda@Edge |
| Spam/abuse detection | Logistic Regression, < 10MB | < 5ms | Cloudflare Workers |
| Language detection | FastText, < 5MB | < 3ms | Edge Functions |

### 2. Request Preprocessing

Processing requests at the edge before forwarding to origin:

- **Request validation**: Schema validation, input sanitization
- **Authentication**: JWT validation at the edge
- **Rate limiting**: Per-user rate limiting at edge
- **Bot detection**: Identify and challenge bots before reaching origin
- **Prompt injection detection**: Pattern matching on user input before reaching LLM

### 3. Response Postprocessing

Modifying responses at the edge before delivering to users:

- **PII redaction**: Scan and redact PII from AI responses before delivery
- **Response transformation**: Format conversion, compression
- **A/B testing**: Serve different response variants to different user segments
- **Geo-personalization**: Add location-specific content to responses

---

## Edge Computing Platforms

### AWS Lambda@Edge

```javascript
// Lambda@Edge: Prompt injection detection at the edge
exports.handler = async (event) => {
  const request = event.Records[0].cf.request;

  if (request.uri.startsWith('/v1/chat')) {
    const body = JSON.parse(request.body);

    // Check for prompt injection patterns
    const injectionPatterns = [
      /ignore\s+all\s+previous/i,
      /system\s*:\s*override/i,
      /developer\s*mode/i,
      /BEGIN\s+EXECUTION/i,
    ];

    for (const pattern of injectionPatterns) {
      if (pattern.test(body.message)) {
        return {
          status: '403',
          statusDescription: 'Forbidden',
          body: JSON.stringify({
            error: 'Request blocked: suspicious input detected'
          }),
          headers: {
            'content-type': [{ key: 'Content-Type', value: 'application/json' }],
          },
        };
      }
    }
  }

  return request; // Forward to origin
};
```

**Limitations:**
- Max execution time: 30 seconds (viewer request/response), 5 seconds (origin request/response)
- Max memory: 10 GB
- No GPU access (CPU-only inference)
- Model size limit: must fit in deployment package (250 MB unzipped)

### Cloudflare Workers

```javascript
// Cloudflare Worker: PII redaction at the edge
addEventListener('fetch', event => {
  event.respondWith(handleRequest(event.request));
});

async function handleRequest(request) {
  // Forward request to origin
  const response = await fetch(request);

  // Only process AI response
  if (!response.headers.get('content-type')?.includes('application/json')) {
    return response;
  }

  // Clone and modify
  const modifiedResponse = new Response(response.body, response);

  // PII pattern detection and redaction
  const body = await modifiedResponse.text();
  const redacted = body
    .replace(/\b\d{3}-\d{2}-\d{4}\b/g, '***-**-****')  // SSN
    .replace(/\b\d{16}\b/g, '****-****-****-$1')  // Credit card
    .replace(/[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}/g, '[EMAIL REDACTED]');  // Email

  return new Response(redacted, {
    status: modifiedResponse.status,
    headers: modifiedResponse.headers,
  });
}
```

**Advantages:**
- Faster cold starts than Lambda@Edge
- More edge locations (300+ vs. ~30 for Lambda@Edge)
- Built-in KV storage for state
- AI inference via Workers AI (limited model selection)

### CloudFront Functions

```javascript
// CloudFront Function: Lightweight request validation
function handler(event) {
  var request = event.request;

  // Validate Content-Type
  if (request.method === 'POST' &&
      request.headers['content-type'] &&
      request.headers['content-type'].value !== 'application/json') {
    return {
      statusCode: 415,
      statusDescription: 'Unsupported Media Type',
    };
  }

  // Validate request size (check Content-Length header)
  if (request.headers['content-length']) {
    var size = parseInt(request.headers['content-length'].value);
    if (size > 10 * 1024 * 1024) { // 10 MB limit
      return {
        statusCode: 413,
        statusDescription: 'Payload Too Large',
      };
    }
  }

  return request;
}
```

**Limitations:**
- JavaScript subset only (no external libraries)
- Max execution time: 1 ms
- Max memory: 2 MB
- Best for simple header/body manipulation

---

## Edge AI Inference Architecture

### Option 1: Lightweight Models at Edge

```
User Request
    │
    ▼
┌─────────────────────────┐
│  Edge Function           │
│  - Intent classification │  <-- Tiny model (< 50 MB)
│  - Language detection    │      Runs on CPU
│  - Abuse detection       │      < 10ms latency
└────────────┬────────────┘
             │
             ├─ Low-risk intent: Answer from cache
             │
             ├─ Medium-risk: Forward to regional LLM
             │
             └─ High-risk: Forward to core LLM with full context
```

**Models suitable for edge:**
- FastText (language detection, text classification)
- TinyBERT (intent classification, sentiment analysis)
- ONNX-optimized models (various tasks)
- TensorFlow Lite models (image classification, NER)

### Option 2: Edge Preprocessing + Core Inference

```
User Request
    │
    ▼
┌─────────────────────────┐
│  Edge Function           │
│  - Input validation      │
│  - Prompt injection check│
│  - PII detection in input│
│  - Rate limiting         │
└────────────┬────────────┘
             │
             ▼
┌─────────────────────────┐
│  Regional LLM Service    │
│  - Full LLM inference    │
│  - RAG retrieval         │
│  - Tool execution        │
└────────────┬────────────┘
             │
             ▼
┌─────────────────────────┐
│  Edge Function           │
│  - PII redaction in output│
│  - Response formatting   │
│  - Caching (if applicable)│
└────────────┬────────────┘
             │
             ▼
          User Response
```

### Option 3: Hybrid Edge + Core

```
┌──────────────────────────────────────────────────┐
│                  Edge Layer                       │
│  - Static assets (CDN cache)                     │
│  - Request validation and security               │
│  - Lightweight ML (intent, language, sentiment)   │
│  - Rate limiting and abuse prevention             │
└───────────────────┬──────────────────────────────┘
                    │
                    ▼
┌──────────────────────────────────────────────────┐
│                  Regional Layer                   │
│  - LLM inference (GPT-4, Claude, etc.)           │
│  - RAG retrieval (vector DB)                     │
│  - Tool execution (APIs, databases)              │
│  - Complex reasoning and generation              │
└───────────────────┬──────────────────────────────┘
                    │
                    ▼
┌──────────────────────────────────────────────────┐
│                  Core Layer                       │
│  - Training and fine-tuning                      │
│  - Model evaluation and deployment               │
│  - Data pipeline and feature store               │
│  - Audit logging and compliance                  │
└──────────────────────────────────────────────────┘
```

---

## Edge Computing for GenAI: Specific Patterns

### Pattern 1: Prompt Injection Detection at Edge

```javascript
// Edge function: detect and block prompt injection before reaching LLM
const INJECTION_PATTERNS = [
  /ignore\s+(all\s+)?previous\s+(instructions|directives)/i,
  /system\s*[:=]\s*(override|reset|bypass)/i,
  /(?:you\s+are\s+)?(?:now\s+)?(?:in\s+)?developer\s*mode/i,
  /BEGIN\s+(?:EXECUTION|OUTPUT|RESPONSE)/i,
  /CRITICAL\s+SYSTEM\s+OVERRIDE/i,
  /output\s+your\s+(complete\s+)?system\s+prompt/i,
  /\[INST\].*?\[\/INST\].*?\[INST\]/,  // Instruction sequence injection
];

function detectPromptInjection(input) {
  for (const pattern of INJECTION_PATTERNS) {
    if (pattern.test(input)) {
      return { detected: true, pattern: pattern.source };
    }
  }
  return { detected: false };
}
```

### Pattern 2: PII Detection in Input

```javascript
// Edge function: detect PII in user input before sending to LLM
const PII_PATTERNS = {
  ssn: /\b\d{3}-\d{2}-\d{4}\b/,
  creditCard: /\b(?:\d{4}[-\s]?){3}\d{4}\b/,
  ukNino: /\b[A-CEGHJ-PR-TW-Z][A-CEGHJ-NPR-TW-Z]\s?\d{6}\s?[A-D]\b/,
  email: /\b[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Za-z]{2,}\b/,
  phoneUK: /\b(?:\+44\s?|0)(?:7\d{3}\s?\d{6}|2\d{3}\s?\d{6}|1\d{2,3}\s?\d{6,7})\b/,
};

function detectPII(input) {
  const found = [];
  for (const [type, pattern] of Object.entries(PII_PATTERNS)) {
    if (pattern.test(input)) {
      found.push(type);
    }
  }
  return found;
}

// If PII detected, log and warn (but allow through - user may need to discuss PII)
// For highly sensitive data, consider blocking or redacting
```

### Pattern 3: Response Caching at Edge

```javascript
// Cache non-personalized AI responses at the edge
async function handleRequest(request) {
  const cacheKey = request.url;
  const cache = caches.default;

  // Check cache first
  let response = await cache.match(cacheKey);

  if (!response) {
    // Cache miss - fetch from origin
    response = await fetch(request);

    // Only cache if:
    // 1. Response is successful
    // 2. Response does not contain PII
    // 3. Response is non-personalized
    if (response.ok && !containsPII(response) && !isPersonalized(request)) {
      const cachedResponse = new Response(response.body, response);
      cachedResponse.headers.set('Cache-Control', 'public, max-age=300');
      cachedResponse.headers.set('CDN-Cache-Control', 'public, max-age=300');
      event.waitUntil(cache.put(cacheKey, cachedResponse));
    }
  }

  return response;
}
```

---

## Edge Computing Limitations

| Limitation | Impact | Mitigation |
|-----------|--------|-----------|
| **No GPU at edge** | Cannot run large models | Use lightweight models, forward heavy inference to core |
| **Memory limits** (2 MB - 10 GB) | Cannot load large models or datasets | Optimize model size, use streaming |
| **Execution time limits** (1 ms - 30 s) | Cannot run long inference jobs | Keep edge processing lightweight |
| **Cold starts** (100 ms - 5 s) | Adds latency for infrequent functions | Provisioned concurrency, keep functions warm |
| **Limited libraries** | Cannot use arbitrary ML frameworks | Use ONNX, TensorFlow Lite, or pre-compiled models |

---

## Edge Computing Cost

| Platform | Pricing Model | Cost per Million Requests |
|----------|--------------|--------------------------|
| **Lambda@Edge** | Request duration + count | ~$0.60 (avg 100ms execution) |
| **Cloudflare Workers** | Request count + CPU time | ~$0.50 (first 10M free) |
| **CloudFront Functions** | Request count | ~$0.10 |

### Cost Comparison: Edge vs. Origin Processing

| Operation | Edge Cost | Origin Cost | Savings |
|-----------|-----------|-------------|---------|
| Request validation | $0.10/M | $1.00/M (compute) | 90% |
| Prompt injection check | $0.50/M | $5.00/M (compute + LLM) | 90% |
| PII redaction | $0.50/M | $5.00/M (compute) | 90% |
| Intent classification (edge model) | $0.50/M | $10.00/M (LLM API) | 95% |

---

## Edge Computing Monitoring

### Key Metrics

| Metric | Alert Threshold | Description |
|--------|---------------|-------------|
| **Edge function errors** | > 0.1% | Function execution errors |
| **Edge latency (P95)** | > 50ms | Edge function execution time |
| **Injection detection rate** | > 5% of requests | Unusual spike in blocked requests |
| **PII detection rate** | > 1% of requests | PII found in user input |
| **Cache hit rate (edge)** | < 50% | Edge cache effectiveness |

---

## Cross-References

- [README.md](README.md) -- Infrastructure overview
- [cdn.md](cdn.md) -- CDN architecture
- [networking.md](networking.md) -- Network architecture
- [compute.md](compute.md) -- Compute options
- [../security/README.md](../security/README.md) -- Security practices
