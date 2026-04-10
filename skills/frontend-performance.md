# Skill: Frontend Performance

## Core Principles

1. **Measure First, Optimize Second** — Never guess what's slow. Use Web Vitals, Lighthouse, and Real User Monitoring (RUM) to identify actual bottlenecks.
2. **The Critical Path Is Sacred** — Minimize what blocks the first paint. Defer everything that isn't needed for the initial view.
3. **Bundle Size Is a Budget** — Treat your JavaScript bundle like a memory budget. Every kilobyte costs milliseconds of load time. Enforce budgets with CI checks.
4. **Render Only What's Visible** — Virtualize long lists, lazy-load below-the-fold content, and use skeletons to keep the UI responsive during data fetching.
5. **Network Is the Bottleneck** — The fastest code is the code that never needs to be downloaded. Reduce payloads, compress aggressively, and cache everything you can.

## Mental Models

### Web Vitals and Thresholds
```
┌─────────────────────────────────────────────────────────────┐
│                    Core Web Vitals                          │
│                                                             │
│  LCP - Largest Contentful Paint                             │
│  When: The largest content element becomes visible          │
│  Target: < 2.5s                                             │
│  Banking context: The chat interface or dashboard loads      │
│                                                             │
│  INP - Interaction to Next Paint                            │
│  When: Time from user interaction to visual response        │
│  Target: < 200ms                                            │
│  Banking context: Sending a message, clicking a document     │
│                                                             │
│  CLS - Cumulative Layout Shift                              │
│  When: Visual stability during page load                    │
│  Target: < 0.1                                              │
│  Banking context: Chat messages shifting as they load        │
│                                                             │
│  FCP - First Contentful Paint                               │
│  When: First content appears on screen                      │
│  Target: < 1.8s                                             │
│  Banking context: User sees something immediately            │
│                                                             │
│  TTFB - Time to First Byte                                  │
│  When: Server response time                                 │
│  Target: < 800ms                                            │
│  Banking context: API Gateway + backend response time        │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### The Performance Budget
```
┌─────────────────────────────────────────────────────────────┐
│              Frontend Performance Budget                     │
│                                                             │
│  JavaScript Bundle:                                         │
│    Main chunk:     < 200KB (gzipped)                        │
│    Vendor chunk:   < 300KB (gzipped)                        │
│    Total initial:  < 500KB (gzipped)                        │
│                                                             │
│  CSS:              < 50KB (gzipped)                         │
│  Images:           < 100KB per image, lazy-loaded           │
│  Fonts:            < 100KB total, subset + WOFF2            │
│                                                             │
│  Time Budgets:                                              │
│    TTFB:           < 800ms                                  │
│    FCP:            < 1.8s                                   │
│    LCP:            < 2.5s                                   │
│    TTI:            < 3.8s                                   │
│    INP:            < 200ms                                  │
│                                                             │
│  Page Size:        < 1.5MB total                            │
│  Requests:         < 50 initial requests                    │
│                                                             │
│  CI Gate: Build fails if budget exceeded                    │
└─────────────────────────────────────────────────────────────┘
```

### The Performance Optimization Priority
```
Priority 1: Reduce Critical Resources (biggest impact)
├── Minimize initial JS bundle (code splitting, tree shaking)
├── Defer non-critical CSS
├── Preload critical resources (fonts, above-the-fold images)
└── Server-side render initial content

Priority 2: Optimize Delivery (high impact)
├── Enable gzip/brotli compression
├── Use HTTP/2 or HTTP/3
├── Set proper cache headers (immutable assets: 1 year)
└── Use CDN for static assets

Priority 3: Optimize Execution (medium impact)
├── Virtualize long lists (React Window)
├── Memoize expensive computations (useMemo, useCallback)
├── Debounce user input (search, chat)
└── Use Web Workers for heavy computation

Priority 4: Optimize Rendering (refinement)
├── Avoid unnecessary re-renders (React.memo)
├── Use CSS transforms instead of layout-triggering properties
├── Batch state updates
└── Use content-visibility for off-screen content
```

## Step-by-Step Approach

### 1. Code Splitting and Lazy Loading (Next.js)

```typescript
// BAD: Everything bundled together
import { ChatInterface } from './components/ChatInterface';
import { DocumentViewer } from './components/DocumentViewer';
import { SettingsPanel } from './components/SettingsPanel';
import { AnalyticsDashboard } from './components/AnalyticsDashboard';

