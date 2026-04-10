# End-to-End Testing for Banking GenAI

## What Is E2E Testing?

End-to-end tests verify complete user flows from start to finish, exercising the full stack: frontend, backend, database, external APIs, and GenAI components. E2E tests answer the question: "Does the feature work for a real user?"

## E2E Test Design Principles

1. **Test user journeys, not code paths**: E2E tests should mirror how real users interact with the system.

2. **Minimize E2E tests**: They are slow, flaky, and expensive. Use them for critical flows only.

3. **Stability over coverage**: 10 reliable E2E tests are better than 100 flaky ones.

4. **Independent tests**: Each test should be able to run in isolation without depending on other tests.

## Playwright for E2E Testing

```python
# tests/e2e/test_mortgage_advisor_flow.py
import pytest
from playwright.sync_api import Page, expect

@pytest.fixture(autouse=True)
def setup_test_data(db):
    """Set up test data before each test."""
    create_test_user(db, email="test-user@bank.example", tier="premium")
    seed_mortgage_rates(db)

def test_customer_completes_mortgage_inquiry(page: Page, base_url: str):
    """E2E: Customer asks about mortgage affordability and receives advice."""

    # 1. Navigate to GenAI chat
    page.goto(f"{base_url}/genai-advisor")

    # 2. Authenticate (if not already)
    page.fill("#email", "test-user@bank.example")
    page.fill("#password", "TestP@ssw0rd!")
    page.click("#login-btn")
    expect(page.locator("#welcome-message")).to_be_visible()

    # 3. Send a mortgage inquiry
    chat_input = page.locator("#chat-input")
    chat_input.fill("Can I afford a $450,000 home with a $95,000 salary?")
    page.click("#send-btn")

    # 4. Wait for streaming response
    response_area = page.locator("#chat-response")
    expect(response_area).to_be_visible(timeout=30_000)

    # 5. Verify response contains expected elements
    expect(page.locator(".disclaimer")).to_be_visible(
        timeout=10_000
    )
    expect(page.locator(".monthly-payment")).to_contain_text("$")
    expect(page.locator(".source-documents")).to_be_visible()

    # 6. Verify feedback mechanism is present
    expect(page.locator("#thumbs-up")).to_be_visible()
    expect(page.locator("#thumbs-down")).to_be_visible()

def test_customer_receieves_compliant_advice(page: Page, base_url: str):
    """E2E: AI advice includes required compliance disclaimers."""

    page.goto(f"{base_url}/genai-advisor")
    authenticate(page)

    page.locator("#chat-input").fill(
        "What is the best mortgage product for me?"
    )
    page.click("#send-btn")

    # Wait for complete response
    expect(page.locator(".response-complete")).to_be_visible(timeout=30_000)

    # Compliance: disclaimer must be visible
    disclaimer = page.locator(".disclaimer")
    expect(disclaimer).to_be_visible()
    expect(disaction).to_contain_text("informational purposes")

    # Compliance: no guaranteed approval language
    response_text = page.locator("#chat-response").inner_text()
    assert "guaranteed" not in response_text.lower()
    assert "will be approved" not in response_text.lower()

def test_customer_views_source_documents(page: Page, base_url: str):
    """E2E: Customer can expand and view source documents."""

    page.goto(f"{base_url}/genai-advisor")
    authenticate(page)

    page.locator("#chat-input").fill("What are current mortgage rates?")
    page.click("#send-btn")
    expect(page.locator(".response-complete")).to_be_visible(timeout=30_000)

    # Expand source documents
    page.click(".source-documents-toggle")
    expect(page.locator(".source-document")).to_have_count_at_least(1)

    # Each source should have content and attribution
    for i in range(page.locator(".source-document").count()):
        source = page.locator(".source-document").nth(i)
        expect(source.locator(".source-content")).to_be_visible()
        expect(source.locator(".source-attribution")).to_be_visible()
```

## E2E Test Architecture

```
┌─────────────────────────────────────────────────────────────┐
│  E2E TEST ARCHITECTURE                                      │
│                                                             │
│  Playwright Test Runner                                     │
│  ┌──────────────────────────────────────────────────────┐  │
│  │  Test: Customer completes mortgage inquiry           │  │
│  │                                                      │  │
│  │  Browser ──> Frontend ──> API ──> Services ──> DB   │  │
│  │    │            │           │          │         │    │  │
│  │    │            │           │          │         └> Test │  │
│  │    │            │           │          └───────> Container│  │
│  │    │            │           └────────────────> Mock LLM  │  │
│  │    │            └─────────────────────> Test Frontend    │  │
│  │    └> Headless Chrome                                    │  │
│  └──────────────────────────────────────────────────────────┘  │
│                                                                 │
│  Test Environment:                                              │
│  - Frontend: Built and served from test build                   │
│  - Backend: Running in test mode with test DB                   │
│  - Database: PostgreSQL test container, fresh per test          │
│  - LLM: Mock server with canned responses                       │
│  - Vector DB: pgvector test container with seeded documents     │
│  - Auth: Test identity provider with test users                 │
└─────────────────────────────────────────────────────────────────┘
```

