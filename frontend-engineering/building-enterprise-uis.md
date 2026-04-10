# Building Enterprise UIs — Complex Data Tables, Forms, Dashboards, Navigation

## Overview

Enterprise banking UIs require complex, data-dense interfaces that remain usable, accessible, and performant. This document covers patterns for the most common enterprise UI patterns.

## Complex Data Tables

### Audit Log Table with Server-Side Pagination, Sorting, and Filtering

```tsx
// src/components/audit/AuditLogDataTable.tsx
'use client';

import { useState } from 'react';
import { useQuery } from '@tanstack/react-query';
import { useVirtualizer } from '@tanstack/react-virtual';
import { auditKeys } from '@/lib/api/queryKeys';
import type { AuditLog, AuditFilters } from '@/types';

interface AuditLogDataTableProps {
  defaultFilters?: AuditFilters;
}

export function AuditLogDataTable({ defaultFilters }: AuditLogDataTableProps) {
  const [filters, setFilters] = useState<AuditFilters>({
    page: 1,
    pageSize: 50,
    sortBy: 'timestamp',
    sortOrder: 'desc',
    ...defaultFilters,
  });

  const { data, isLoading, isError } = useQuery({
    queryKey: auditKeys.list(filters),
    queryFn: () => fetchAuditLogs(filters),
    staleTime: 30_000,
  });

  const handleSort = (column: string) => {
    setFilters(prev => ({
      ...prev,
      sortBy: column,
      sortOrder: prev.sortBy === column && prev.sortOrder === 'asc' ? 'desc' : 'asc',
    }));
  };

  if (isLoading) return <TableSkeleton />;
  if (isError) return <ErrorFallback onRetry={() => queryClient.invalidateQueries({ queryKey: auditKeys.list(filters) })} />;

  return (
    <div className="space-y-4">
      {/* Filter controls */}
      <AuditLogFilters filters={filters} onFilterChange={setFilters} />

      {/* Data table */}
      <div className="border rounded-lg overflow-hidden">
        <table className="w-full text-sm">
          <caption className="sr-only">Audit logs</caption>
          <thead className="bg-muted">
            <tr>
              <SortableTh column="timestamp" current={filters} onSort={handleSort}>
                Timestamp
              </SortableTh>
              <SortableTh column="user" current={filters} onSort={handleSort}>
                User
              </SortableTh>
              <SortableTh column="action" current={filters} onSort={handleSort}>
                Action
              </SortableTh>
              <SortableTh column="resource" current={filters} onSort={handleSort}>
                Resource
              </SortableTh>
              <th scope="col" className="text-left p-3">Status</th>
            </tr>
          </thead>
          <tbody>
            {data?.logs.map(log => (
              <tr key={log.id} className="border-t">
                <td className="p-3 whitespace-nowrap">{log.timestamp.toLocaleString()}</td>
                <td className="p-3">{log.user}</td>
                <td className="p-3">{log.action}</td>
                <td className="p-3 max-w-xs truncate">{log.resource}</td>
                <td className="p-3"><StatusBadge status={log.status} /></td>
              </tr>
            ))}
          </tbody>
        </table>
      </div>

      {/* Pagination */}
      <Pagination
        currentPage={filters.page}
        totalPages={data?.totalPages ?? 1}
        onPageChange={(page) => setFilters(prev => ({ ...prev, page }))}
      />
    </div>
  );
}

function SortableTh({
  column,
  current,
  onSort,
  children,
}: {
  column: string;
  current: { sortBy: string; sortOrder: 'asc' | 'desc' };
  onSort: (column: string) => void;
  children: React.ReactNode;
}) {
  const isActive = current.sortBy === column;

  return (
    <th scope="col" className="text-left p-3">
      <button
        onClick={() => onSort(column)}
        className="flex items-center gap-1 font-medium hover:text-primary"
        aria-sort={isActive ? (current.sortOrder === 'asc' ? 'ascending' : 'descending') : 'none'}
      >
        {children}
        {isActive && (
          <span aria-hidden="true">{current.sortOrder === 'asc' ? '↑' : '↓'}</span>
        )}
      </button>
    </th>
  );
}
```

## Complex Forms

### Multi-Step Policy Review Form

