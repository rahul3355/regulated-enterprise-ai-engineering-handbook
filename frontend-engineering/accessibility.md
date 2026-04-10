# Accessibility — WCAG 2.1 AA, Screen Readers, Keyboard Navigation, ARIA Patterns

## Overview

Accessibility is not optional. In banking, it is a legal requirement and a moral obligation. All frontend code must meet WCAG 2.1 AA standards. This document defines the requirements and provides patterns for common UI components in our GenAI platform.

## Legal and Regulatory Context

- **ADA (Americans with Disabilities Act)** — Applies to digital banking services in the US
- **Section 508** — Federal accessibility requirements
- **EN 301 549** — European accessibility standard for ICT
- **AODA (Ontario)** — Accessibility standards for Canadian operations
- **Internal Policy** — Our bank requires WCAG 2.1 AA as a minimum for all customer-facing and internal applications

## WCAG 2.1 AA Checklist

### Perceivable

- [ ] All images have meaningful alt text (or `alt=""` for decorative images)
- [ ] Color is not the only means of conveying information
- [ ] Contrast ratio meets 4.5:1 for normal text, 3:1 for large text
- [ ] Content reflows at 200% zoom without horizontal scrolling
- [ ] Video content has captions
- [ ] Form inputs have associated labels

### Operable

- [ ] All functionality is keyboard accessible
- [ ] No keyboard traps
- [ ] Focus indicators are visible and meet 3:1 contrast ratio
- [ ] Skip navigation link is present
- [ ] Time limits can be extended or disabled
- [ ] No content flashes more than 3 times per second

### Understandable

- [ ] Page language is set (`<html lang="en">`)
- [ ] Form errors are identified and described in text
- [ ] Navigation is consistent across pages
- [ ] Input purpose is programmatically determinable

### Robust

- [ ] Valid HTML (no nesting violations)
- [ ] ARIA roles, states, and properties are used correctly
- [ ] Status messages are announced to assistive technology

## Skip Navigation

Every page must have a skip-to-content link.

```tsx
// src/components/layout/SkipToMainContent.tsx
export function SkipToMainContent() {
  return (
    <a
      href="#main-content"
      className="sr-only focus:not-sr-only focus:absolute focus:top-4 focus:left-4 focus:z-50 focus:px-4 focus:py-2 focus:bg-primary focus:text-primary-foreground focus:rounded"
    >
      Skip to main content
    </a>
  );
}

// Usage in root layout
export default function RootLayout({ children }) {
  return (
    <html lang="en">
      <body>
        <SkipToMainContent />
        <Header />
        <main id="main-content">{children}</main>
      </body>
    </html>
  );
}
```

## Focus Management

### Visible Focus Indicators

```css
/* src/styles/globals.css */
/* Never remove outline without providing an alternative */
*:focus-visible {
  outline: 2px solid hsl(var(--primary));
  outline-offset: 2px;
  border-radius: 2px;
}

/* For interactive elements that need a more prominent focus ring */
button:focus-visible,
a:focus-visible,
[tabindex]:focus-visible {
  outline: 2px solid hsl(var(--primary));
  outline-offset: 2px;
  box-shadow: 0 0 0 4px hsl(var(--primary) / 0.2);
}

/* NEVER do this: */
/* *:focus { outline: none; } */
```

### Focus Trapping in Modals

