# Frontend Testing — Jest, React Testing Library, Playwright, Visual Regression

## Overview

Testing is not optional. In a banking application, bugs can lead to compliance violations, data leaks, and financial loss. This document defines our testing strategy, tooling, and patterns.

## Testing Pyramid

```
                    ┌─────────────────┐
                    │   E2E Tests     │  Playwright — critical user journeys
                    │   (Few, Slow)   │  Chat flows, auth, review workflows
                    ├─────────────────┤
                    │ Integration     │  RTL — component interactions
                    │   Tests         │  Forms, dialogs, data tables
                    ├─────────────────┤
                    │ Unit Tests      │  Jest — pure functions, hooks, utils
                    │   (Many, Fast)  │  Sanitization, validation, formatting
                    └─────────────────┘
```

**Coverage Targets:**
- Unit tests: > 80% line coverage
- Integration tests: All user-facing components
- E2E tests: All critical user journeys
- Accessibility tests: All pages (automated axe-core scans)

## Unit Tests — Jest + React Testing Library

### Testing Utility Functions

```tsx
// src/lib/security/__tests__/sanitizeMarkdown.test.ts
import { sanitizeAndRenderMarkdown } from '../sanitizeMarkdown';

describe('sanitizeAndRenderMarkdown', () => {
  it('renders safe markdown to HTML', () => {
    const result = sanitizeAndRenderMarkdown('**Hello** world');
    expect(result).toContain('<strong>Hello</strong>');
    expect(result).toContain('world');
  });

  it('removes script tags', () => {
    const input = 'Hello <script>alert("xss")</script> world';
    const result = sanitizeAndRenderMarkdown(input);
    expect(result).not.toContain('<script>');
    expect(result).not.toContain('alert');
  });

  it('removes event handlers', () => {
    const input = '<img src=x onerror="alert(1)">';
    const result = sanitizeAndRenderMarkdown(input);
    expect(result).not.toContain('onerror');
    expect(result).not.toContain('alert');
  });

  it('removes javascript: URIs', () => {
    const input = '[Click](javascript:alert(1))';
    const result = sanitizeAndRenderMarkdown(input);
    expect(result).not.toContain('javascript:');
  });

  it('removes iframe tags', () => {
    const input = '<iframe src="https://evil.com"></iframe>';
    const result = sanitizeAndRenderMarkdown(input);
    expect(result).not.toContain('iframe');
  });

  it('preserves allowed formatting', () => {
    const input = `# Title

**Bold** and *italic* text.

- List item 1
- List item 2

\`\`\`python
print("hello")
\`\`\`

> Blockquote
`;
    const result = sanitizeAndRenderMarkdown(input);
    expect(result).toContain('<h1>');
    expect(result).toContain('<strong>');
    expect(result).toContain('<em>');
    expect(result).toContain('<ul>');
    expect(result).toContain('<pre>');
    expect(result).toContain('<blockquote>');
  });
});
```

### Testing Custom Hooks

```tsx
// src/hooks/__tests__/useChatStream.test.ts
import { renderHook, act, waitFor } from '@testing-library/react';
import { useChatStream } from '../useChatStream';

// Mock fetch
global.fetch = vi.fn();

describe('useChatStream', () => {
  beforeEach(() => {
    vi.resetAllMocks();
  });

  it('sends a message and receives streaming response', async () => {
    // Mock SSE stream
    const mockStream = new ReadableStream({
      start(controller) {
        controller.enqueue(new TextEncoder().encode('data: {"delta": {"content": "Hello"}}\n'));
        controller.enqueue(new TextEncoder().encode('data: {"delta": {"content": " world"}}\n'));
        controller.enqueue(new TextEncoder().encode('data: [DONE]\n'));
        controller.close();
      },
    });

    (global.fetch as Mock).mockResolvedValue({
      ok: true,
      body: mockStream,
    });

    const { result } = renderHook(() => useChatStream());

    await act(async () => {
      result.current.sendMessage('Hi');
    });

    // Wait for streaming to complete
    await waitFor(() => {
      expect(result.current.isStreaming).toBe(false);
    });

    expect(result.current.messages).toHaveLength(2);
    expect(result.current.messages[1].content).toBe('Hello world');
  });

  it('handles streaming errors', async () => {
    (global.fetch as Mock).mockResolvedValue({
      ok: false,
      status: 500,
    });

    const { result } = renderHook(() => useChatStream());

    await act(async () => {
      result.current.sendMessage('Hi');
    });

    await waitFor(() => {
      expect(result.current.error).not.toBeNull();
    });

    expect(result.current.isStreaming).toBe(false);
  });

  it('supports stopping streaming', async () => {
    // Mock a slow stream
    const mockStream = new ReadableStream({
      start(controller) {
        setTimeout(() => {
          controller.enqueue(new TextEncoder().encode('data: {"delta": {"content": "A"}}\n'));
        }, 100);
      },
    });

    (global.fetch as Mock).mockResolvedValue({
      ok: true,
      body: mockStream,
    });

    const { result } = renderHook(() => useChatStream());

    await act(async () => {
      result.current.sendMessage('Hi');
    });

    // Stop before stream completes
    act(() => {
      result.current.stopStreaming();
    });

    expect(result.current.isStreaming).toBe(false);
  });
});
```

### Testing Components

```tsx
// src/components/chat/__tests__/ChatInput.test.tsx
import { render, screen, fireEvent } from '@testing-library/react';
import { ChatInput } from '../ChatInput';

