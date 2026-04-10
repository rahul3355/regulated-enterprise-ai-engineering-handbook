# Async Python Patterns

## Why Asyncio

Asyncio enables handling thousands of concurrent connections with a single thread. In banking GenAI platforms, async is essential for I/O-bound operations: database queries, external API calls, file uploads, and AI inference requests. Async code achieves high throughput without the memory overhead of thread-per-request models.

## Event Loop Fundamentals

```python
import asyncio

# The event loop is the core of asyncio
# It runs coroutines, callbacks, and futures in a single thread

async def main():
    """Main coroutine - entry point."""
    print('Starting...')
    await asyncio.sleep(1)  # Non-blocking sleep
    print('Done!')

# Run the event loop
asyncio.run(main())
```

### How the Event Loop Works

```
Event Loop:
1. Run ready coroutines until they await
2. When coroutine awaits, register callback
3. Continue running other coroutines
4. When I/O completes, resume awaiting coroutine
5. Repeat until all coroutines complete

Key: No parallelism within single loop
     Concurrency through interleaving
     CPU-bound work blocks the loop
```

## Async/Await Patterns

### Basic Async Function

```python
import aiohttp
import asyncio

async def fetch_account(session: aiohttp.ClientSession, account_id: str) -> dict:
    """Fetch a single account asynchronously."""
    async with session.get(f'/v1/accounts/{account_id}') as response:
        return await response.json()

async def fetch_all_accounts(account_ids: list[str]) -> list[dict]:
    """Fetch all accounts concurrently."""
    async with aiohttp.ClientSession() as session:
        # Create tasks for all accounts
        tasks = [
            fetch_account(session, acc_id)
            for acc_id in account_ids
        ]
        # Run all tasks concurrently
        results = await asyncio.gather(*tasks)
        return results
```

### Concurrent Execution Patterns

```python
# Pattern 1: asyncio.gather (all or nothing)
async def fetch_with_gather():
    """All succeed or all fail."""
    results = await asyncio.gather(
        fetch_account('ACC-001'),
        fetch_account('ACC-002'),
        fetch_account('ACC-003'),
    )
    return results

# Pattern 2: asyncio.as_completed (process as they finish)
async def fetch_as_completed():
    """Process results as they arrive."""
    tasks = [fetch_account(acc_id) for acc_id in account_ids]

    for coro in asyncio.as_completed(tasks):
        result = await coro
        process_result(result)  # Process immediately

# Pattern 3: asyncio.wait (with timeout/return_when)
async def fetch_with_timeout():
    """Wait for some tasks with timeout."""
    tasks = [fetch_account(acc_id) for acc_id in account_ids]

    done, pending = await asyncio.wait(tasks, timeout=10)

    # Cancel pending tasks
    for task in pending:
        task.cancel()

    # Collect results from completed tasks
    results = [task.result() for task in done]
    return results
```

### Error Handling in Concurrent Tasks

```python
async def fetch_with_error_handling():
    """Handle individual task failures."""
    tasks = [fetch_account(acc_id) for acc_id in account_ids]

    # return_exceptions=True: errors returned as values, not raised
    results = await asyncio.gather(*tasks, return_exceptions=True)

    for i, result in enumerate(results):
        if isinstance(result, Exception):
            print(f'Account {account_ids[i]} failed: {result}')
        else:
            print(f'Account {account_ids[i]} succeeded: {result}')


async def fetch_with_individual_error_handling():
    """Handle errors per task."""
    async def safe_fetch(account_id: str):
        try:
            return await fetch_account(account_id)
        except aiohttp.ClientError as e:
            print(f'Failed to fetch {account_id}: {e}')
            return None

    tasks = [safe_fetch(acc_id) for acc_id in account_ids]
    results = await asyncio.gather(*tasks)

    # Filter out None results
    return [r for r in results if r is not None]
```

## Semaphores for Concurrency Control

