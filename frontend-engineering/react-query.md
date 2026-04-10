# React Query — Data Fetching, Caching, Mutations, Optimistic Updates

## Overview

TanStack React Query v5 is our standard for all server state management. This document covers production patterns specific to the banking GenAI platform.

## Query Key Factory

Every domain should have a structured query key factory. This ensures consistency and enables precise cache invalidation.

```tsx
// src/lib/api/queryKeys.ts
import type { AuditFilters, FeedbackFilters } from '@/types';

export const auditKeys = {
  all: ['audit'] as const,
  lists: () => [...auditKeys.all, 'list'] as const,
  list: (filters: AuditFilters) => [...auditKeys.lists(), filters] as const,
  detail: (id: string) => [...auditKeys.all, 'detail', id] as const,
  export: (filters: AuditFilters) => [...auditKeys.all, 'export', filters] as const,
};

export const feedbackKeys = {
  all: ['feedback'] as const,
  lists: () => [...feedbackKeys.all, 'list'] as const,
  list: (filters: FeedbackFilters) => [...feedbackKeys.lists(), filters] as const,
  summary: () => [...feedbackKeys.all, 'summary'] as const,
  reviewQueue: () => [...feedbackKeys.all, 'review-queue'] as const,
};

export const conversationKeys = {
  all: ['conversations'] as const,
  lists: () => [...conversationKeys.all, 'list'] as const,
  list: (userId: string) => [...conversationKeys.lists(), userId] as const,
  detail: (id: string) => [...conversationKeys.all, 'detail', id] as const,
  messages: (id: string) => [...conversationKeys.detail(id), 'messages'] as const,
};
```

## Banking-Optimized Query Configuration

```tsx
// src/lib/api/client.ts
import { QueryClient } from '@tanstack/react-query';

export const queryClient = new QueryClient({
  defaultOptions: {
    queries: {
      staleTime: 60_000,              // 1 min — balance between freshness and server load
      gcTime: 10 * 60 * 1000,         // 10 min garbage
      retry: (failureCount, error) => {
        if (error instanceof Response) {
          if (error.status === 401) return false;  // Don't retry auth failures
          if (error.status === 403) return false;  // Don't retry permission errors
          if (error.status === 404) return false;  // Resource doesn't exist
          if (error.status >= 500) return failureCount < 2;  // Server errors: 2 retries
        }
        return failureCount < 2;
      },
      retryDelay: (attemptIndex) => Math.min(1000 * 2 ** attemptIndex, 5000),
      refetchOnWindowFocus: false,    // Banking: don't refetch just because user alt-tabbed
      refetchOnReconnect: true,
      refetchOnMount: true,
    },
    mutations: {
      retry: false,  // NEVER auto-retry mutations in banking — idempotency is the backend's concern
    },
  },
});
```

## Infinite Queries for Audit Logs and Message Lists

```tsx
// src/hooks/useAuditLogs.ts
import { useInfiniteQuery, useQueryClient } from '@tanstack/react-query';
import { auditKeys } from '@/lib/api/queryKeys';
import type { AuditLog, AuditFilters } from '@/types';

interface AuditLogsPage {
  logs: AuditLog[];
  nextCursor: string | null;
}

export function useAuditLogs(filters: AuditFilters) {
  return useInfiniteQuery<AuditLogsPage>({
    queryKey: auditKeys.list(filters),
    queryFn: async ({ pageParam }) => {
      const params = new URLSearchParams({
        ...filters,
        cursor: pageParam ?? '',
        limit: '50',
      });

      const response = await fetch(`/api/audit?${params}`);
      if (!response.ok) throw new Error('Failed to fetch audit logs');
      return response.json();
    },
    initialPageParam: null as string | null,
    getNextPageParam: (lastPage) => lastPage.nextCursor,
    staleTime: 30_000,
  });
}

// Usage component with virtualization
import { useVirtualizer } from '@tanstack/react-virtual';

function AuditLogTable({ filters }: { filters: AuditFilters }) {
  const {
    data,
    fetchNextPage,
    hasNextPage,
    isFetchingNextPage,
  } = useAuditLogs(filters);

  const allLogs = data?.pages.flatMap(page => page.logs) ?? [];

  const parentRef = React.useRef<HTMLDivElement>(null);
  const virtualizer = useVirtualizer({
    count: allLogs.length,
    getScrollElement: () => parentRef.current,
    estimateSize: () => 48,
    overscan: 10,
  });

  return (
    <div ref={parentRef} className="h-96 overflow-auto">
      <div style={{ height: `${virtualizer.getTotalSize()}px`, position: 'relative' }}>
        {virtualizer.getVirtualItems().map((virtualRow) => {
          const log = allLogs[virtualRow.index];
          return (
            <div
              key={log.id}
              ref={virtualizer.measureElement}
              data-index={virtualRow.index}
              className="absolute left-0 top-0 w-full border-b px-4 py-2"
              style={{ transform: `translateY(${virtualRow.start}px)` }}
            >
              <AuditLogRow log={log} />
            </div>
          );
        })}
      </div>
      {hasNextPage && (
        <button
          onClick={() => fetchNextPage()}
          disabled={isFetchingNextPage}
          className="w-full p-3 text-sm text-muted-foreground"
        >
          {isFetchingNextPage ? 'Loading...' : 'Load more'}
        </button>
      )}
    </div>
  );
}
```