```tsx
// src/components/forms/PolicyReviewForm.tsx
'use client';

import { useState } from 'react';
import { useForm, FormProvider } from 'react-hook-form';
import { zodResolver } from '@hookform/resolvers/zod';
import { z } from 'zod';
import { Button } from '@/components/ui/Button';
import { AccessibleInput } from '@/components/forms/AccessibleInput';

const policyReviewSchema = z.object({
  // Step 1: Policy Information
  policyId: z.string().min(1, 'Policy ID is required'),
  title: z.string().min(1, 'Title is required').max(200),
  category: z.enum(['aml', 'kyc', 'gdpr', 'pci-dss', 'basel']),
  effectiveDate: z.string().refine(val => !isNaN(Date.parse(val)), 'Invalid date'),

  // Step 2: Content
  summary: z.string().min(50, 'Summary must be at least 50 characters'),
  keyChanges: z.string().min(1, 'Key changes are required'),

  // Step 3: Approval
  reviewerName: z.string().min(1, 'Reviewer name is required'),
  reviewerRole: z.string().min(1, 'Reviewer role is required'),
  approved: z.boolean(),
  comments: z.string().optional(),
});

type PolicyReviewForm = z.infer<typeof policyReviewSchema>;

const STEPS = [
  { id: 'policy', title: 'Policy Information' },
  { id: 'content', title: 'Content Review' },
  { id: 'approval', title: 'Approval' },
];

export function PolicyReviewForm({ policy }: { policy: Policy }) {
  const [currentStep, setCurrentStep] = useState(0);

  const form = useForm<PolicyReviewForm>({
    resolver: zodResolver(policyReviewSchema),
    defaultValues: {
      policyId: policy.id,
      title: policy.title,
      category: policy.category,
      effectiveDate: policy.effectiveDate.toISOString().split('T')[0],
    },
    mode: 'onChange',
  });

  const { trigger, getValues, formState } = form;
  const isLastStep = currentStep === STEPS.length - 1;

  const handleNext = async () => {
    const fieldsToValidate: (keyof PolicyReviewForm)[] = getFieldsForStep(currentStep);
    const isValid = await trigger(fieldsToValidate);
    if (isValid) {
      setCurrentStep(s => Math.min(s + 1, STEPS.length - 1));
    }
  };

  const handlePrevious = () => {
    setCurrentStep(s => Math.max(s - 1, 0));
  };

  const onSubmit = async (data: PolicyReviewForm) => {
    await submitPolicyReview(data);
  };

  return (
    <FormProvider {...form}>
      <form onSubmit={form.handleSubmit(onSubmit)} className="space-y-6">
        {/* Step indicator */}
        <nav aria-label="Progress">
          <ol className="flex items-center space-x-4" role="list">
            {STEPS.map((step, index) => (
              <li key={step.id} className="flex items-center">
                <div
                  className={cn(
                    'w-8 h-8 rounded-full flex items-center justify-center text-sm font-medium',
                    index < currentStep && 'bg-primary text-primary-foreground',
                    index === currentStep && 'bg-primary text-primary-foreground',
                    index > currentStep && 'bg-muted text-muted-foreground',
                  )}
                  aria-current={index === currentStep ? 'step' : undefined}
                >
                  {index < currentStep ? '✓' : index + 1}
                </div>
                <span className="ml-2 text-sm hidden sm:inline">{step.title}</span>
              </li>
            ))}
          </ol>
        </nav>

        {/* Step content */}
        <div className="space-y-4">
          {currentStep === 0 && <PolicyInfoStep />}
          {currentStep === 1 && <ContentStep />}
          {currentStep === 2 && <ApprovalStep />}
        </div>

        {/* Navigation */}
        <div className="flex justify-between pt-4">
          <Button
            type="button"
            variant="secondary"
            onClick={handlePrevious}
            disabled={currentStep === 0}
          >
            Previous
          </Button>

          {isLastStep ? (
            <Button type="submit" isLoading={formState.isSubmitting}>
              Submit Review
            </Button>
          ) : (
            <Button type="button" onClick={handleNext}>
              Next
            </Button>
          )}
        </div>
      </form>
    </FormProvider>
  );
}

function getFieldsForStep(step: number): (keyof PolicyReviewForm)[] {
  switch (step) {
    case 0:
      return ['policyId', 'title', 'category', 'effectiveDate'];
    case 1:
      return ['summary', 'keyChanges'];
    case 2:
      return ['reviewerName', 'reviewerRole', 'approved'];
    default:
      return [];
  }
}
```

## Dashboards

### Compliance Dashboard with Grid Layout

