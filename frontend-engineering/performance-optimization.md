# Performance Optimization — Code Splitting, Virtualization, Memoization, Bundle Analysis, Web Vitals

## Overview

Performance is a feature, not an afterthought. Our users include employees on branch office networks with limited bandwidth. The frontend must feel fast for everyone.

## Performance Budgets

| Metric | Budget | Measurement |
|--------|--------|-------------|
| First Contentful Paint (FCP) | < 1.5s | Lighthouse, RUM |
| Largest Contentful Paint (LCP) | < 2.5s | Lighthouse, RUM |
| Time to Interactive (TTI) | < 3.5s | Lighthouse |
| Cumulative Layout Shift (CLS) | < 0.1 | Lighthouse, RUM |
| Interaction to Next Paint (INP) | < 200ms | Lighthouse, RUM |
| Total Bundle (initial) | < 200KB gzipped | Bundle analysis |
| Total Bundle (route) | < 100KB gzipped | Route analysis |
| JavaScript parse time | < 200ms | DevTools |

## Code Splitting

### Next.js Automatic Code Splitting

Next.js automatically code-splits by route. Each route gets its own JavaScript chunk.

```tsx
// Next.js automatically creates separate chunks for:
// /dashboard/chat    → chat page chunk
// /dashboard/compliance → compliance page chunk
// Shared components → shared chunk

// No manual configuration needed for route-level splitting
```

### Dynamic Imports for Heavy Components

```tsx
// src/app/(dashboard)/compliance/page.tsx
import dynamic from 'next/dynamic';

// Heavy chart component — loaded only when compliance page renders
const ComplianceChart = dynamic(
  () => import('@/components/compliance/ComplianceChart'),
  {
    loading: () => <ChartSkeleton />,
    ssr: false,  // Skip server rendering for chart (canvas-based)
  },
);

// Audit log table with virtualization — heavy dependency
const AuditLogTable = dynamic(
  () => import('@/components/audit/AuditLogTable'),
  {
    loading: () => <TableSkeleton rows={5} />,
  },
);

// Markdown renderer — only needed for assistant messages
const MarkdownRenderer = dynamic(
  () => import('@/components/chat/MarkdownRenderer'),
  {
    loading: () => <div className="animate-pulse h-24 bg-muted rounded" />,
  },
);
```

### Conditional Loading with Intersection Observer

```tsx
// src/components/shared/LazySection.tsx
import { useRef, useState, useEffect } from 'react';

interface LazySectionProps {
  children: React.ReactNode;
  rootMargin?: string;
}

export function LazySection({ children, rootMargin = '200px' }: LazySectionProps) {
  const ref = useRef<HTMLDivElement>(null);
  const [isVisible, setIsVisible] = useState(false);

  useEffect(() => {
    const observer = new IntersectionObserver(
      ([entry]) => {
        if (entry.isIntersecting) {
          setIsVisible(true);
          observer.disconnect();
        }
      },
      { rootMargin },
    );

    if (ref.current) {
      observer.observe(ref.current);
    }

    return () => observer.disconnect();
  }, [rootMargin]);

  return (
    <div ref={ref}>
      {isVisible ? children : <div className="h-48 bg-muted/20 rounded" />}
    </div>
  );
}

// Usage
function ComplianceDashboard() {
  return (
    <div className="space-y-6">
      <MetricsPanel />  {/* Above the fold — loads immediately */}
      <AlertsPanel />   {/* Above the fold */}
      <LazySection>
        <HistoricalTrends /> {/* Below the fold — loads when scrolled into view */}
      </LazySection>
    </div>
  );
}
```

## Virtualization

For lists with 100+ items, always virtualize.

