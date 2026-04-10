# Frontend Observability for Banking GenAI

## Real User Monitoring (RUM)

Frontend observability captures what users actually experience in their browsers and mobile apps. Backend metrics tell you how fast your server responded. RUM tells you how fast the page actually rendered for the user -- which can be dramatically different.

### Why Frontend Observability Matters for Banking GenAI

A GenAI chat interface in a banking app has unique frontend challenges:

1. **Streaming responses**: Users watch text appear character by character. Time-to-first-token matters more than total response time.
2. **Complex UI state**: Loading states, typing indicators, error states, and streaming animations all affect perceived performance.
3. **Mobile-first**: Many banking customers access advisory services from mobile devices with variable network conditions.
4. **Trust signals**: Users need to see disclaimers, source citations, and confidence indicators -- rendering these affects experience.

## Core Web Vitals

Google's Core Web Vitals are the standard metrics for user experience:

### Largest Contentful Paint (LCP)

**What it measures**: Time until the largest content element is visible. For a GenAI chat, this is typically the first rendered response.

**Target**: < 2.5 seconds

```javascript
// Measure LCP
import { onLCP } from 'web-vitals';

onLCP((metric) => {
  // Send to observability
  sendToAnalytics({
    metric: 'LCP',
    value: metric.value,
    rating: metric.rating,  // 'good', 'needs-improvement', 'poor'
    page: window.location.pathname,
    user_segment: getUserSegment(),  // Never PII
    feature: 'genai-chat'
  });
});
```

**Banking context**: A customer asking "Can I afford this home?" waits to see the advisory response. LCP measures when they first see the answer.

### First Input Delay (FID) / Interaction to Next Paint (INP)

**What it measures**: Time from user interaction (click, tap, keypress) to the browser's next paint. Measures interactivity.

**Target**: FID < 100ms, INP < 200ms

```javascript
import { onINP } from 'web-vitals';

onINP((metric) => {
  sendToAnalytics({
    metric: 'INP',
    value: metric.value,
    rating: metric.rating,
    page: window.location.pathname,
    feature: 'genai-chat',
    // Did the user experience jank while typing?
    interaction_type: metric.entries[0]?.name
  });
});
```

**Banking context**: A customer typing a follow-up question about mortgage rates should see immediate response. Delayed input feedback feels broken.

### Cumulative Layout Shift (CLS)

**What it measures**: Unexpected layout shifts during page lifetime. Content that moves around while reading is disorienting.

**Target**: < 0.1

```javascript
import { onCLS } from 'web-vitals';

onCLS((metric) => {
  sendToAnalytics({
    metric: 'CLS',
    value: metric.value,
    rating: metric.rating,
    page: window.location.pathname,
    feature: 'genai-chat',
    // What caused the layout shift?
    largest_shift_source: metric.entries[0]?.sources?.[0]?.node?.tagName
  });
});
```

**Banking context**: Streaming text that causes the page to jump around while a customer is reading financial advice is confusing and untrustworthy.

## Frontend Error Tracking

### JavaScript Errors

```javascript
// Global error handler
window.addEventListener('error', (event) => {
  sendToAnalytics({
    type: 'js_error',
    message: event.message,
    filename: event.filename,
    lineno: event.lineno,
    colno: event.colno,
    stack: event.error?.stack,
    url: window.location.href,
    feature: 'genai-chat',
    user_agent: navigator.userAgent
  });
});

// Unhandled promise rejections
window.addEventListener('unhandledrejection', (event) => {
  sendToAnalytics({
    type: 'unhandled_promise_rejection',
    message: event.reason?.message || String(event.reason),
    stack: event.reason?.stack,
    url: window.location.href,
    feature: 'genai-chat'
  });
});
```

### API Error Tracking

```javascript
// Intercept API errors
async function apiCall(endpoint, options) {
  const startTime = performance.now();
  try {
    const response = await fetch(`/api${endpoint}`, options);
    const duration = performance.now() - startTime;

    if (!response.ok) {
      sendToAnalytics({
        type: 'api_error',
        endpoint,
        status: response.status,
        duration_ms: duration,
        method: options.method || 'GET',
        feature: 'genai-chat'
      });
      throw new Error(`API error: ${response.status}`);
    }

    return response.json();
  } catch (error) {
    sendToAnalytics({
      type: 'api_error',
      endpoint,
      error: error.message,
      duration_ms: performance.now() - startTime,
      feature: 'genai-chat'
    });
    throw error;
  }
}
```

## GenAI-Specific Frontend Metrics

### Time to First Token (TTFT)

The most important latency metric for streaming GenAI responses:

```javascript
async function streamResponse(query) {
  const startTime = performance.now();
  let firstTokenTime = null;
  let fullResponse = '';

  const response = await fetch('/api/v1/chat', {
    method: 'POST',
    body: JSON.stringify({ query }),
    headers: { 'Content-Type': 'application/json' }
  });

  const reader = response.body.getReader();
  const decoder = new TextDecoder();

  while (true) {
    const { done, value } = await reader.read();
    if (done) break;

    const text = decoder.decode(value);
    if (firstTokenTime === null) {
      firstTokenTime = performance.now() - startTime;
      sendToAnalytics({
        type: 'genai_ttft',
        value_ms: firstTokenTime,
        query_length: query.length,
        user_tier: getUserTier(),
        feature: 'genai-chat'
      });
    }

    fullResponse += text;
    updateUI(fullResponse);
  }

  // Total response time
  const totalTime = performance.now() - startTime;
  sendToAnalytics({
    type: 'genai_total_response',
    value_ms: totalTime,
    ttft_ms: firstTokenTime,
    response_length: fullResponse.length,
    feature: 'genai-chat'
  });
}
```

