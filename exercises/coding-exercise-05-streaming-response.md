# Coding Exercise 05: Streaming Response Handler

> Implement Server-Sent Events (SSE) streaming for GenAI responses — how production LLM APIs deliver real-time output.

## Problem Statement

You are building a streaming response handler for the bank's GenAI chat API. Instead of waiting for the complete response, the API should stream tokens to the client as they are generated, providing a ChatGPT-like experience.

The implementation must:

1. Use Server-Sent Events (SSE) for streaming
2. Simulate token-by-token generation from an LLM
3. Include metadata (tokens generated, latency) in the final event
4. Handle client disconnection gracefully
5. Support cancellation via a `close` event from the client

**Banking Context:** When an employee asks a complex question, the GenAI assistant may take 3-5 seconds to generate a full response. Streaming improves perceived latency — users see the response building in real-time rather than waiting for a blank screen.

## Constraints

- Use FastAPI with `StreamingResponse`
- SSE format: `data: {json}\n\n` for each event
- Event types: `token`, `done`, `error`
- Simulate LLM generation with incremental text (no real model needed)
- Include `request_id` in every event
- Handle `asyncio.CancelledError` for client disconnection
- Maximum streaming duration: 30 seconds (timeout)

## Expected Output

```
# Client receives SSE events:

data: {"event": "token", "request_id": "uuid", "data": "According", "index": 0}

data: {"event": "token", "request_id": "uuid", "data": " to", "index": 1}

data: {"event": "token", "request_id": "uuid", "data": " the", "index": 2}

data: {"event": "token", "request_id": "uuid", "data": " expense", "index": 3}

...

data: {"event": "done", "request_id": "uuid", "data": {"tokens_generated": 45, "total_latency_ms": 3200}}
```

## Hints

### Hint 1: SSE Event Generator

```python
import asyncio
import json
import uuid
from typing import AsyncGenerator

async def generate_sse_events(
    request_id: str,
    full_response: str,
    token_delay: float = 0.05
) -> AsyncGenerator[str, None]:
    """Generate SSE events token by token."""
    words = full_response.split()
    tokens_generated = 0

    for i, word in enumerate(words):
        # Check if client disconnected
        if await asyncio.sleep(token_delay, return_value=False):
            break

        token_data = {
            "event": "token",
            "request_id": request_id,
            "data": word + (" " if i < len(words) - 1 else ""),
            "index": i,
        }
        yield f"data: {json.dumps(token_data)}\n\n"
        tokens_generated += 1

    # Final event with metadata
    done_data = {
        "event": "done",
        "request_id": request_id,
        "data": {
            "tokens_generated": tokens_generated,
        }
    }
    yield f"data: {json.dumps(done_data)}\n\n"
```

### Hint 2: FastAPI StreamingResponse

```python
from fastapi import FastAPI
from fastapi.responses import StreamingResponse

app = FastAPI()

@app.get("/v1/chat/stream")
async def stream_chat(query: str):
    request_id = str(uuid.uuid4())
    response_text = simulate_llm_response(query)

    return StreamingResponse(
        generate_sse_events(request_id, response_text),
        media_type="text/event-stream",
        headers={
            "Cache-Control": "no-cache",
            "Connection": "keep-alive",
            "X-Accel-Buffering": "no",  # Disable nginx buffering
        }
    )
```

### Hint 3: Simulating LLM Latency

```python
import random
import asyncio

async def simulate_token_generation(text: str, base_delay: float = 0.05):
    """Simulate token-by-token generation with variable delay."""
    for char in text:
        delay = base_delay + random.uniform(0, 0.03)
        await asyncio.sleep(delay)
        yield char
```

## Example Solution