## Mutations with Optimistic Updates

### Feedback Submission (Low-Risk Optimistic)

```tsx
// src/hooks/useFeedback.ts
import { useMutation, useQueryClient } from '@tanstack/react-query';
import { feedbackKeys } from '@/lib/api/queryKeys';

interface SubmitFeedbackInput {
  messageId: string;
  rating: 'thumbs-up' | 'thumbs-down';
  comment?: string;
}

export function useSubmitFeedback() {
  const queryClient = useQueryClient();

  return useMutation({
    mutationFn: (input: SubmitFeedbackInput) =>
      fetch('/api/feedback', {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify(input),
      }).then(r => {
        if (!r.ok) throw new Error('Failed to submit feedback');
        return r.json();
      }),

    onMutate: async (input) => {
      // Cancel any outgoing feedback queries
      await queryClient.cancelQueries({ queryKey: feedbackKeys.summary() });

      // Snapshot current summary
      const previousSummary = queryClient.getQueryData(feedbackKeys.summary());

      // Optimistically update
      queryClient.setQueryData(feedbackKeys.summary(), (old: FeedbackSummary) => ({
        ...old,
        totalFeedbacks: (old?.totalFeedbacks ?? 0) + 1,
        thumbsUp: input.rating === 'thumbs-up'
          ? (old?.thumbsUp ?? 0) + 1
          : (old?.thumbsUp ?? 0),
        thumbsDown: input.rating === 'thumbs-down'
          ? (old?.thumbsDown ?? 0) + 1
          : (old?.thumbsDown ?? 0),
      }));

      return { previousSummary };
    },

    onError: (_err, _input, context) => {
      if (context?.previousSummary) {
        queryClient.setQueryData(feedbackKeys.summary(), context.previousSummary);
      }
    },

    onSettled: () => {
      queryClient.invalidateQueries({ queryKey: feedbackKeys.summary() });
    },
  });
}
```

### Policy Update (Conservative — No Optimistic)

For compliance-critical mutations, we do NOT use optimistic updates. Wait for server confirmation.

```tsx
// src/hooks/usePolicyUpdate.ts
import { useMutation, useQueryClient } from '@tanstack/react-query';
import { policyKeys } from '@/lib/api/queryKeys';

export function useUpdatePolicy() {
  const queryClient = useQueryClient();

  return useMutation({
    mutationFn: async ({ id, ...data }: PolicyUpdate) => {
      const response = await fetch(`/api/policies/${id}`, {
        method: 'PUT',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify(data),
      });
      if (!response.ok) {
        const error = await response.json();
        throw new Error(error.message ?? 'Failed to update policy');
      }
      return response.json();
    },

    // NO onMutate — compliance-critical, must confirm server-side
    onSuccess: (_data, variables) => {
      // Invalidate specific queries
      queryClient.invalidateQueries({ queryKey: policyKeys.detail(variables.id) });
      queryClient.invalidateQueries({ queryKey: policyKeys.lists() });
    },
  });
}
```

## Dependent Queries

When one query depends on data from another, React Query handles it elegantly.