function App() {
  return (
    <div>
      <ChatInterface />
      <DocumentViewer />
      <SettingsPanel />
      <AnalyticsDashboard />
    </div>
  );
}

// GOOD: Route-based code splitting (Next.js does this automatically)
// app/chat/page.tsx — only loads when user visits /chat
export default function ChatPage() {
  return <ChatInterface />;
}

// GOOD: Component-level lazy loading with React.lazy
import { lazy, Suspense } from 'react';
import { LoadingSkeleton } from './components/LoadingSkeleton';

const DocumentViewer = lazy(() => import('./components/DocumentViewer'));
const SettingsPanel = lazy(() => import('./components/SettingsPanel'));

function App() {
  return (
    <div>
      <ChatInterface />
      <Suspense fallback={<LoadingSkeleton />}>
        <DocumentViewer />
      </Suspense>
      <Suspense fallback={<LoadingSkeleton />}>
        <SettingsPanel />
      </Suspense>
    </div>
  );
}

// GOOD: Dynamic imports based on user interaction
async function loadDocumentViewer() {
  const { DocumentViewer } = await import('./components/DocumentViewer');
  return DocumentViewer;
}

// Use when user clicks "View Document"
const [Viewer, setViewer] = useState<React.ComponentType | null>(null);
const handleViewDocument = async () => {
  const Component = await loadDocumentViewer();
  setViewer(() => Component);
};
```

### 2. Virtualized Lists for Large Datasets

```typescript
import { FixedSizeList as List } from 'react-window';
import AutoSizer from 'react-virtualized-auto-sizer';

interface ConversationMessage {
  id: string;
  role: 'user' | 'assistant';
  content: string;
  timestamp: string;
}

// BAD: Rendering thousands of messages directly
function MessageListBad({ messages }: { messages: ConversationMessage[] }) {
  return (
    <div>
      {messages.map(msg => (
        <MessageCard key={msg.id} message={msg} />
      ))}
    </div>
  );
  // With 1000 messages, this creates 1000 DOM nodes → janky scrolling
}

// GOOD: Virtualized list — only renders visible items
function MessageListVirtualized({ messages }: { messages: ConversationMessage[] }) {
  return (
    <AutoSizer>
      {({ height, width }) => (
        <List
          height={height}
          width={width}
          itemCount={messages.length}
          itemSize={80} // Fixed height per message
          itemData={messages}
        >
          {({ index, style, data }) => (
            <div style={style}>
              <MessageCard message={data[index]} />
            </div>
          )}
        </List>
      )}
    </AutoSizer>
  );
  // With 1000 messages, only ~10 are in the DOM at any time
}

// GOOD: Infinite loading with react-query + virtualization
import { useInfiniteQuery } from '@tanstack/react-query';

function MessageListInfinite() {
  const {
    data,
    fetchNextPage,
    hasNextPage,
    isFetchingNextPage,
  } = useInfiniteQuery({
    queryKey: ['conversation-messages', conversationId],
    queryFn: ({ pageParam = 0 }) =>
      fetchMessages(conversationId, { page: pageParam, limit: 50 }),
    getNextPageParam: (lastPage) => lastPage.nextPage,
  });

  const allMessages = data?.pages.flatMap(page => page.messages) ?? [];

  return (
    <AutoSizer>
      {({ height, width }) => (
        <List
          height={height}
          width={width}
          itemCount={allMessages.length + (hasNextPage ? 1 : 0)}
          itemSize={80}
          itemData={allMessages}
          onItemsRendered={({ visibleStopIndex }) => {
            // Load more when user scrolls near the bottom
            if (visibleStopIndex >= allMessages.length - 10 && hasNextPage) {
              fetchNextPage();
            }
          }}
        >
          {({ index, style, data }) => {
            if (index >= data.length) {
              return <div style={style}><LoadingSpinner /></div>;
            }
            return (
              <div style={style}>
                <MessageCard message={data[index]} />
              </div>
            );
          }}
        </List>
      )}
    </AutoSizer>
  );
}
```

### 3. Optimizing API Data Fetching

```typescript
import { useQuery } from '@tanstack/react-query';