```python
"""
SSE Streaming Response Handler for GenAI Chat API
Provides real-time token streaming with cancellation support.
"""

import asyncio
import json
import time
import uuid
import random
import logging
from datetime import datetime
from typing import AsyncGenerator, Optional

from fastapi import FastAPI, Request, HTTPException
from fastapi.responses import StreamingResponse

logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)

app = FastAPI(title="GenAI Streaming Chat API")

# ─── Mock LLM Response Generation ─────────────────────────────

RESPONSE_BANK = {
    "expense": (
        "According to the Expense Reporting Policy (v3.2), employees must submit "
        "expense reports within 30 days of the expense date. Expenses over $50 require "
        "itemized receipts. Expenses over $500 require manager pre-approval. "
        "International expenses must be submitted in USD using the exchange rate "
        "on the date of the expense. Late submissions require manager approval "
        "and may be subject to additional review by the finance team. "
        "All expenses are processed in the next payroll cycle after approval."
    ),
    "vacation": (
        "Full-time employees receive 20 vacation days per year, plus 5 personal days. "
        "Vacation requests must be submitted at least 2 weeks in advance through "
        "the HR portal. Managers approve requests based on team coverage. "
        "Unused vacation days can be carried over to the next year, up to a maximum "
        "of 5 days. Alternatively, employees may request payout of unused days "
        "during the annual benefits enrollment period."
    ),
    "security": (
        "All employees must complete security awareness training annually. "
        "Passwords must be at least 12 characters with mixed case, numbers, and symbols. "
        "Multi-factor authentication is required for all systems accessing customer data. "
        "Suspicious emails must be reported to the security team within 1 hour. "
        "Laptops must be encrypted using BitLocker. Mobile devices must have "
        "remote wipe capability enabled. Any suspected data breach must be "
        "reported immediately to the CISO office."
    ),
    "default": (
        "Thank you for your question. Based on our internal knowledge base, "
        "I've found the following information that may be helpful. "
        "Please note that this information is based on the most recent policy "
        "documents available. If you need more specific guidance, "
        "please contact the relevant department directly. "
        "You can also ask me more specific questions and I'll do my best "
        "to provide accurate answers from our policy library."
    ),
}

def get_mock_response(query: str) -> str:
    """Select a mock response based on query keywords."""
    query_lower = query.lower()
    for keyword, response in RESPONSE_BANK.items():
        if keyword != "default" and keyword in query_lower:
            return response
    return RESPONSE_BANK["default"]

# ─── SSE Event Generator ──────────────────────────────────────

async def stream_tokens(
    request_id: str,
    full_text: str,
    max_tokens: int = 100,
) -> AsyncGenerator[str, None]:
    """
    Stream SSE events token by token, simulating LLM generation.

    Handles client disconnection via asyncio.CancelledError.
    Enforces a 30-second timeout.
    """
    start_time = time.time()
    words = full_text.split()[:max_tokens]  # Limit max tokens
    tokens_generated = 0

    try:
        for i, word in enumerate(words):
            # Check timeout (30 seconds max)
            if time.time() - start_time > 30:
                error_event = {
                    "event": "error",
                    "request_id": request_id,
                    "data": {"error": "timeout", "message": "Response generation timed out"},
                }
                yield f"data: {json.dumps(error_event)}\n\n"
                break

            # Simulate variable token generation delay
            delay = random.uniform(0.02, 0.08)
            await asyncio.sleep(delay)

            # Check for client disconnect (CancelledError will be raised)
            token_data = {
                "event": "token",
                "request_id": request_id,
                "data": word + (" " if i < len(words) - 1 else ""),
                "index": i,
            }
            yield f"data: {json.dumps(token_data)}\n\n"
            tokens_generated += 1

        # Send completion event
        total_latency_ms = int((time.time() - start_time) * 1000)
        done_event = {
            "event": "done",
            "request_id": request_id,
            "data": {
                "tokens_generated": tokens_generated,
                "total_latency_ms": total_latency_ms,
                "model": "mock-gpt-4",
            }
        }
        yield f"data: {json.dumps(done_event)}\n\n"

        logger.info(
            f"Streaming complete for {request_id}: "
            f"{tokens_generated} tokens in {total_latency_ms}ms"
        )

    except asyncio.CancelledError:
        # Client disconnected
        logger.info(f"Client disconnected for request {request_id}")
        yield f"data: {json.dumps({'event': 'done', 'request_id': request_id, 'data': {'cancelled': True}})}\n\n"
        raise

# ─── API Endpoints ─────────────────────────────────────────────

@app.post("/v1/chat/stream")
async def stream_chat(request: Request):
    """
    Stream a chat response using Server-Sent Events.

    Request body: {"query": "string"}
    Response: SSE stream of tokens
    """
    try:
        body = await request.json()
    except json.JSONDecodeError:
        raise HTTPException(status_code=400, detail="Invalid JSON")

    query = body.get("query", "")
    if not query or len(query) > 4000:
        raise HTTPException(status_code=400, detail="Query must be 1-4000 characters")

    request_id = str(uuid.uuid4())
    response_text = get_mock_response(query)

    logger.info(f"Starting stream for {request_id}: query='{query[:50]}...'")

    return StreamingResponse(
        stream_tokens(request_id, response_text),
        media_type="text/event-stream",
        headers={
            "Cache-Control": "no-cache, no-transform",
            "Connection": "keep-alive",
            "X-Accel-Buffering": "no",
            "Content-Encoding": "identity",
        }
    )

@app.get("/health")
async def health():
    return {"status": "healthy", "streaming": True}

# ─── Client Example ────────────────────────────────────────────
"""
To test with curl:

curl -N -X POST http://localhost:8000/v1/chat/stream \
  -H "Content-Type: application/json" \
  -d '{"query": "What is the expense policy?"}'

To test with JavaScript:

const eventSource = new EventSource('/v1/chat/stream');
// Actually, SSE uses GET. For POST + SSE, use fetch:

const response = await fetch('/v1/chat/stream', {
    method: 'POST',
    headers: {'Content-Type': 'application/json'},
    body: JSON.stringify({query: 'What is the expense policy?'})
});

const reader = response.body.getReader();
const decoder = new TextDecoder();

while (true) {
    const {done, value} = await reader.read();
    if (done) break;
    const text = decoder.decode(value);
    // Parse SSE events from text
    for (const line of text.split('\\n')) {
        if (line.startsWith('data: ')) {
            const event = JSON.parse(line.slice(6));
            if (event.event === 'token') {
                process.stdout.write(event.data);
            } else if (event.event === 'done') {
                console.log('\\nDone:', event.data);
            }
        }
    }
}
"""
```

## Extensions

1. **Add real LLM integration:** Replace mock response with actual streaming from an LLM API that supports streaming (OpenAI, Anthropic).

2. **Implement backpressure:** If the client is slow to consume events, buffer up to N events before dropping or pausing.

3. **Add retry logic:** SSE supports automatic reconnection with `Retry:` header. Implement resume from last received token index.

4. **Build a React client:** Create a simple chat UI that consumes the SSE stream and displays tokens as they arrive with a typing indicator.

5. **Add WebSocket support:** Compare SSE vs. WebSocket for bidirectional streaming (client can send follow-up queries without new connections).

## Interview Relevance

Streaming is essential for production GenAI UX:

| Skill | Why It Matters |
|-------|---------------|
| Async generators | Core Python streaming pattern |
| SSE protocol | Standard for server-to-client streaming |
| Cancellation handling | Resource cleanup on disconnect |
| Timeout enforcement | Prevent runaway processes |
| FastAPI StreamingResponse | Production framework for streaming |

**Follow-up questions:**
- "When would you use WebSocket instead of SSE?"
- "How do you handle a client that disconnects mid-stream?"
- "What happens if the LLM API is slower than the client can consume?"
- "How do you test streaming endpoints?"
- "What nginx configuration is needed for SSE?"
