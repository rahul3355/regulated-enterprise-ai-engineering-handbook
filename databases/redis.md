# Redis for Caching, Sessions, and Rate Limiting

## Overview

Redis is an in-memory data store used in banking platforms for low-latency operations: caching frequently accessed data, session management, rate limiting, and as an online feature store for ML models. This guide covers Redis deployment patterns, data structures, and banking-specific use cases.

## Redis Use Cases in Banking

```
1. Caching Layer
   - Account balance cache (reduces DB load)
   - Product catalog cache
   - Customer profile cache
   - API response cache

2. Session Management
   - User session storage
   - API token validation
   - CSRF token storage

3. Rate Limiting
   - API rate limiting (token bucket)
   - Login attempt limiting
   - Transaction frequency limiting

4. Real-time Counters
   - Transaction count per window
   - Failed login attempts
   - Active user count

5. Feature Store (Online)
   - Real-time ML features
   - Fraud detection features
   - Recommendation features

6. Distributed Locks
   - Prevent duplicate processing
   - Leader election
   - Job coordination
```

## Caching Patterns

```python
"""Redis caching patterns for banking."""
import redis
import json
import logging
from typing import Optional
from datetime import timedelta

logger = logging.getLogger(__name__)

class BankingCache:
    """Redis-based cache for banking data."""
    
    def __init__(self, redis_url: str, default_ttl: int = 300):
        self.client = redis.from_url(redis_url, decode_responses=True)
        self.default_ttl = default_ttl
    
    def get_account_balance(self, account_id: int) -> Optional[float]:
        """Get cached account balance."""
        key = f"balance:{account_id}"
        cached = self.client.get(key)
        if cached:
            return float(cached)
        return None
    
    def set_account_balance(self, account_id: int, balance: float, ttl: int = 60):
        """Cache account balance with short TTL."""
        key = f"balance:{account_id}"
        self.client.setex(key, ttl, str(balance))
    
    def invalidate_account_balance(self, account_id: int):
        """Remove cached balance after transaction."""
        key = f"balance:{account_id}"
        self.client.delete(key)
    
    def get_or_set(self, key: str, fn, ttl: int = None) -> any:
        """Cache-aside pattern: Get from cache or compute and cache."""
        ttl = ttl or self.default_ttl
        cached = self.client.get(key)
        if cached:
            return json.loads(cached)
        
        # Compute value
        value = fn()
        
        # Cache it
        self.client.setex(key, ttl, json.dumps(value))
        
        return value
    
    def cache_customer_profile(self, customer_id: int, profile: dict, ttl: int = 600):
        """Cache customer profile as JSON."""
        key = f"customer:{customer_id}:profile"
        self.client.setex(key, ttl, json.dumps(profile))
    
    def get_customer_profile(self, customer_id: int) -> Optional[dict]:
        """Get cached customer profile."""
        key = f"customer:{customer_id}:profile"
        cached = self.client.get(key)
        if cached:
            return json.loads(cached)
        return None
```

## Rate Limiting