describe('ChatInput', () => {
  it('renders a textarea and send button', () => {
    render(
      <ChatInput onSend={() => {}} onStop={() => {}} isStreaming={false} />,
    );

    expect(screen.getByRole('textbox', { name: /send a message/i })).toBeInTheDocument();
    expect(screen.getByRole('button', { name: /send message/i })).toBeInTheDocument();
  });

  it('calls onSend when the send button is clicked', () => {
    const onSend = vi.fn();
    render(
      <ChatInput onSend={onSend} onStop={() => {}} isStreaming={false} />,
    );

    const textarea = screen.getByRole('textbox');
    fireEvent.change(textarea, { target: { value: 'Hello' } });

    const sendButton = screen.getByRole('button', { name: /send message/i });
    fireEvent.click(sendButton);

    expect(onSend).toHaveBeenCalledWith('Hello');
  });

  it('calls onSend when Enter is pressed', () => {
    const onSend = vi.fn();
    render(
      <ChatInput onSend={onSend} onStop={() => {}} isStreaming={false} />,
    );

    const textarea = screen.getByRole('textbox');
    fireEvent.change(textarea, { target: { value: 'Hello' } });
    fireEvent.keyDown(textarea, { key: 'Enter' });

    expect(onSend).toHaveBeenCalledWith('Hello');
  });

  it('does not send when input is empty', () => {
    const onSend = vi.fn();
    render(
      <ChatInput onSend={onSend} onStop={() => {}} isStreaming={false} />,
    );

    const sendButton = screen.getByRole('button', { name: /send message/i });
    fireEvent.click(sendButton);

    expect(onSend).not.toHaveBeenCalled();
  });

  it('shows stop button when streaming', () => {
    render(
      <ChatInput onSend={() => {}} onStop={() => {}} isStreaming={true} />,
    );

    expect(screen.getByRole('button', { name: /stop generating/i })).toBeInTheDocument();
    expect(screen.queryByRole('button', { name: /send message/i })).not.toBeInTheDocument();
  });

  it('creates a new line on Shift+Enter', () => {
    const onSend = vi.fn();
    render(
      <ChatInput onSend={onSend} onStop={() => {}} isStreaming={false} />,
    );

    const textarea = screen.getByRole('textbox');
    fireEvent.change(textarea, { target: { value: 'Line 1' } });
    fireEvent.keyDown(textarea, { key: 'Enter', shiftKey: true });

    expect(onSend).not.toHaveBeenCalled();
    expect(textarea).toHaveValue('Line 1');
  });

  it('has proper accessibility labels', () => {
    render(
      <ChatInput onSend={() => {}} onStop={() => {}} isStreaming={false} />,
    );

    const textbox = screen.getByRole('textbox');
    expect(textbox).toHaveAccessibleDescription();
  });
});
```

## Integration Tests — Complex Components

```tsx
// src/components/forms/__tests__/PolicyReviewForm.test.tsx
import { render, screen, fireEvent, waitFor } from '@testing-library/react';
import { PolicyReviewForm } from '../PolicyReviewForm';

// Mock react-hook-form dependencies
vi.mock('@hookform/resolvers/zod', () => ({ zodResolver: vi.fn(() => () => ({ values: {}, errors: {} })) }));