// BAD: Fetching on every render, no caching
function ChatBad() {
  const [conversations, setConversations] = useState([]);

  useEffect(() => {
    fetch('/api/conversations')
      .then(res => res.json())
      .then(setConversations);
  }, []);
  // No caching, no error handling, no loading state, no refetching
}

// GOOD: React Query with proper configuration
function ChatGood() {
  const {
    data: conversations,
    isLoading,
    error,
    isFetching,
    refetch,
  } = useQuery({
    queryKey: ['conversations', userId],
    queryFn: () => fetchConversations(userId),
    staleTime: 5 * 60 * 1000,     // Data is fresh for 5 minutes
    gcTime: 30 * 60 * 1000,       // Keep in cache for 30 minutes
    refetchOnWindowFocus: false,  // Don't refocus on window focus
    retry: 2,                     // Retry failed requests twice
    retryDelay: (attemptIndex) => Math.min(1000 * 2 ** attemptIndex, 10000),
  });

  if (isLoading) return <ConversationListSkeleton />;
  if (error) return <ErrorMessage onRetry={() => refetch()} />;

  return (
    <ConversationList
      conversations={conversations}
      isRefreshing={isFetching}
    />
  );
}

// GOOD: Prefetching data when user is likely to need it
const queryClient = useQueryClient();

// Prefetch conversation details when user hovers over a conversation
const handleHover = (conversationId: string) => {
  queryClient.prefetchQuery({
    queryKey: ['conversation', conversationId],
    queryFn: () => fetchConversation(conversationId),
    staleTime: 2 * 60 * 1000,
  });
};
```

### 4. Optimizing Re-renders with React.memo and useMemo

```typescript
// BAD: Re-renders on every parent state change
function MessageItem({ message, isSelected, onSelect }: MessageItemProps) {
  // Expensive: syntax highlighting on every render
  const highlightedContent = highlightSyntax(message.content);

  return (
    <div onClick={() => onSelect(message.id)} className={isSelected ? 'selected' : ''}>
      <div className="content">{highlightedContent}</div>
      <div className="timestamp">{formatDate(message.timestamp)}</div>
    </div>
  );
}

// GOOD: Memoized component with computed values
const MessageItem = React.memo(function MessageItem({
  message,
  isSelected,
  onSelect,
}: MessageItemProps) {
  // Only recompute when message.content changes
  const highlightedContent = useMemo(
    () => highlightSyntax(message.content),
    [message.content]
  );

  const formattedDate = useMemo(
    () => formatDate(message.timestamp),
    [message.timestamp]
  );

  // Stable callback reference
  const handleClick = useCallback(
    () => onSelect(message.id),
    [onSelect, message.id]
  );

  return (
    <div onClick={handleClick} className={isSelected ? 'selected' : ''}>
      <div className="content">{highlightedContent}</div>
      <div className="timestamp">{formattedDate}</div>
    </div>
  );
});

// When to use each:
// React.memo: When the component receives the same props frequently
// useMemo: When computing an expensive value from props/state
// useCallback: When passing a callback to a memoized child component
//
// Don't overuse: The cost of memoization is non-zero.
// Only apply when profiling shows re-renders are a problem.
```

### 5. Performance Monitoring with Web Vitals

```typescript
// Report Web Vitals to analytics/observability platform
import { onLCP, onINP, onCLS, onFCP, onTTFB } from 'web-vitals';

function sendToAnalytics(metric: {
  name: string;
  value: number;
  delta: number;
  rating: string;
  id: string;
  navigationType: string;
}) {
  // Send to your observability platform
  fetch('/api/metrics', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      metric_name: `web_vital_${metric.name.toLowerCase()}`,
      value: metric.value,
      rating: metric.rating,
      page: window.location.pathname,
      user_agent: navigator.userAgent,
      connection: (navigator as any).connection?.effectiveType,
    }),
    keepalive: true,
  });
}

// Register observers
onLCP(sendToAnalytics);
onINP(sendToAnalytics);
onCLS(sendToAnalytics);
onFCP(sendToAnalytics);
onTTFB(sendToAnalytics);