```python
async def process_payments_with_limit(payments: list[dict], max_concurrent: int = 10):
    """Process payments with concurrency limit."""
    semaphore = asyncio.Semaphore(max_concurrent)

    async def process_one(payment: dict):
        async with semaphore:
            return await execute_payment(payment)

    tasks = [process_one(p) for p in payments]
    results = await asyncio.gather(*tasks, return_exceptions=True)

    return results


async def rate_limited_api_call(api_key: str, max_per_second: int = 10):
    """Rate limit API calls."""
    semaphore = asyncio.Semaphore(max_per_second)

    async def call_with_limit():
        async with semaphore:
            return await make_api_call()

    # Also need time-based rate limiting
    tasks = [call_with_limit() for _ in range(100)]
    results = await asyncio.gather(*tasks, return_exceptions=True)
    return results
```

## Async Context Managers

```python
import aiohttp
from contextlib import asynccontextmanager


@asynccontextmanager
async def get_db_session():
    """Async context manager for database sessions."""
    session = await async_engine.connect()
    try:
        yield session
        await session.commit()
    except Exception:
        await session.rollback()
        raise
    finally:
        await session.close()


# Usage
async def create_account(data: dict):
    async with get_db_session() as session:
        account = Account(**data)
        session.add(account)
        return account


# Async file operations
async def read_file_async(filepath: str) -> str:
    """Read file asynchronously."""
    import aiofiles
    async with aiofiles.open(filepath, 'r') as f:
        return await f.read()
```

## Async Redis Operations

```python
import redis.asyncio as aioredis

class AsyncCacheService:
    def __init__(self, redis_url: str):
        self.redis = aioredis.from_url(redis_url)

    async def get(self, key: str) -> str | None:
        """Get value from cache."""
        return await self.redis.get(key)

    async def set(self, key: str, value: str, ttl: int = 300):
        """Set value in cache with TTL."""
        await self.redis.set(key, value, ex=ttl)

    async def delete(self, key: str):
        """Delete value from cache."""
        await self.redis.delete(key)

    async def get_many(self, keys: list[str]) -> dict:
        """Get multiple values concurrently."""
        # Pipeline: single round trip
        pipe = self.redis.pipeline()
        for key in keys:
            pipe.get(key)
        values = await pipe.execute()
        return dict(zip(keys, values))
```

## Async Database Operations

```python
from sqlalchemy.ext.asyncio import create_async_engine, AsyncSession
from sqlalchemy.orm import sessionmaker

# Create async engine
engine = create_async_engine(
    'postgresql+asyncpg://user:pass@localhost:5432/banking',
    pool_size=20,
    max_overflow=10,
)

AsyncSessionLocal = sessionmaker(
    engine,
    class_=AsyncSession,
    expire_on_commit=False,
)


async def get_db_session() -> AsyncSession:
    """FastAPI dependency for async database session."""
    async with AsyncSessionLocal() as session:
        try:
            yield session
            await session.commit()
        except Exception:
            await session.rollback()
            raise


# Async ORM queries
async def get_account_with_transactions(account_id: str) -> dict:
    """Fetch account and transactions concurrently."""
    async with AsyncSessionLocal() as session:
        # Fetch account
        account = await session.get(Account, account_id)

        # Fetch transactions
        result = await session.execute(
            select(Transaction)
            .where(Transaction.account_id == account_id)
            .order_by(Transaction.created_at.desc())
            .limit(50)
        )
        transactions = result.scalars().all()

        return {
            'account': account,
            'transactions': transactions,
        }
```

## Blocking Code in Async Context

### Problem: Blocking the Event Loop

```python
# BAD: Blocking call in async function
async def bad_example():
    result = requests.get('https://api.example.com/data')  # BLOCKS!
    return result.json()

# GOOD: Async HTTP call
async def good_example():
    async with aiohttp.ClientSession() as session:
        async with session.get('https://api.example.com/data') as response:
            return await response.json()
```

### Running CPU-Bound Code