describe('PolicyReviewForm', () => {
  const mockPolicy = {
    id: 'policy-1',
    title: 'AML Policy v3.2',
    category: 'aml',
    effectiveDate: new Date('2024-01-01'),
  };

  it('renders the first step by default', () => {
    render(<PolicyReviewForm policy={mockPolicy} />);

    expect(screen.getByText('Policy Information')).toBeInTheDocument();
    expect(screen.getByLabelText(/title/i)).toHaveValue('AML Policy v3.2');
  });

  it('advances to the next step when valid data is entered', async () => {
    render(<PolicyReviewForm policy={mockPolicy} />);

    // Fill in step 1 fields
    fireEvent.change(screen.getByLabelText(/title/i), { target: { value: 'Updated Title' } });

    // Click Next
    fireEvent.click(screen.getByRole('button', { name: /next/i }));

    await waitFor(() => {
      expect(screen.getByText('Content Review')).toBeInTheDocument();
    });
  });

  it('shows validation errors for required fields', async () => {
    render(<PolicyReviewForm policy={mockPolicy} />);

    // Clear required field
    fireEvent.change(screen.getByLabelText(/title/i), { target: { value: '' } });

    // Try to advance
    fireEvent.click(screen.getByRole('button', { name: /next/i }));

    await waitFor(() => {
      expect(screen.getByRole('alert')).toHaveTextContent(/title is required/i);
    });
  });

  it('submits the form on the last step', async () => {
    render(<PolicyReviewForm policy={mockPolicy} />);

    // Navigate through all steps...
    // Fill final step fields
    fireEvent.change(screen.getByLabelText(/reviewer name/i), { target: { value: 'John Doe' } });

    // Click submit
    fireEvent.click(screen.getByRole('button', { name: /submit review/i }));

    await waitFor(() => {
      // Should redirect or show success
    });
  });
});
```

## E2E Tests — Playwright

```tsx
// tests/e2e/chat.spec.ts
import { test, expect } from '@playwright/test';

test.describe('GenAI Chat', () => {
  test.beforeEach(async ({ page }) => {
    // Authenticate via API
    await page.request.post('/api/auth/login', {
      data: {
        username: process.env.E2E_TEST_USER,
        password: process.env.E2E_TEST_PASSWORD,
      },
    });
    await page.goto('/dashboard/chat');
  });

  test('user can send a message and receive a response', async ({ page }) => {
    // Verify welcome message
    await expect(page.getByText('How can I help you today?')).toBeVisible();

    // Type and send a message
    const input = page.getByRole('textbox', { name: /send a message/i });
    await input.fill('What is our AML policy?');
    await input.press('Enter');

    // User message appears
    await expect(page.getByText('What is our AML policy?')).toBeVisible();

    // Assistant response appears (with streaming)
    await expect(page.getByRole('log')).toContainText(/.*loading|.*thinking|.*[a-zA-Z]{5,}.*/);

    // Wait for response to complete
    await expect(page.getByText('Stop')).not.toBeVisible();
  });

  test('user can stop a streaming response', async ({ page }) => {
    const input = page.getByRole('textbox', { name: /send a message/i });
    await input.fill('Write a long essay about banking regulations');
    await input.press('Enter');

    // Wait for streaming to start
    await expect(page.getByRole('button', { name: /stop generating/i })).toBeVisible();

    // Click stop
    await page.getByRole('button', { name: /stop generating/i }).click();

    // Streaming should stop
    await expect(page.getByRole('button', { name: /stop generating/i })).not.toBeVisible();
  });

  test('user can provide feedback on a response', async ({ page }) => {
    // Send message and wait for response...
    const input = page.getByRole('textbox', { name: /send a message/i });
    await input.fill('Test question');
    await input.press('Enter');
    await expect(page.getByText('Test question')).toBeVisible();

    // Wait for response, then click thumbs up
    await expect(page.getByRole('button', { name: /this response was helpful/i })).toBeVisible();
    await page.getByRole('button', { name: /this response was helpful/i }).click();

    // Confirmation appears
    await expect(page.getByText('Thank you for your feedback')).toBeVisible();
  });

  test('chat page is accessible', async ({ page }) => {
    await page.goto('/dashboard/chat');

    // Check for skip navigation
    await expect(page.getByRole('link', { name: /skip to main content/i })).toBeVisible();

    // Check for chat input label
    await expect(page.getByRole('textbox', { name: /send a message/i })).toBeVisible();

    // Check heading
    await expect(page.getByRole('heading', { name: /how can i help/i })).toBeVisible();
  });
});
```

## Visual Regression Tests

```tsx
// tests/e2e/visual.spec.ts
import { test, expect } from '@playwright/test';