```tsx
// src/components/ui/Dialog.tsx
import { useEffect, useRef } from 'react';
import { createPortal } from 'react-dom';

interface DialogProps {
  isOpen: boolean;
  onClose: () => void;
  title: string;
  children: React.ReactNode;
}

export function Dialog({ isOpen, onClose, title, children }: DialogProps) {
  const dialogRef = useRef<HTMLDivElement>(null);
  const previousFocusRef = useRef<HTMLElement | null>(null);

  useEffect(() => {
    if (isOpen) {
      // Store current focus
      previousFocusRef.current = document.activeElement as HTMLElement;
      // Focus the dialog
      dialogRef.current?.focus();
      // Trap focus inside dialog
      const handleKeyDown = (e: KeyboardEvent) => {
        if (e.key === 'Tab') {
          const focusableElements = dialogRef.current?.querySelectorAll(
            'button, [href], input, select, textarea, [tabindex]:not([tabindex="-1"])',
          );
          if (!focusableElements?.length) return;

          const first = focusableElements[0] as HTMLElement;
          const last = focusableElements[focusableElements.length - 1] as HTMLElement;

          if (e.shiftKey && document.activeElement === first) {
            e.preventDefault();
            last.focus();
          } else if (!e.shiftKey && document.activeElement === last) {
            e.preventDefault();
            first.focus();
          }
        }
        if (e.key === 'Escape') {
          onClose();
        }
      };

      document.addEventListener('keydown', handleKeyDown);
      return () => document.removeEventListener('keydown', handleKeyDown);
    }

    // Restore focus on close
    if (!isOpen && previousFocusRef.current) {
      previousFocusRef.current.focus();
    }
  }, [isOpen, onClose]);

  if (!isOpen) return null;

  return createPortal(
    <div
      className="fixed inset-0 z-50 flex items-center justify-center bg-black/50"
      role="dialog"
      aria-modal="true"
      aria-labelledby="dialog-title"
    >
      <div
        ref={dialogRef}
        tabIndex={-1}
        className="bg-background rounded-lg shadow-lg max-w-lg w-full mx-4 p-6"
      >
        <h2 id="dialog-title" className="text-lg font-semibold">{title}</h2>
        <div className="mt-4">{children}</div>
        <button
          onClick={onClose}
          className="mt-4 px-4 py-2 rounded bg-primary text-primary-foreground"
        >
          Close
        </button>
      </div>
    </div>,
    document.body,
  );
}
```

## Form Accessibility

```tsx
// src/components/forms/AccessibleInput.tsx
import { useId } from 'react';

interface AccessibleInputProps extends React.InputHTMLAttributes<HTMLInputElement> {
  label: string;
  error?: string;
  required?: boolean;
  description?: string;
}

export function AccessibleInput({
  label,
  error,
  required,
  description,
  id,
  ...props
}: AccessibleInputProps) {
  const generatedId = useId();
  const inputId = id ?? generatedId;
  const descriptionId = `${inputId}-description`;
  const errorId = `${inputId}-error`;

  return (
    <div className="space-y-2">
      <label htmlFor={inputId} className="text-sm font-medium">
        {label}
        {required && (
          <span className="text-destructive ml-1" aria-hidden="true">
            *
          </span>
        )}
      </label>

      {description && (
        <p id={descriptionId} className="text-sm text-muted-foreground">
          {description}
        </p>
      )}

      <input
        id={inputId}
        aria-required={required}
        aria-invalid={!!error}
        aria-describedby={
          [description && descriptionId, error && errorId].filter(Boolean).join(' ') || undefined
        }
        aria-errormessage={error ? errorId : undefined}
        className="flex h-10 w-full rounded-md border border-input bg-background px-3 py-2 text-sm"
        {...props}
      />

      {error && (
        <p id={errorId} className="text-sm text-destructive" role="alert" aria-live="polite">
          {error}
        </p>
      )}
    </div>
  );
}
```

## ARIA Patterns for Common Components

### Status Messages (Live Regions)

```tsx
// ✅ GOOD: Announce status changes to screen readers
function SubmissionStatus({ status }: { status: 'idle' | 'submitting' | 'success' | 'error' }) {
  return (
    <div role="status" aria-live="polite">
      {status === 'submitting' && <p>Submitting your feedback...</p>}
      {status === 'success' && <p>Feedback submitted successfully.</p>}
      {status === 'error' && <p>Failed to submit feedback. Please try again.</p>}
    </div>
  );
}

// For important alerts that should interrupt
function CriticalAlert({ message }: { message: string }) {
  return (
    <div role="alert" aria-live="assertive">
      {message}
    </div>
  );
}
```

### Tab Panel