```tsx
// src/components/audit/AuditLogTable.tsx
import { useRef } from 'react';
import { useVirtualizer } from '@tanstack/react-virtual';
import type { AuditLog } from '@/types';

interface AuditLogTableProps {
  logs: AuditLog[];
  hasNextPage: boolean;
  fetchNextPage: () => void;
}

export function AuditLogTable({ logs, hasNextPage, fetchNextPage }: AuditLogTableProps) {
  const parentRef = useRef<HTMLDivElement>(null);

  const virtualizer = useVirtualizer({
    count: logs.length,
    getScrollElement: () => parentRef.current,
    estimateSize: () => 56,  // Estimated row height in pixels
    overscan: 5,             // Render 5 rows above and below viewport
  });

  return (
    <div className="space-y-4">
      <div
        ref={parentRef}
        className="h-96 overflow-auto border rounded-lg"
        onScroll={(e) => {
          const target = e.currentTarget;
          const isNearBottom =
            target.scrollHeight - target.scrollTop - target.clientHeight < 200;
          if (isNearBottom && hasNextPage) {
            fetchNextPage();
          }
        }}
        role="log"
        aria-label="Audit log entries"
      >
        <div
          style={{
            height: `${virtualizer.getTotalSize()}px`,
            width: '100%',
            position: 'relative',
          }}
        >
          {virtualizer.getVirtualItems().map((virtualRow) => {
            const log = logs[virtualRow.index];
            return (
              <div
                key={log.id}
                ref={virtualizer.measureElement}
                data-index={virtualRow.index}
                className="absolute left-0 top-0 flex w-full items-center gap-4 border-b px-4"
                style={{
                  height: `${virtualRow.size}px`,
                  transform: `translateY(${virtualRow.start}px)`,
                }}
              >
                <AuditLogCell>{log.timestamp.toLocaleString()}</AuditLogCell>
                <AuditLogCell>{log.user}</AuditLogCell>
                <AuditLogCell>{log.action}</AuditLogCell>
                <AuditLogCell>{log.resource}</AuditLogCell>
                <AuditLogCell>
                  <StatusBadge status={log.status} />
                </AuditLogCell>
              </div>
            );
          })}
        </div>
      </div>
    </div>
  );
}

function AuditLogCell({ children }: { children: React.ReactNode }) {
  return (
    <div className="min-w-0 flex-1 truncate text-sm py-3">
      {children}
    </div>
  );
}
```

### Virtualizing Chat Messages

```tsx
// src/components/chat/MessageList.tsx
import { useRef, useEffect } from 'react';
import { useVirtualizer } from '@tanstack/react-virtual';
import type { Message } from '@/types';

interface MessageListProps {
  messages: Message[];
  isStreaming: boolean;
}

export function MessageList({ messages, isStreaming }: MessageListProps) {
  const parentRef = useRef<HTMLDivElement>(null);

  const virtualizer = useVirtualizer({
    count: messages.length,
    getScrollElement: () => parentRef.current,
    estimateSize: () => 100,  // Estimated message height
    overscan: 3,
  });

  // Auto-scroll to bottom when new messages arrive
  useEffect(() => {
    if (parentRef.current) {
      parentRef.current.scrollTop = parentRef.current.scrollHeight;
    }
  }, [messages.length]);

  return (
    <div ref={parentRef} className="flex-1 overflow-auto" role="log" aria-live="polite">
      <div style={{ height: `${virtualizer.getTotalSize()}px`, position: 'relative' }}>
        {virtualizer.getVirtualItems().map((virtualRow) => {
          const message = messages[virtualRow.index];
          return (
            <div
              key={message.id}
              ref={virtualizer.measureElement}
              data-index={virtualRow.index}
              className="absolute left-0 w-full"
              style={{
                height: `${virtualRow.size}px`,
                transform: `translateY(${virtualRow.start}px)`,
              }}
            >
              <MessageBubble variant={message.role}>
                <MessageContent content={message.content} />
              </MessageBubble>
            </div>
          );
        })}
      </div>
    </div>
  );
}
```

## Memoization

### When to Memoize

```tsx
// ✅ Memoize expensive computations
function MessageContent({ markdown }: { markdown: string }) {
  // Markdown parsing and sanitization is expensive
  const renderedHtml = useMemo(
    () => sanitizeAndRenderMarkdown(markdown),
    [markdown],
  );

  return <div dangerouslySetInnerHTML={{ __html: renderedHtml }} />;
}

// ✅ Memoize callback when it's a dependency
function ChatInput({ onSend }: { onSend: (text: string) => void }) {
  // onSend is stable — ChatInput won't re-render unnecessarily
}

// ❌ Don't memoize trivially cheap operations
const displayName = useMemo(() => user.name, [user.name]); // Just property access — unnecessary
```

### React.memo — Use Sparingly