test.describe('Visual Regression', () => {
  test('compliance dashboard matches snapshot', async ({ page }) => {
    await page.goto('/dashboard/compliance');
    await expect(page).toHaveScreenshot('compliance-dashboard.png', {
      fullPage: true,
      maxDiffPixelRatio: 0.02, // Allow 2% difference
    });
  });

  test('chat interface matches snapshot', async ({ page }) => {
    await page.goto('/dashboard/chat');
    await expect(page).toHaveScreenshot('chat-interface.png', {
      fullPage: true,
      maxDiffPixelRatio: 0.02,
    });
  });

  test('review queue matches snapshot', async ({ page }) => {
    await page.goto('/review');
    await expect(page).toHaveScreenshot('review-queue.png', {
      maxDiffPixelRatio: 0.02,
    });
  });
});
```

## Test Configuration

```ts
// vitest.config.ts
import { defineConfig } from 'vitest/config';

export default defineConfig({
  test: {
    environment: 'jsdom',
    setupFiles: ['./tests/setup.ts'],
    globals: true,
    coverage: {
      provider: 'v8',
      reporter: ['text', 'json', 'html'],
      exclude: [
        'node_modules/',
        'tests/',
        '**/*.d.ts',
        '**/*.config.ts',
      ],
      thresholds: {
        lines: 80,
        functions: 80,
        branches: 75,
        statements: 80,
      },
    },
  },
});
```

```ts
// playwright.config.ts
import { defineConfig, devices } from '@playwright/test';

export default defineConfig({
  testDir: './tests/e2e',
  fullyParallel: true,
  forbidOnly: !!process.env.CI,
  retries: process.env.CI ? 2 : 0,
  workers: process.env.CI ? 1 : undefined,
  reporter: 'html',
  use: {
    baseURL: 'http://localhost:3000',
    trace: 'on-first-retry',
    screenshot: 'only-on-failure',
  },
  projects: [
    {
      name: 'chromium',
      use: { ...devices['Desktop Chrome'] },
    },
    {
      name: 'firefox',
      use: { ...devices['Desktop Firefox'] },
    },
    {
      name: 'webkit',
      use: { ...devices['Desktop Safari'] },
    },
  ],
  webServer: {
    command: 'npm run build && npm run start',
    url: 'http://localhost:3000',
    reuseExistingServer: !process.env.CI,
  },
});
```

## CI Integration

```yaml
# .github/workflows/frontend-tests.yml
name: Frontend Tests
on:
  pull_request:
    paths: ['frontend/**']

jobs:
  unit-tests:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: npm ci
      - run: npm run test:unit -- --coverage
      - name: Check coverage thresholds
        run: npm run test:coverage

  e2e-tests:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: npm ci
      - run: npx playwright install --with-deps
      - run: npm run test:e2e
      - uses: actions/upload-artifact@v4
        if: failure()
        with:
          name: playwright-report
          path: playwright-report/

  accessibility-tests:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: npm ci
      - run: npx playwright install --with-deps
      - run: npm run test:a11y
```

## Common Mistakes

### 1. Testing Implementation Details

```tsx
// ❌ BAD: Testing internal state
const { result } = renderHook(() => useCounter());
expect(result.current.count).toBe(0);
result.current.setCount(5);
expect(result.current.count).toBe(5);

// ✅ GOOD: Testing behavior and output
render(<Counter />);
expect(screen.getByText('0')).toBeInTheDocument();
fireEvent.click(screen.getByRole('button', { name: /increment/i }));
expect(screen.getByText('1')).toBeInTheDocument();
```

### 2. No E2E Tests for Critical Flows

Unit tests cannot catch integration bugs between components, API calls, and authentication.

### 3. Flaky Tests from Async Timing

```tsx
// ❌ BAD: Arbitrary timeout
await page.waitForTimeout(1000);

// ✅ GOOD: Wait for specific condition
await expect(page.getByText('Response')).toBeVisible();
```

## Cross-References

- `./frontend-testing.md` — This document
- `./error-boundaries.md` — Testing error boundaries
- `./accessibility.md` — Accessibility testing with axe-core
- `./genai-chat-interfaces.md` — Testing chat flows
- `../testing-and-quality/` — Backend testing strategies
- `./frontend-observability.md` — Test result monitoring

## Interview Questions

1. How do you test a custom hook that makes API calls?
2. What is the difference between unit, integration, and E2E tests?
3. How do you write a Playwright test for a streaming chat response?
4. What coverage thresholds do you set for a banking application?
5. How do you test accessibility in automated tests?
6. Design a test strategy for a multi-step form with validation.