### Streaming Render Performance

```javascript
// Measure rendering performance during streaming
let tokensReceived = 0;
let streamStartTime = performance.now();

function onTokenReceived(token) {
  tokensReceived++;
  const elapsed = performance.now() - streamStartTime;

  // Report tokens per second every 100 tokens
  if (tokensReceived % 100 === 0) {
    const tokensPerSecond = (tokensReceived / elapsed) * 1000;
    sendToAnalytics({
      type: 'genai_streaming_rate',
      tokens_per_second: tokensPerSecond,
      total_tokens: tokensReceived,
      elapsed_ms: elapsed,
      feature: 'genai-chat'
    });
  }
}
```

### User Engagement with AI Responses

```javascript
// Track how users interact with AI responses
function trackResponseInteraction(responseId, interaction) {
  sendToAnalytics({
    type: 'ai_response_interaction',
    response_id: responseId,
    interaction,  // 'thumbs_up', 'thumbs_down', 'copied', 'expanded_sources'
    time_to_interact_ms: getTimeSinceResponse(),
    response_length: getResponseLength(responseId),
    feature: 'genai-chat'
  });
}
```

## Frontend Observability Architecture

```
┌─────────────────────────────────────────────────────────────┐
│  Browser / Mobile App                                       │
│  ┌──────────────────────────────────────────────────────┐   │
│  │  Web Vitals SDK (web-vitals)                         │   │
│  │  Error Tracking (Sentry/DataDog RUM)                 │   │
│  │  Custom GenAI Metrics (TTFT, streaming rate)         │   │
│  │  User Feedback Tracking                              │   │
│  └────────────────────────┬─────────────────────────────┘   │
│                           │                                  │
│                    Beacon API                                │
│                    (async, non-blocking)                     │
└───────────────────────────┬─────────────────────────────────┘
                            │
                 ┌──────────┴──────────┐
                 │  Telemetry          │
                 │  Ingestion          │
                 │  (RUM SDK)          │
                 └──────────┬──────────┘
                            │
              ┌─────────────┼─────────────┐
              │             │             │
         ┌────┴────┐  ┌────┴────┐  ┌────┴────┐
         │  Web    │  │  Error  │  │ Custom  │
         │ Vitals  │  │ Tracking│  │ GenAI   │
         │ Dashboard│ │ Dashboard│ │Dashboard│
         └─────────┘  └─────────┘  └─────────┘
```

## Frontend Dashboard

```
┌──────────────────────────────────────────────────────────────┐
│  GENAI CHAT - FRONTEND EXPERIENCE                            │
├──────────────┬──────────────┬──────────────┬─────────────────┤
│  LCP         │  INP         │  CLS         │  TTFT           │
│  1.8s        │  45ms        │  0.03        │  1.2s           │
│  (good)      │  (good)      │  (good)      │  (target <2s)   │
│  -12% vs lw  │  +5% vs lw   │  stable      │  +8% vs lw      │
└──────────────┴──────────────┴──────────────┴─────────────────┘

┌──────────────────────────────┬───────────────────────────────┐
│  Web Vitals Over Time        │  Error Rate by Browser        │
│  [Time series for LCP/CLS]   │  Chrome: 0.1%                 │
│                              │  Safari: 0.3%                 │
│                              │  Firefox: 0.2%                │
│                              │  Mobile Safari: 0.5%          │
└──────────────────────────────┴───────────────────────────────┘

┌──────────────────────────────┬───────────────────────────────┐
│  User Satisfaction            │  API Latency (frontend view)  │
│  Thumbs up: 87%              │  p50: 2.1s                    │
│  Thumbs down: 13%            │  p95: 4.8s                    │
│  Trend: +3% vs last week     │  p99: 8.2s                    │
└──────────────────────────────┴───────────────────────────────┘
```

## Performance Optimization for Frontend

### Reducing LCP for GenAI Chat

```javascript
// Preload the API connection
const preconnect = document.createElement('link');
preconnect.rel = 'preconnect';
preconnect.href = 'https://api.bank.example';
document.head.appendChild(preconnect);

// Use skeleton UI to improve perceived LCP
function showSkeleton() {
  document.getElementById('chat-response-skeleton').style.display = 'block';
}

function hideSkeleton() {
  document.getElementById('chat-response-skeleton').style.display = 'none';
}
```

### Reducing CLS for Streaming Text

```css
/* Reserve space for streaming response */
.chat-response-container {
  min-height: 200px;
}

/* Prevent layout shift when sources expand */
.sources-section {
  max-height: 0;
  overflow: hidden;
  transition: max-height 0.3s ease;
}

.sources-section.expanded {
  max-height: 500px;
}
```

## Common Mistakes

1. **Only monitoring backend latency**: A 500ms backend response can become 3 seconds for a user on a slow mobile connection. Monitor both.

2. **Ignoring mobile users**: Banking customers increasingly use mobile apps. Desktop-only RUM misses the majority of users.

3. **Not tracking TTFT**: For streaming GenAI, total response time is less important than time to first token. Users can wait for a long response if they see progress immediately.

4. **Missing CLS from dynamic content**: Expanding source citations, disclaimers, and follow-up suggestions cause layout shifts that hurt user trust.

5. **Not correlating frontend and backend**: If frontend shows 5-second latency but backend shows 500ms, the network is the bottleneck. Correlate both views.

6. **Sampling too aggressively**: RUM data is cheap to store. Sample at 100% for production GenAI features to catch rare but impactful issues.