// Also report to your RUM (Real User Monitoring) platform
// e.g., Datadog RUM, New Relic, or internal platform
```

### 6. Next.js-Specific Optimizations

```typescript
// next.config.js — Performance optimizations
const nextConfig = {
  // Compress responses
  compress: true,

  // Image optimization
  images: {
    formats: ['image/avif', 'image/webp'],
    minimumCacheTTL: 31536000, // 1 year
    deviceSizes: [640, 750, 828, 1080, 1200],
    imageSizes: [16, 32, 48, 64, 96, 128, 256],
  },

  // Analyze bundle size (use @next/bundle-analyzer)
  webpack: (config, { isServer }) => {
    if (!isServer) {
      // Don't include moment.js locale files
      config.resolve.alias['moment/locale'] = false;
    }
    return config;
  },
};

// Use next/image for automatic optimization
import Image from 'next/image';

function DocumentCard({ document }: { document: Doc }) {
  return (
    <div>
      {/* BAD: Regular img tag — no optimization */}
      <img src={document.thumbnail} alt={document.title} />

      {/* GOOD: Next.js Image — automatic optimization */}
      <Image
        src={document.thumbnail}
        alt={document.title}
        width={200}
        height={150}
        loading="lazy"           // Lazy loading
        placeholder="blur"       // Blur placeholder while loading
        sizes="(max-width: 768px) 100vw, 200px" // Responsive sizes
        quality={75}             // Compression quality
      />
    </div>
  );
}

// Use next/font for zero layout shift font loading
import { Inter } from 'next/font/google';

const inter = Inter({
  subsets: ['latin'],
  display: 'swap',
  preload: true,
  variable: '--font-inter',
});

export default function RootLayout({ children }: { children: React.ReactNode }) {
  return (
    <html className={inter.variable}>
      <body>{children}</body>
    </html>
  );
}
```

### 7. Bundle Analysis and Enforcement

```bash
# Install bundle analyzer
npm install --save-dev @next/bundle-analyzer

# next.config.js
const withBundleAnalyzer = require('@next/bundle-analyzer')({
  enabled: process.env.ANALYZE === 'true',
});

module.exports = withBundleAnalyzer(nextConfig);

# Run analysis
ANALYZE=true npm run build

# This opens an interactive treemap showing bundle composition
# Look for:
# - Large dependencies you didn't know you had
# - Duplicate packages (same package at different versions)
# - Opportunities for code splitting

# Enforce bundle size limits in CI
# webpack.config.js (or via next.config.js)
const BundleAnalyzerPlugin = require('webpack-bundle-analyzer').BundleAnalyzerPlugin;
const CompressionPlugin = require('compression-webpack-plugin');