```tsx
// src/app/(dashboard)/compliance/page.tsx
import { Suspense } from 'react';
import { ErrorBoundary } from '@/components/shared/ErrorBoundary';

export default async function ComplianceDashboard() {
  const data = await fetchComplianceDashboardData();

  return (
    <div className="space-y-6">
      <div className="flex items-center justify-between">
        <h1 className="text-2xl font-bold">Compliance Dashboard</h1>
        <ExportButton reportType="compliance" />
      </div>

      {/* Summary cards */}
      <div className="grid grid-cols-1 sm:grid-cols-2 lg:grid-cols-4 gap-4">
        <SummaryCard
          title="Overall Compliance"
          value={`${data.overallCompliance}%`}
          trend={data.complianceTrend}
          icon={<ShieldIcon />}
        />
        <SummaryCard
          title="Active Violations"
          value={data.activeViolations}
          trend={data.violationTrend}
          icon={<AlertIcon />}
          variant="destructive"
        />
        <SummaryCard
          title="Pending Reviews"
          value={data.pendingReviews}
          icon={<ClockIcon />}
        />
        <SummaryCard
          title="Last Audit"
          value={data.lastAuditDate.toLocaleDateString()}
          icon={<CalendarIcon />}
        />
      </div>

      {/* Main content grid */}
      <div className="grid grid-cols-1 lg:grid-cols-2 gap-6">
        <ErrorBoundary name="compliance-by-region">
          <Suspense fallback={<ChartSkeleton />}>
            <ComplianceByRegionChart data={data.complianceByRegion} />
          </Suspense>
        </ErrorBoundary>

        <ErrorBoundary name="recent-violations">
          <Suspense fallback={<TableSkeleton />}>
            <RecentViolationsTable violations={data.recentViolations} />
          </Suspense>
        </ErrorBoundary>
      </div>

      {/* Full-width section */}
      <ErrorBoundary name="audit-trail">
        <Suspense fallback={<TableSkeleton />}>
          <AuditTrailTable />
        </Suspense>
      </ErrorBoundary>
    </div>
  );
}
```

## Navigation Patterns

### Breadcrumb Navigation

```tsx
// src/components/layout/Breadcrumbs.tsx
interface BreadcrumbItem {
  label: string;
  href?: string;
}

interface BreadcrumbsProps {
  items: BreadcrumbItem[];
}

export function Breadcrumbs({ items }: BreadcrumbsProps) {
  return (
    <nav aria-label="Breadcrumb" className="mb-6">
      <ol className="flex items-center space-x-2 text-sm">
        <li>
          <a href="/dashboard" className="text-muted-foreground hover:text-primary">
            Dashboard
          </a>
        </li>
        {items.map((item, index) => (
          <li key={item.href ?? item.label} className="flex items-center space-x-2">
            <ChevronRightIcon className="h-4 w-4 text-muted-foreground" aria-hidden="true" />
            {item.href ? (
              <a
                href={item.href}
                className={cn(
                  'hover:text-primary',
                  index === items.length - 1 ? 'text-primary font-medium' : 'text-muted-foreground',
                )}
                aria-current={index === items.length - 1 ? 'page' : undefined}
              >
                {item.label}
              </a>
            ) : (
              <span className="text-primary font-medium" aria-current="page">
                {item.label}
              </span>
            )}
          </li>
        ))}
      </ol>
    </nav>
  );
}
```

## Common Mistakes

### 1. Client-Side Pagination for Large Datasets

```tsx
// ❌ BAD: Loading all 10,000 audit logs to the client
const allLogs = await fetch('/api/audit?limit=10000');

// ✅ GOOD: Server-side pagination
const { data } = useQuery({
  queryKey: auditKeys.list({ page, pageSize: 50 }),
  queryFn: () => fetchAuditLogs({ page, pageSize: 50 }),
});
```

### 2. Uncontrolled Forms for Complex Data

```tsx
// ❌ BAD: Managing form state manually
const [title, setTitle] = useState('');
const [category, setCategory] = useState('');
// ... 20 more useState calls

// ✅ GOOD: React Hook Form
const form = useForm({ resolver: zodResolver(schema) });
```

### 3. Missing Loading States

```tsx
// ❌ BAD: Blank page while loading
if (loading) return null;

// ✅ GOOD: Skeleton UI
if (loading) return <DashboardSkeleton />;
```

## Cross-References

- `./react-query.md` — Server state and pagination
- `./design-systems.md` — Component library for tables, forms, cards
- `./accessibility.md` — Accessible tables and forms
- `./performance-optimization.md` — Virtualization for large tables
- `./building-enterprise-uis.md` — This document

## Interview Questions

1. Design a data table with server-side pagination, sorting, and filtering.
2. How do you build a multi-step form with validation at each step?
3. What patterns do you use for dashboard layouts with multiple data sources?
4. How do you handle loading states for a page with 6 independent data fetches?
5. Design an accessible breadcrumb navigation component.