```tsx
// ✅ GOOD: Memoize when parent re-renders frequently
const MessageBubble = React.memo(function MessageBubble({
  message,
  isStreaming,
}: MessageBubbleProps) {
  return (
    <div className={cn('p-4', isStreaming && 'animate-pulse')}>
      <MessageContent content={message.content} />
    </div>
  );
});

// ❌ BAD: Memoize everything
// The cost of props comparison can exceed the cost of re-rendering
```

## Bundle Analysis

### Weekly Bundle Audit

```bash
# Install bundle analyzer
npm install --save-dev @next/bundle-analyzer

# next.config.ts
import bundleAnalyzer from '@next/bundle-analyzer';
const withBundleAnalyzer = bundleAnalyzer({ enabled: process.env.ANALYZE === 'true' });

export default withBundleAnalyzer({
  // ... Next.js config
});

# Run analysis
ANALYZE=true npm run build
```

### Bundle Budget in CI

```json
// package.json
{
  "bundlesize": [
    {
      "path": ".next/static/chunks/main-*.js",
      "maxSize": "50 kB"
    },
    {
      "path": ".next/static/chunks/app/(dashboard)/chat/*.js",
      "maxSize": "80 kB"
    },
    {
      "path": ".next/static/chunks/app/(dashboard)/compliance/*.js",
      "maxSize": "100 kB"
    }
  ]
}
```

## Web Vitals Monitoring

```tsx
// src/app/layout.tsx
import { WebVitalsReporter } from '@/components/shared/WebVitalsReporter';

export default function RootLayout({ children }: { children: React.ReactNode }) {
  return (
    <html lang="en">
      <body>
        {children}
        <WebVitalsReporter />
      </body>
    </html>
  );
}

// src/components/shared/WebVitalsReporter.tsx
'use client';

import { useReportWebVitals } from 'next/web-vitals';
import type { Metric } from 'next/web-vitals';

export function WebVitalsReporter() {
  useReportWebVitals((metric: Metric) => {
    // Send to our telemetry platform
    sendToTelemetry({
      name: metric.name,
      value: metric.value,
      delta: metric.delta,
      rating: metric.rating,
      id: metric.id,
      navigationType: metric.navigationType,
      url: window.location.href,
    });
  });

  return null; // This component renders nothing
}

// Track budget violations
function checkBudgets(metric: Metric) {
  const budgets = {
    FCP: 1500,
    LCP: 2500,
    INP: 200,
    CLS: 0.1,
  };

  const budget = budgets[metric.name as keyof typeof budgets];
  if (budget && metric.value > budget) {
    // Alert the team
    sendBudgetAlert({ metric: metric.name, value: metric.value, budget });
  }
}
```

## Common Mistakes

### 1. Over-Memoization

```tsx
// ❌ BAD: Memoizing a simple display
const UserName = React.memo(({ name }: { name: string }) => <span>{name}</span>);

// ✅ GOOD: Only memoize when profiling shows a problem
const UserName = ({ name }: { name: string }) => <span>{name}</span>;
```

### 2. Not Virtualizing Long Lists

```tsx
// ❌ BAD: Rendering 10,000 audit log rows
{auditLogs.map(log => <AuditLogRow key={log.id} log={log} />)}

// ✅ GOOD: Virtualize
<AuditLogTable logs={auditLogs} />
```

### 3. Shipping Unnecessary Dependencies

```tsx
// ❌ BAD: Importing all of lodash
import _ from 'lodash';

// ✅ GOOD: Import only what you need
import debounce from 'lodash/debounce';
```

### 4. Not Setting Image Dimensions

```tsx
// ❌ BAD: Layout shift
<img src="/bank-logo.png" />

// ✅ GOOD: Explicit dimensions prevent CLS
<img src="/bank-logo.png" width="120" height="40" alt="Bank Logo" />
```

## Cross-References

- `./react-query.md` — Prefetching for perceived performance
- `./frontend-observability.md` — Web Vitals reporting
- `./accessibility.md` — Performance impacts accessibility
- `./genai-chat-interfaces.md` — Chat message list virtualization
- `./streaming-responses.md` — Streaming render performance

## Interview Questions

1. How do you enforce performance budgets in a CI/CD pipeline?
2. Explain when you would use virtualization and how it works.
3. What are Web Vitals and how do you monitor them in production?
4. How does Next.js handle code splitting? What additional optimization can you apply?
5. When is `React.memo` counterproductive?
6. Design a lazy-loading strategy for a dashboard with 12 widgets.