```tsx
// src/components/ui/Tabs.tsx
import { useState, useId, createContext, useContext } from 'react';

interface TabsContextValue {
  activeTab: string;
  setActiveTab: (id: string) => void;
  prefix: string;
}

const TabsContext = createContext<TabsContextValue>({
  activeTab: '',
  setActiveTab: () => {},
  prefix: '',
});

function Tabs({ children, defaultValue }: { children: React.ReactNode; defaultValue: string }) {
  const [activeTab, setActiveTab] = useState(defaultValue);
  const prefix = useId();

  return (
    <TabsContext.Provider value={{ activeTab, setActiveTab, prefix }}>
      <div role="tablist">{children}</div>
    </TabsContext.Provider>
  );
}

function Tab({ value, children }: { value: string; children: React.ReactNode }) {
  const { activeTab, setActiveTab, prefix } = useContext(TabsContext);
  const tabId = `${prefix}-tab-${value}`;
  const panelId = `${prefix}-panel-${value}`;

  return (
    <>
      <button
        role="tab"
        id={tabId}
        aria-selected={activeTab === value}
        aria-controls={panelId}
        tabIndex={activeTab === value ? 0 : -1}
        onClick={() => setActiveTab(value)}
        className={cn(
          'px-4 py-2 text-sm border-b-2 transition-colors',
          activeTab === value
            ? 'border-primary text-primary'
            : 'border-transparent text-muted-foreground',
        )}
      >
        {children}
      </button>
      {/* TabPanel would be rendered as a sibling */}
    </>
  );
}

Tabs.Tab = Tab;
```

### Compliance Data Table

```tsx
// src/components/compliance/ComplianceTable.tsx
import type { ComplianceReport } from '@/types';

interface ComplianceTableProps {
  reports: ComplianceReport[];
}

export function ComplianceTable({ reports }: ComplianceTableProps) {
  return (
    <div className="overflow-x-auto">
      <table className="w-full text-sm">
        <caption className="text-left sr-only">
          Compliance reports by region and status
        </caption>
        <thead>
          <tr className="border-b">
            <th scope="col" className="text-left p-3">Region</th>
            <th scope="col" className="text-left p-3">Compliant</th>
            <th scope="col" className="text-left p-3">Pending Review</th>
            <th scope="col" className="text-left p-3">Violations</th>
            <th scope="col" className="text-left p-3">Last Audit Date</th>
          </tr>
        </thead>
        <tbody>
          {reports.map((report) => (
            <tr key={report.region} className="border-b">
              <td className="p-3 font-medium">{report.region}</td>
              <td className="p-3">
                <span className="inline-flex items-center gap-1 text-green-700">
                  <CheckIcon className="h-4 w-4" aria-hidden="true" />
                  {report.compliant}
                </span>
              </td>
              <td className="p-3">{report.pendingReview}</td>
              <td className="p-3">
                {report.violations > 0 ? (
                  <span className="inline-flex items-center gap-1 text-destructive">
                    <AlertIcon className="h-4 w-4" aria-hidden="true" />
                    {report.violations}
                  </span>
                ) : (
                  '0'
                )}
              </td>
              <td className="p-3">{report.lastAuditDate.toLocaleDateString()}</td>
            </tr>
          ))}
        </tbody>
      </table>
    </div>
  );
}
```

## AI Chat Accessibility