```tsx
// src/hooks/useConversationWithMessages.ts
export function useConversationWithMessages(conversationId: string) {
  // First query: get conversation metadata
  const { data: conversation, isLoading: loadingConversation } = useQuery({
    queryKey: conversationKeys.detail(conversationId),
    queryFn: () => fetchConversation(conversationId),
  });

  // Second query: depends on first — only runs when conversation is loaded
  const { data: messages, isLoading: loadingMessages } = useQuery({
    queryKey: conversationKeys.messages(conversationId),
    queryFn: () => fetchMessages(conversationId),
    enabled: !!conversation,  // Only run when conversation is available
  });

  return {
    conversation,
    messages,
    isLoading: loadingConversation || loadingMessages,
  };
}
```

## Prefetching for Performance

```tsx
// Prefetch policy details when user hovers over a policy link
function PolicyLink({ policyId, children }: { policyId: string; children: React.ReactNode }) {
  const queryClient = useQueryClient();

  return (
    <Link
      href={`/compliance/policies/${policyId}`}
      onMouseEnter={() => {
        queryClient.prefetchQuery({
          queryKey: policyKeys.detail(policyId),
          queryFn: () => fetchPolicy(policyId),
          staleTime: 5 * 60 * 1000,
        });
      }}
    >
      {children}
    </Link>
  );
}

// Prefetch next page in audit logs
function AuditLogTable() {
  const { data, hasNextPage, fetchNextPage, queryKey } = useAuditLogs(filters);
  const queryClient = useQueryClient();

  const prefetchNextPage = () => {
    if (!hasNextPage) return;
    queryClient.prefetchInfiniteQuery({
      queryKey,
      queryFn: ... /* same as useInfiniteQuery */,
      pages: 1,
    });
  };

  return (
    <div onMouseLeave={prefetchNextPage}>
      {/* ... */}
    </div>
  );
}
```

## Query Cancellation

```tsx
// Abort in-flight requests when component unmounts
// React Query handles this automatically if you pass AbortSignal

async function fetchPolicies(signal: AbortSignal): Promise<Policy[]> {
  const response = await fetch('/api/policies', { signal });
  if (!response.ok) throw new Error('Failed to fetch policies');
  return response.json();
}

export function usePolicies() {
  return useQuery({
    queryKey: policyKeys.list({}),
    queryFn: ({ signal }) => fetchPolicies(signal),
  });
}
// When component unmounts, React Query automatically aborts the request
```

## Common Mistakes

### 1. Not Using Query Key Factories

```tsx
// ❌ BAD: Inconsistent keys
useQuery({ queryKey: ['audit'], ... });
useQuery({ queryKey: ['audit-logs'], ... });
useQuery({ queryKey: ['logs'], ... });
queryClient.invalidateQueries({ queryKey: ['audit'] }); // Only invalidates one!

// ✅ GOOD: Factory
useQuery({ queryKey: auditKeys.list({}), ... });
queryClient.invalidateQueries({ queryKey: auditKeys.lists() }); // Invalidates all audit list queries
```

### 2. Forgetting to Cancel Queries on Optimistic Updates

```tsx
// ❌ BAD: Race condition
onMutate: async (variables) => {
  queryClient.setQueryData(key, variables); // Old data might overwrite this
}

// ✅ GOOD: Cancel first
onMutate: async (variables) => {
  await queryClient.cancelQueries({ queryKey: key });  // Stop in-flight requests
  const previous = queryClient.getQueryData(key);
  queryClient.setQueryData(key, variables);
  return { previous };
}
```

### 3. Overly Broad Cache Invalidation

```tsx
// ❌ BAD: Nuke everything
queryClient.invalidateQueries();

// ✅ GOOD: Surgical invalidation
queryClient.invalidateQueries({ queryKey: policyKeys.detail(policyId) });
```

## Cross-References

- `./state-management.md` — Server vs client state classification
- `./streaming-responses.md` — Streaming response handling (SSE, not React Query)
- `./genai-chat-interfaces.md` — Conversation state management
- `./performance-optimization.md` — Prefetching and virtualization
- `./human-review-flows.md` — Review queue mutations
- `../backend-engineering/api-design/` — API design that supports efficient caching

## Interview Questions

1. Explain the query key factory pattern and why it matters.
2. How do you implement optimistic updates with rollback?
3. When would you NOT use optimistic updates in a banking application?
4. Design a paginated audit log table with infinite scroll.
5. How does React Query handle request cancellation?
6. What happens when two components use the same query key simultaneously?