module.exports = {
  plugins: [
    new CompressionPlugin({
      algorithm: 'brotliCompress',
      test: /\.(js|css|html|svg)$/,
      threshold: 10240,  // Only compress files > 10KB
      minRatio: 0.8,
    }),
    // Fail the build if bundle exceeds budget
    new webpack.performanceHintsPlugin({
      maxAssetSize: 250000,  // 250KB
      maxEntrypointSize: 500000,  // 500KB
      hints: 'error',
    }),
  ],
};
```

## Common Pitfalls

| Pitfall | Consequence | Fix |
|---------|------------|-----|
| Monolithic JavaScript bundle | Slow initial load, poor caching | Code splitting, lazy loading |
| Not compressing images | Massive payloads, slow LCP | Use next/image, WebP/AVIF formats |
| Rendering long lists without virtualization | Janky scrolling, high memory | react-window, react-virtualized |
| Unnecessary re-renders | Poor INP, CPU waste | React.memo, useMemo, useCallback |
| Fetching data in useEffect without caching | Duplicate requests, slow interactions | React Query with caching |
| Not setting cache headers | Re-downloading static assets on every visit | Immutable cache for hashed assets |
| Loading fonts without font-display: swap | Invisible text during font load | Use next/font or font-display: swap |
| Blocking the main thread with heavy computation | Unresponsive UI during processing | Web Workers, requestIdleCallback |
| Not measuring performance | Optimizing the wrong thing | Web Vitals, Lighthouse, RUM |
| Over-memoization | Code complexity with minimal benefit | Profile first, memoize only hot paths |

## Banking-Specific Concerns

1. **Internal Network Constraints** — Banking internal apps may run on slower networks (especially for remote workers on VPN). Optimize for higher latency and lower bandwidth than consumer apps.
2. **Accessibility Requirements** — Banking apps must meet WCAG 2.1 AA standards. Performance optimizations must not break accessibility (e.g., lazy loading must not remove content from screen readers).
3. **Legacy Browser Support** — Some banking environments still use older browsers. Ensure polyfills are loaded conditionally (not in the main bundle).
4. **Corporate Proxy Interference** — Corporate proxies may strip cache headers or compress responses. Test in the actual deployment environment.
5. **Session Timeout Impact** — Banking sessions time out frequently. Prefetched data may become stale. Handle expired sessions gracefully.

## GenAI-Specific Concerns

1. **Streaming Responses** — GenAI responses stream token by token. Use streaming UI patterns (append each token to the DOM as it arrives) rather than waiting for the complete response.
2. **Markdown Rendering** — LLM responses often include markdown. Use efficient markdown renderers and memoize the rendering to avoid re-parsing on every token.
3. **Long Conversation Context** — As conversations grow, re-rendering the entire message list becomes expensive. Use virtualization and only render new messages.
4. **Syntax Highlighting Cost** — Code blocks in AI responses require syntax highlighting, which is computationally expensive. Lazy-load the highlighter and only highlight visible code blocks.
5. **Typing Animation Overhead** — Simulated typing animations cause constant re-renders. Use CSS animations instead of JavaScript-driven animations for better performance.

## Metrics to Monitor

| Metric | Alert Threshold | Why It Matters |
|--------|----------------|----------------|
| LCP (p75) | > 2.5s | Users perceive slow loading |
| INP (p75) | > 200ms | UI feels unresponsive |
| CLS (p75) | > 0.1 | Visual instability, misclicks |
| JS bundle size | > 500KB (gzipped) | Slow initial load |
| Time to Interactive | > 3.8s | Page appears but isn't usable |
| React re-render count | Track trend per component | Unnecessary CPU usage |
| API response time (frontend) | > 1s | Backend or network issue |
| Cache hit rate | < 80% | Assets being re-downloaded |
| Error rate (client-side) | > 1% | JavaScript errors breaking UI |
| Memory usage (tab) | > 500MB | Memory leak or excessive DOM |

## Interview Questions

1. How would you diagnose and fix a slow-loading React application?
2. What is code splitting and how does it improve performance?
3. How do you optimize a React component that renders a list of 10,000 items?
4. What are Core Web Vitals and why do they matter?
5. How do you prevent unnecessary re-renders in React?
6. What is the difference between `useMemo` and `useCallback`? When would you use each?
7. How would you set up a performance budget for a frontend application?
8. What strategies would you use to optimize image loading in a document-heavy application?

## Hands-On Exercise

### Exercise: Optimize a Slow GenAI Chat Interface

**Problem:** The GenAI chat interface for the bank's internal assistant is slow. Users report:
- The page takes 5+ seconds to load
- Scrolling through long conversations is janky
- The UI freezes while the AI response is being rendered (with markdown and code blocks)
- Each new token causes a visible lag

**Constraints:**
- The current bundle size is 1.2MB (uncompressed)
- Conversations can have 500+ messages
- Responses include markdown, code blocks with syntax highlighting, and citations
- The app uses React 18 and Next.js 14
- You cannot change the backend API

**Expected Output:**
- Bundle analysis showing the biggest contributors
- Code splitting strategy with lazy-loaded components
- Virtualized message list implementation
- Streaming response rendering optimization
- Memoization strategy for expensive computations (markdown parsing, syntax highlighting)
- Performance budget for the application

**Hints:**
- Run `ANALYZE=true npm run build` to identify bundle bloat
- Use `react-window` for the message list
- Lazy-load the markdown renderer and syntax highlighter
- Use React streaming for the AI response
- Memoize markdown parsing with `useMemo`

**Extension:**
- Set up Web Vitals monitoring and alerting
- Implement a skeleton screen that matches the final layout
- Add a performance regression test to CI (fail the build if LCP degrades by > 10%)
- Implement a Service Worker for offline access to recent conversations

---

**Related files:**
- `frontend-engineering/react-performance.md` — React performance deep-dive
- `frontend-engineering/nextjs-optimization.md` — Next.js-specific optimizations
- `frontend-engineering/bundle-optimization.md` — Bundle size optimization
- `observability/frontend-monitoring.md` — RUM and Web Vitals
- `skills/backend-performance.md` — Backend performance optimization