```tsx
// src/components/chat/AccessibleChat.tsx
export function ChatInterface() {
  const { messages, isStreaming, sendMessage } = useConversation();

  return (
    <div className="flex flex-col h-full">
      <div
        role="log"
        aria-live="polite"
        aria-label="Chat messages"
        className="flex-1 overflow-auto p-4 space-y-4"
      >
        {messages.map((message) => (
          <MessageBubble key={message.id} variant={message.role}>
            <MessageContent content={message.content} />
            <time className="text-xs text-muted-foreground">
              {message.timestamp.toLocaleTimeString()}
            </time>
          </MessageBubble>
        ))}
        {isStreaming && (
          <div role="status" aria-live="polite" className="text-sm text-muted-foreground">
            <span className="sr-only">AI is generating a response</span>
            <div className="flex gap-1" aria-hidden="true">
              <span className="w-2 h-2 bg-muted-foreground rounded-full animate-bounce" style={{ animationDelay: '0ms' }} />
              <span className="w-2 h-2 bg-muted-foreground rounded-full animate-bounce" style={{ animationDelay: '150ms' }} />
              <span className="w-2 h-2 bg-muted-foreground rounded-full animate-bounce" style={{ animationDelay: '300ms' }} />
            </div>
          </div>
        )}
      </div>

      <form
        onSubmit={(e) => { e.preventDefault(); sendMessage(input); }}
        className="border-t p-4"
        aria-label="Send a message"
      >
        <label htmlFor="chat-input" className="sr-only">
          Type your message
        </label>
        <div className="flex gap-2">
          <input
            id="chat-input"
            type="text"
            value={input}
            onChange={(e) => setInput(e.target.value)}
            placeholder="Ask me anything..."
            className="flex-1 rounded-md border px-3 py-2 text-sm"
            aria-describedby="chat-input-help"
          />
          <button type="submit" disabled={isStreaming || !input.trim()}>
            {isStreaming ? 'Generating...' : 'Send'}
          </button>
        </div>
        <p id="chat-input-help" className="text-xs text-muted-foreground mt-1">
          Press Enter to send. Shift+Enter for a new line.
        </p>
      </form>
    </div>
  );
}
```

## Testing Accessibility

```bash
# Automated accessibility testing with axe-core
npm install --save-dev @axe-core/playwright

# Playwright config
// tests/e2e/accessibility.spec.ts
import { test, expect } from '@playwright/test';
import AxeBuilder from '@axe-core/playwright';

test('chat page has no accessibility violations', async ({ page }) => {
  await page.goto('/dashboard/chat');

  const accessibilityScanResults = await new AxeBuilder({ page })
    .withTags(['wcag2a', 'wcag2aa'])
    .analyze();

  expect(accessibilityScanResults.violations).toEqual([]);
});

test('compliance dashboard has no accessibility violations', async ({ page }) => {
  await page.goto('/dashboard/compliance');

  const accessibilityScanResults = await new AxeBuilder({ page })
    .withTags(['wcag2a', 'wcag2aa'])
    .analyze();

  expect(accessibilityScanResults.violations).toEqual([]);
});
```

## Common Mistakes

### 1. Using Color Alone

```tsx
// ❌ BAD: Only color indicates status
<span className={status === 'error' ? 'text-red-500' : 'text-green-500'}>
  {status}
</span>

// ✅ GOOD: Icon + text + color
<span className={cn('inline-flex items-center gap-1', status === 'error' ? 'text-destructive' : 'text-green-700')}>
  {status === 'error' ? <XIcon className="h-4 w-4" aria-hidden="true" /> : <CheckIcon className="h-4 w-4" aria-hidden="true" />}
  {status === 'error' ? 'Failed' : 'Passed'}
</span>
```

### 2. Meaningless Link Text

```tsx
// ❌ BAD
<a href="/policy-123">Click here</a>

// ✅ GOOD
<a href="/policy-123">View Anti-Money Laundering Policy</a>
```

### 3. Missing Form Labels

```tsx
// ❌ BAD
<input placeholder="Search policies..." />

// ✅ GOOD
<label htmlFor="policy-search" className="sr-only">Search policies</label>
<input id="policy-search" type="search" placeholder="Search policies..." />
```

## Cross-References

- `./design-systems.md` — Accessible component library patterns
- `./genai-chat-interfaces.md` — Chat accessibility requirements
- `./building-enterprise-uis.md` — Accessible data tables
- `./error-boundaries.md` — Accessible error messages
- `../testing-and-quality/` — Accessibility testing strategies

## Interview Questions

1. What are the four principles of WCAG? Give a banking UI example for each.
2. How do you implement focus trapping in a modal dialog?
3. Explain the difference between `role="status"` and `role="alert"`.
4. How do you make a data table accessible to screen reader users?
5. What is the minimum contrast ratio for WCAG 2.1 AA?
6. Design an accessible form with validation errors.
