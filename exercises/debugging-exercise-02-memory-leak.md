# Debugging Exercise 02: Memory Leak

> Debug a memory leak in a Python GenAI service — a subtle but critical production issue.

## Problem Statement

The GenAI chat service is experiencing gradual memory growth. Over 24 hours, memory usage grows from 512MB to 4GB, eventually triggering OOMKill and pod restart. The service restarts every 18-24 hours.

Find and fix the memory leak.

## Symptoms

```
Observations:
- Memory grows linearly at ~170MB/hour
- Restart fixes it temporarily (memory returns to 512MB)
- No increase in traffic correlates with memory growth
- GC runs frequently but doesn't reclaim memory
- No large objects in heap dumps (no single object is huge)
- Issue started after deploying v2.3.0 two weeks ago

What changed in v2.3.0:
- Added response caching (Redis)
- Added conversation history tracking
- Added embedding cache for repeated queries
- Updated Pydantic from v1 to v2
- Added request/response logging buffer for debugging
```

## Investigation

```bash
# Check memory growth
kubectl top pod -n genai --containers

# Get heap dump (using tracemalloc)
curl http://genai-chat:8000/debug/memdump

# Analyze with Fil or tracemalloc
python -c "import tracemalloc; tracemalloc.start()"
```

## Hints

### Hint 1: Check the Logging Buffer

```python
# In v2.3.0, a debug logging buffer was added:
class DebugLogger:
    _buffer = []  # Class-level list — never cleared!

    @classmethod
    def log(cls, request_id, query, response):
        cls._buffer.append({
            "request_id": request_id,
            "query": query,       # Full query text (can be 4000 chars)
            "response": response,  # Full response (can be 10K+ chars)
            "timestamp": time.time(),
        })
        # No size limit, no cleanup!

# At 120 queries/minute × 14KB average = ~1.7MB/minute
# = ~100MB/hour from the buffer alone
```

### Hint 2: Check the Embedding Cache

```python
# Embedding cache uses an unbounded lru_cache:
from functools import lru_cache

@lru_cache(maxsize=None)  # No limit!
def get_embedding(query: str) -> list[float]:
    return model.encode(query).tolist()

# Every unique query is cached forever
# At 120 queries/min with 70% unique = 84 new cache entries/min
# Each entry: 384 floats × 24 bytes + query string = ~10KB
# = ~50MB/hour from embedding cache
```

### Hint 3: Check the Conversation History

```python
# Conversation history stored in a global dict:
conversation_histories = {}  # user_id → list of messages

# Every interaction appends to the list:
def add_to_history(user_id: str, query: str, response: str):
    if user_id not in conversation_histories:
        conversation_histories[user_id] = []
    conversation_histories[user_id].append({
        "query": query,
        "response": response,
        "timestamp": time.time(),
    })
    # No limit on history length per user
    # No eviction of old histories
```

## Root Cause Analysis

Three memory leaks identified:

| Source | Growth Rate | Fix |
|--------|------------|-----|
| Debug logger buffer | ~100MB/hour | Use bounded deque with maxlen |
| Unbounded embedding cache | ~50MB/hour | Set maxsize on lru_cache |
| Unbounded conversation history | ~20MB/hour | Limit history length, add TTL |

Total: ~170MB/hour → matches observed growth rate.

## Fixes

```python
# Fix 1: Bounded debug buffer
from collections import deque

class DebugLogger:
    _buffer = deque(maxlen=1000)  # Keep only last 1000 entries

    @classmethod
    def log(cls, request_id, query, response):
        cls._buffer.append({
            "request_id": request_id,
            "query_preview": query[:200],  # Don't store full text
            "response_preview": response[:500],
            "timestamp": time.time(),
        })

# Fix 2: Bounded embedding cache
@lru_cache(maxsize=10000)  # Max 10K embeddings (~100MB)
def get_embedding(query: str) -> list[float]:
    return model.encode(query).tolist()

# Fix 3: Bounded conversation history
MAX_HISTORY_PER_USER = 20
HISTORY_TTL_HOURS = 24

def add_to_history(user_id: str, query: str, response: str):
    if user_id not in conversation_histories:
        conversation_histories[user_id] = deque(maxlen=MAX_HISTORY_PER_USER)
    conversation_histories[user_id].append({
        "query_preview": query[:500],
        "response_preview": response[:1000],
        "timestamp": time.time(),
    })

# Cleanup old histories periodically
def cleanup_old_histories():
    cutoff = time.time() - (HISTORY_TTL_HOURS * 3600)
    to_remove = [
        uid for uid, hist in conversation_histories.items()
        if hist and hist[-1]["timestamp"] < cutoff
    ]
    for uid in to_remove:
        del conversation_histories[uid]
```

## Prevention Measures

1. **Memory monitoring:** Track RSS memory over time with alerting on linear growth
2. **Memory limits in code review:** Flag unbounded collections (dicts, lists, caches without limits)
3. **Automated memory testing:** Run load tests for 4+ hours and verify memory is stable
4. **Use memory profiling in CI:** Tools like `pytest-memray` or `Fil` profiler
5. **Production heap dumps:** Enable on-demand heap analysis in production

## Extensions

1. **Reproduce with load testing:** Write a load test that runs for 4 hours and demonstrates the memory growth.

2. **Build a memory dashboard:** Create a Grafana dashboard showing memory trends, GC statistics, and object counts.

3. **Implement memory budgets:** Set per-component memory budgets and alert when exceeded.

4. **Compare GC strategies:** Experiment with `gc.disable()` during request processing and `gc.collect()` between requests.

5. **Async memory leaks:** Investigate memory leaks specific to async Python (unclosed connections, task references).

## Interview Relevance

Memory leak debugging tests deep Python skills:

| Skill | Why It Matters |
|-------|---------------|
| Memory profiling | Identifying leak sources |
| Python GC understanding | Why isn't GC catching this? |
| Bounded data structures | Preventing unbounded growth |
| Production debugging | Investigating without restarting |
| Prevention design | Systematic approach to avoiding leaks |

**Follow-up questions:**
- "Why doesn't Python's garbage collector catch these leaks?"
- "What's the difference between reference cycles and true leaks?"
- "How would you debug this in a production system you can't restart?"
- "What tools would you use for memory profiling in Python?"
