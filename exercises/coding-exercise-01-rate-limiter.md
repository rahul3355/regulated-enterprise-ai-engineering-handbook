# Coding Exercise 01: Implement a Rate Limiter

## Problem Statement

You are building a GenAI advisory platform for a retail bank. The platform uses expensive LLM API calls, and you need to prevent any single customer from consuming excessive resources.

Implement a **Token Bucket Rate Limiter** that enforces per-user query limits on the chat API. The bank requires:

- Standard customers: 50 queries per hour
- Premium customers: 200 queries per hour
- The rate limiter must be thread-safe (multiple requests arrive concurrently)
- If a user exceeds their limit, return the time until they can make another request

## Constraints and Requirements

1. **Accuracy**: The rate limiter must not allow more than the configured rate over any sliding window.
2. **Concurrency**: Must handle concurrent requests from the same user without race conditions.
3. **Memory**: Must not leak memory for users who stop making requests.
4. **Performance**: The rate check must complete in under 1ms per request.
5. **Banking context**: The rate limit response must include a `Retry-After` header for compliance with API standards.

## Expected Output

```python
limiter = RateLimiter()

# Standard customer: 50 requests/hour
limiter.configure(user_id="cust-123", tier="standard")

for i in range(55):
    allowed, info = limiter.check("cust-123")
    if not allowed:
        print(f"Request {i+1}: DENIED. Retry after {info['retry_after_seconds']}s")
    else:
        print(f"Request {i+1}: ALLOWED. Remaining: {info['remaining']}")

# Output:
# Request 1: ALLOWED. Remaining: 49
# Request 2: ALLOWED. Remaining: 48
# ...
# Request 50: ALLOWED. Remaining: 0
# Request 51: DENIED. Retry after 72.0s
# Request 52: DENIED. Retry after 72.0s
# ...
```

## Hints

<details>
<summary>Hint 1 (Subtle)</summary>
Think about what happens between tokens being "added" to the bucket. You do not need to actually add tokens on a timer. Consider calculating available tokens based on elapsed time since the last request.
</details>

<details>
<summary>Hint 2 (More specific)</summary>
Use the formula: `available_tokens = min(max_tokens, last_tokens + (elapsed_time * refill_rate))`. Update the bucket state only when a request arrives. This is called the "lazy" or "on-demand" approach.
</details>

<details>
<summary>Hint 3 (Direct)</summary>
Use `threading.Lock()` per user to ensure thread safety. Store bucket state (tokens, last_refill_time) in a dictionary. For cleanup, consider using `time-to-live` eviction or periodic garbage collection.
</details>

## Example Solution

```python
import time
import threading
from dataclasses import dataclass, field
from typing import Optional


@dataclass
class BucketState:
    tokens: float
    last_refill: float
    max_tokens: int
    refill_rate: float  # tokens per second


class RateLimiter:
    TIER_CONFIG = {
        "standard": {"max_tokens": 50, "window_seconds": 3600},
        "premium": {"max_tokens": 200, "window_seconds": 3600},
    }

    def __init__(self):
        self._buckets: dict[str, BucketState] = {}
        self._locks: dict[str, threading.Lock] = {}
        self._global_lock = threading.Lock()

    def configure(self, user_id: str, tier: str) -> None:
        """Configure rate limit tier for a user."""
        config = self.TIER_CONFIG[tier]
        refill_rate = config["max_tokens"] / config["window_seconds"]

        with self._global_lock:
            self._buckets[user_id] = BucketState(
                tokens=config["max_tokens"],
                last_refill=time.time(),
                max_tokens=config["max_tokens"],
                refill_rate=refill_rate,
            )
            self._locks[user_id] = threading.Lock()

    def check(self, user_id: str) -> tuple[bool, dict]:
        """Check if a request is allowed. Returns (allowed, info)."""
        with self._global_lock:
            if user_id not in self._buckets:
                raise ValueError(f"User {user_id} not configured")
            lock = self._locks[user_id]
            bucket = self._buckets[user_id]

        with lock:
            now = time.time()
            elapsed = now - bucket.last_refill

            # Refill tokens based on elapsed time
            new_tokens = bucket.tokens + (elapsed * bucket.refill_rate)
            bucket.tokens = min(new_tokens, bucket.max_tokens)
            bucket.last_refill = now

            if bucket.tokens >= 1.0:
                bucket.tokens -= 1.0
                return True, {
                    "remaining": int(bucket.tokens),
                    "limit": bucket.max_tokens,
                }
            else:
                # Calculate time until next token is available
                tokens_needed = 1.0 - bucket.tokens
                retry_after = tokens_needed / bucket.refill_rate
                return False, {
                    "retry_after_seconds": round(retry_after, 1),
                    "limit": bucket.max_tokens,
                }

    def cleanup_inactive(self, max_idle_seconds: int = 86400) -> int:
        """Remove buckets that have been inactive. Returns count removed."""
        now = time.time()
        removed = 0
        with self._global_lock:
            inactive_users = [
                uid for uid, bucket in self._buckets.items()
                if now - bucket.last_refill > max_idle_seconds
            ]
            for uid in inactive_users:
                del self._buckets[uid]
                del self._locks[uid]
                removed += 1
        return removed
```

## Extensions for Advanced Learners

1. **Distributed rate limiting**: Extend the rate limiter to work across multiple server instances using Redis. Use Redis INCR with TTL for a simpler approach, or use Redis + Lua scripts for the token bucket algorithm.

2. **Sliding window log**: Implement a sliding window log rate limiter that provides more accurate rate limiting than the token bucket. Store request timestamps in a sorted data structure and count requests within the window.

3. **Adaptive rate limiting**: Implement a rate limiter that adjusts limits based on system load. When LLM provider latency increases, automatically reduce per-user limits to protect the system.

4. **Rate limit headers**: Add standard HTTP rate limit headers to the response: `X-RateLimit-Limit`, `X-RateLimit-Remaining`, `X-RateLimit-Reset`.

## Interview Relevance

**How commonly asked**: HIGH

**Common interview variations**:
- "Design a rate limiter for a URL shortener" (same algorithm, different context)
- "Implement a sliding window rate limiter" (variant algorithm)
- "How would you rate limit across multiple servers?" (distributed systems extension)

**What interviewers look for**:
- Understanding of the algorithm (not just memorized code)
- Thread safety considerations
- Edge case handling (concurrent requests, clock skew)
- Ability to discuss tradeoffs (token bucket vs. sliding window vs. fixed window)
- Knowledge of distributed rate limiting (Redis, memcached)

**Follow-up questions to expect**:
- "What happens if two requests arrive at exactly the same time?"
- "How would you persist rate limit state across server restarts?"
- "What is the memory footprint for 1 million users?"
- "How do you handle a sudden burst of requests from a single user?"