```python
import asyncio
from concurrent.futures import ProcessPoolExecutor

def compute_risk_score(data: dict) -> float:
    """CPU-intensive computation (runs in process pool)."""
    # Heavy ML computation
    return model.predict(data)


async def compute_risk_async(data: dict) -> float:
    """Run CPU-bound function in process pool."""
    loop = asyncio.get_event_loop()

    # Offload to process pool (doesn't block event loop)
    with ProcessPoolExecutor() as executor:
        result = await loop.run_in_executor(executor, compute_risk_score, data)

    return result


# For multiple computations
async def compute_risk_scores_batch(data_list: list[dict]) -> list[float]:
    """Batch compute risk scores."""
    loop = asyncio.get_event_loop()

    with ProcessPoolExecutor() as executor:
        # Run all computations in parallel processes
        futures = [
            loop.run_in_executor(executor, compute_risk_score, data)
            for data in data_list
        ]
        return await asyncio.gather(*futures)
```

## Async Generators

```python
async def stream_transactions(account_id: str, batch_size: int = 100):
    """Stream transactions in batches using async generator."""
    offset = 0
    while True:
        transactions = await fetch_transactions(
            account_id,
            limit=batch_size,
            offset=offset,
        )

        if not transactions:
            break

        yield transactions
        offset += batch_size


# Usage
async def process_all_transactions(account_id: str):
    async for batch in stream_transactions(account_id):
        for txn in batch:
            await process_single_transaction(txn)
```

## Async Patterns for GenAI

### Streaming LLM Responses

```python
async def stream_llm_response(prompt: str):
    """Stream LLM tokens as async generator."""
    async for chunk in openai_client.chat.completions.create(
        model='gpt-4',
        messages=[{'role': 'user', 'content': prompt}],
        stream=True,
    ):
        if chunk.choices[0].delta.content:
            yield chunk.choices[0].delta.content


# Usage in FastAPI
from fastapi.responses import StreamingResponse

@app.get('/v1/ai/chat/stream')
async def chat_stream(prompt: str):
    async def generate():
        async for token in stream_llm_response(prompt):
            yield f'data: {json.dumps({"token": token})}\n\n'
        yield 'data: [DONE]\n\n'

    return StreamingResponse(generate(), media_type='text/event-stream')
```

### Concurrent AI Inference

```python
async def batch_analyze_documents(documents: list[str], max_concurrent: int = 5):
    """Analyze multiple documents concurrently with limit."""
    semaphore = asyncio.Semaphore(max_concurrent)

    async def analyze_one(doc: str):
        async with semaphore:
            return await ai_analyze(doc)

    tasks = [analyze_one(doc) for doc in documents]
    results = await asyncio.gather(*tasks, return_exceptions=True)

    return results
```

## Common Mistakes

1. **Blocking the event loop**: Using `requests`, `time.sleep`, or `open()` in async code
2. **No error handling in gather**: One failed task cancels all others
3. **Creating too many connections**: Not limiting concurrent database/API connections
4. **Not cancelling tasks on shutdown**: Background tasks continue after app stops
5. **Mixing sync and async**: Calling async functions from sync context
6. **Ignoring task results**: Fire-and-forget without error handling
7. **Not using connection pooling**: Creating new connection for each request

## Interview Questions

1. How do you handle a CPU-intensive task inside an async FastAPI endpoint?
2. Design an async system that processes 1000 payments concurrently with a limit of 50.
3. What happens when one task in `asyncio.gather()` raises an exception?
4. How do you stream LLM tokens to a client using async generators?
5. Explain the difference between `asyncio.gather()` and `asyncio.as_completed()`.

## Cross-References

- `./fastapi.md` - Async FastAPI endpoints
- `./celery.md` - When to use async vs Celery
- `./sqlalchemy.md` - Async SQLAlchemy patterns
- `../async-processing/README.md` - Background job processing
- `../concurrency/README.md` - Concurrency models comparison