```python
"""Rate limiting with Redis for banking APIs."""
import time
import redis
from typing import Tuple

class RateLimiter:
    """Redis-based rate limiter with multiple strategies."""
    
    def __init__(self, redis_client: redis.Redis):
        self.client = redis_client
    
    def token_bucket(
        self, 
        key: str, 
        max_tokens: int, 
        refill_rate: float,
        cost: int = 1
    ) -> Tuple[bool, dict]:
        """
        Token bucket rate limiting.
        Returns (allowed, info).
        """
        now = time.time()
        pipe = self.client.pipeline(True)
        
        pipe.multi()
        pipe.hgetall(f"ratelimit:{key}")
        pipe.execute()
        
        bucket = self.client.hgetall(f"ratelimit:{key}")
        
        if not bucket:
            # First request: full bucket
            self.client.hset(f"ratelimit:{key}", mapping={
                'tokens': str(max_tokens - cost),
                'last_refill': str(now),
            })
            self.client.expire(f"ratelimit:{key}", int(max_tokens / refill_rate) + 60)
            return True, {'tokens_remaining': max_tokens - cost, 'max_tokens': max_tokens}
        
        last_refill = float(bucket['last_refill'])
        tokens = float(bucket['tokens'])
        
        # Refill tokens
        elapsed = now - last_refill
        tokens_to_add = elapsed * refill_rate
        tokens = min(max_tokens, tokens + tokens_to_add)
        
        if tokens >= cost:
            tokens -= cost
            self.client.hset(f"ratelimit:{key}", mapping={
                'tokens': str(tokens),
                'last_refill': str(now),
            })
            return True, {'tokens_remaining': tokens, 'max_tokens': max_tokens}
        else:
            # Update last_refill time but deny
            self.client.hset(f"ratelimit:{key}", mapping={
                'last_refill': str(now),
            })
            return False, {'tokens_remaining': tokens, 'max_tokens': max_tokens, 'retry_after': (cost - tokens) / refill_rate}
    
    def sliding_window(self, key: str, max_requests: int, window_seconds: int) -> bool:
        """Sliding window rate limiting using sorted sets."""
        now = time.time()
        window_start = now - window_seconds
        
        pipe = self.client.pipeline(True)
        pipe.zremrangebyscore(f"ratelimit:{key}", 0, window_start)
        pipe.zcard(f"ratelimit:{key}")
        pipe.zadd(f"ratelimit:{key}", {str(now): now})
        pipe.expire(f"ratelimit:{key}", window_seconds + 60)
        results = pipe.execute()
        
        current_count = results[1]
        return current_count <= max_requests
    
    def fixed_window_counter(self, key: str, max_requests: int, window_seconds: int) -> bool:
        """Fixed window counter rate limiting."""
        now = time.time()
        window_key = f"ratelimit:{key}:{int(now // window_seconds)}"
        
        pipe = self.client.pipeline(True)
        pipe.get(window_key)
        pipe.ttl(window_key)
        results = pipe.execute()
        
        current_count = int(results[0] or 0)
        
        if current_count >= max_requests:
            return False
        
        pipe = self.client.pipeline(True)
        pipe.incr(window_key)
        ttl = results[1]
        if ttl < 0:
            pipe.expire(window_key, window_seconds)
        pipe.execute()
        
        return True

# Banking API rate limiting example
# 100 requests per minute per user
# 1000 requests per hour per IP

def check_api_rate_limit(user_id: int, ip: str, limiter: RateLimiter) -> bool:
    """Check rate limits for API request."""
    # Per-user limit
    user_allowed, _ = limiter.sliding_window(
        f"user:{user_id}", 
        max_requests=100, 
        window_seconds=60
    )
    
    if not user_allowed:
        return False
    
    # Per-IP limit
    ip_allowed = limiter.fixed_window_counter(
        f"ip:{ip}",
        max_requests=1000,
        window_seconds=3600
    )
    
    return ip_allowed
```

## Distributed Locks

```python
"""Distributed locks with Redis for banking operations."""
import redis
import time
import uuid

class DistributedLock:
    """Redis-based distributed lock."""
    
    def __init__(self, client: redis.Redis, name: str, timeout: int = 30):
        self.client = client
        self.name = f"lock:{name}"
        self.timeout = timeout
        self.token = str(uuid.uuid4())
    
    def acquire(self, blocking: bool = True, retry_interval: float = 0.1) -> bool:
        """Acquire the lock."""
        while True:
            acquired = self.client.set(
                self.name,
                self.token,
                nx=True,  # Only set if not exists
                ex=self.timeout
            )
            
            if acquired:
                return True
            
            if not blocking:
                return False
            
            time.sleep(retry_interval)
    
    def release(self) -> bool:
        """Release the lock (only if we own it)."""
        # Use Lua script for atomic check-and-delete
        script = """
        if redis.call("get", KEYS[1]) == ARGV[1] then
            return redis.call("del", KEYS[1])
        else
            return 0
        end
        """
        result = self.client.eval(script, 1, self.name, self.token)
        return result == 1
    
    def __enter__(self):
        if not self.acquire():
            raise RuntimeError(f"Could not acquire lock: {self.name}")
        return self
    
    def __exit__(self, exc_type, exc_val, exc_tb):
        self.release()

# Usage: Prevent duplicate batch processing
def run_daily_batch_job():
    """Ensure only one instance of daily batch job runs."""
    with DistributedLock(redis_client, "daily_batch_job", timeout=3600):
        # Only one process executes this
        process_daily_etl()
```

## Cross-References

- **Redis Caching**: See [redis-caching.md](redis-caching.md) for caching patterns
- **Feature Stores**: See [feature-stores.md](../data-engineering/feature-stores.md) for ML features

## Interview Questions

1. **How do you implement rate limiting with Redis? Compare token bucket vs sliding window.**
2. **What is the cache-aside pattern? How do you handle cache invalidation?**
3. **How do distributed locks work in Redis? What are the failure modes?**
4. **Your Redis cache shows high memory usage. How do you diagnose and fix it?**
5. **How do you use Redis as an online feature store for ML models?**
6. **What happens when Redis runs out of memory? How do you configure eviction?**

## Checklist: Redis Production Readiness

- [ ] Redis cluster or sentinel for HA
- [ ] Memory limit configured with appropriate eviction policy
- [ ] Persistence enabled (RDB snapshots, AOF for durability)
- [ ] Authentication and network security configured
- [ ] Connection pooling in application
- [ ] Cache TTLs set for all keys
- [ ] Cache invalidation strategy defined
- [ ] Rate limiting thresholds tuned
- [ ] Memory usage monitored and alerted
- [ ] Key expiration monitored (avoid memory leaks)
- [ ] Redis Slow Log monitored