## Stable E2E Tests

### Use Data Attributes, Not CSS Selectors

```html
<!-- GOOD: Stable, semantic selectors -->
<button data-testid="send-message-btn">Send</button>
<div data-testid="chat-response">...</div>
<span data-testid="monthly-payment-amount">$2,844</span>

<!-- BAD: Brittle, implementation-dependent selectors -->
<button class="MuiButton-root MuiButton-contained">Send</button>
<div class="css-1a2b3c4d5e">...</div>
```

```python
# GOOD
expect(page.get_by_test_id("chat-response")).to_be_visible()

# BAD - breaks when CSS changes
expect(page.locator(".css-1a2b3c4d5e")).to_be_visible()
```

### Handle Streaming Responses

```python
def wait_for_complete_response(page: Page, timeout: int = 30_000):
    """Wait for streaming response to complete."""
    # Wait for the streaming indicator to disappear
    expect(page.locator(".streaming-indicator")).to_be_hidden(
        timeout=timeout
    )
    # Verify the response complete marker is present
    expect(page.locator(".response-complete")).to_be_visible(
        timeout=5_000
    )

def wait_for_streaming_to_start(page: Page, timeout: int = 5_000):
    """Wait for streaming to begin (TTFT check)."""
    expect(page.locator(".streaming-indicator")).to_be_visible(
        timeout=timeout
    )
```

### E2E Test Configuration

```python
# conftest.py
import pytest
from playwright.sync_api import sync_playwright
from testcontainers.postgres import PostgresContainer

@pytest.fixture(scope="session")
def browser():
    with sync_playwright() as p:
        browser = p.chromium.launch(
            headless=True,
            slow_mo=100,  # Slow down for stability
        )
        yield browser
        browser.close()

@pytest.fixture
def page(browser):
    context = browser.new_context(
        viewport={"width": 1280, "height": 720},
        locale="en-US",
        timezone_id="America/New_York",
    )
    page = context.new_page()
    yield page
    context.close()

@pytest.fixture(scope="session")
def test_db():
    with PostgresContainer("postgres:16") as pg:
        engine = create_engine(pg.get_connection_url())
        run_migrations(engine)
        yield engine
```

## E2E Tests for Critical Banking Flows

```
┌─────────────────────────────────────────────────────────────┐
│  CRITICAL E2E FLOWS                                          │
├─────────────────────────────────────────────────────────────┤
│  1. Customer authentication and session management           │
│  2. Mortgage affordability inquiry and advice                │
│  3. Document upload and analysis                             │
│  4. Investment portfolio question and response               │
│  5. Credit card product inquiry                              │
│  6. Customer gives negative feedback and escalation          │
│  7. AI provides compliant advice (disclaimer present)        │
│  8. System handles LLM provider failure gracefully           │
│  9. Customer views audit trail of past conversations         │
│  10. Mobile-responsive chat interface                        │
└─────────────────────────────────────────────────────────────┘
```

## CI/CD Integration

```yaml
# .github/workflows/e2e-tests.yml
name: E2E Tests

on:
  push:
    branches: [main]
  pull_request:
    paths:
      - "frontend/**"
      - "backend/**"
      - "tests/e2e/**"

jobs:
  e2e:
    runs-on: ubuntu-latest
    timeout-minutes: 30

    steps:
      - uses: actions/checkout@v4

      - name: Start test environment
        run: docker compose -f docker-compose.test.yml up -d

      - name: Wait for services
        run: |
          ./scripts/wait-for-services.sh

      - name: Run E2E tests
        run: |
          pytest tests/e2e/ \
            --headed \
            --video=on \
            --screenshot=on \
            --tracing=on \
            --browser=chromium \
            -v

      - name: Upload artifacts on failure
        if: failure()
        uses: actions/upload-artifact@v4
        with:
          name: e2e-artifacts
          path: |
            test-results/
            playwright-traces/
```

## Common E2E Testing Mistakes

1. **Too many E2E tests**: E2E tests should cover critical flows only. Use unit and integration tests for everything else.

2. **Tests depend on each other**: Each E2E test must be independently runnable. No shared state between tests.

3. **No retry for flaky tests**: Flaky tests erode trust. Identify and fix flaky tests; do not just add retries.

4. **Testing in production**: E2E tests should run against a test environment with controlled data.

5. **Ignoring mobile**: Banking customers increasingly use mobile. Test on mobile viewports.

6. **Not testing failure modes**: Test what happens when the LLM provider fails. Does the UI show an appropriate error?
