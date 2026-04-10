# Caching for Banking GenAI Platforms

## Why Caching

Caching reduces latency, decreases database load, and improves system resilience. In banking, cache hit ratios of 90%+ for read-heavy workloads (account lookups, product catalogs, exchange rates) translate to 10x throughput improvement. For GenAI platforms, caching embeddings, model outputs, and document analysis results saves expensive inference costs.

## Redis Fundamentals

### Data Structures

```python
import redis
import json
from decimal import Decimal

redis_client = redis.Redis(
    host='redis-primary',
    port=6379,
    db=0,
    decode_responses=True,
    socket_connect_timeout=5,
    socket_timeout=5,
    retry_on_timeout=True,
)

# String - Simple key-value
redis_client.set('account:ACC-001:balance', '15000.00', ex=300)
balance = redis_client.get('account:ACC-001:balance')

# Hash - Structured data
redis_client.hset('account:ACC-001', mapping={
    'name': 'John Doe',
    'type': 'checking',
    'status': 'active',
    'currency': 'USD',
})
account = redis_client.hgetall('account:ACC-001')

# Set - Unique members
redis_client.sadd('account:ACC-001:permissions', 'read', 'transfer', 'view_statements')
permissions = redis_client.smembers('account:ACC-001:permissions')

# Sorted Set - Ranked data
redis_client.zadd('leaderboard:top_accounts', {
    'ACC-001': 150000.00,
    'ACC-002': 98000.00,
    'ACC-003': 75000.00,
})
top_10 = redis_client.zrevrange('leaderboard:top_accounts', 0, 9, withscores=True)

# List - Queue operations
redis_client.rpush('queue:payments', json.dumps(payment_data))
next_payment = redis_client.lpop('queue:payments')
```

## Cache-Aside Pattern (Most Common)

```python
class AccountService:
    def __init__(self, db: Database, cache: redis.Redis):
        self.db = db
        self.cache = cache
        self.cache_ttl = 300  # 5 minutes

    def get_account(self, account_id: str) -> dict:
        cache_key = f'account:{account_id}'

        # 1. Try cache first
        cached = self.cache.get(cache_key)
        if cached:
            return json.loads(cached)

        # 2. Cache miss - read from database
        account = self.db.query(Account).get(account_id)
        if not account:
            raise AccountNotFoundError(account_id)

        # 3. Populate cache
        self.cache.set(
            cache_key,
            json.dumps(account.to_dict()),
            ex=self.cache_ttl,
        )

        return account.to_dict()

    def update_account(self, account_id: str, data: dict) -> dict:
        # 1. Update database first
        account = self.db.update_account(account_id, data)

        # 2. Invalidate cache (write-through invalidation)
        cache_key = f'account:{account_id}'
        self.cache.delete(cache_key)

        return account.to_dict()
```

## Cache Invalidation Strategies

### Time-Based Expiration

```python
# TTL selection by data volatility
CACHE_TTLS = {
    'exchange_rates': 60,       # 1 minute - highly volatile
    'account_balance': 300,     # 5 minutes - moderate volatility
    'customer_profile': 3600,   # 1 hour - low volatility
    'product_catalog': 86400,   # 24 hours - static
    'branch_locations': 604800, # 7 days - rarely changes
}
```

### Event-Driven Invalidation

```python
# Invalidate cache on database changes
def on_account_updated(event: dict):
    """Event handler triggered by database update."""
    account_id = event['account_id']
    cache_key = f'account:{account_id}'
    redis_client.delete(cache_key)

    # Also invalidate related caches
    redis_client.delete(f'account:{account_id}:transactions')
    redis_client.delete(f'account:{account_id}:statements')

# Subscribe to database change events
# Using Debezium CDC -> Kafka -> cache invalidation
```

### Versioned Cache Keys

```python
# Use version suffix for cache keys
def get_account_cache_key(account_id: str, version: str) -> str:
    return f'account:v{version}:{account_id}'

# Update version when schema changes
CURRENT_CACHE_VERSION = '3'

# Old version keys naturally expire
# No need for explicit invalidation on schema change
```

## Cache Stampede Prevention

### Problem

When a popular cache key expires, hundreds of concurrent requests hit the database simultaneously, causing a spike that can crash the database.

### Solution 1: Mutex Lock (Single Flight)

```python
import threading

class CacheStampedeProtector:
    def __init__(self, cache: redis.Redis, db: Database):
        self.cache = cache
        self.db = db
        self._locks = {}

    def get_with_lock(self, key: str, ttl: int = 300) -> dict:
        # Try cache first
        cached = self.cache.get(key)
        if cached:
            return json.loads(cached)

        # Acquire lock for this key
        lock_key = f'lock:{key}'
        acquired = self.cache.set(lock_key, '1', nx=True, ex=10)

        if acquired:
            try:
                # This request loads from database
                data = self.db.query(key)
                self.cache.set(key, json.dumps(data), ex=ttl)
                return data
            finally:
                self.cache.delete(lock_key)
        else:
            # Wait for the lock holder to populate cache
            time.sleep(0.1)
            return self.get_with_lock(key, ttl)  # Retry
```

### Solution 2: Probabilistic Early Expiration

```python
import random

def get_with_early_refresh(key: str, ttl: int, jitter_pct: float = 0.1) -> dict:
    """Refresh cache before it expires to prevent stampede."""
    cached = redis_client.get(key)

    if cached:
        data = json.loads(cached)
        # Check if TTL is within jitter window
        remaining_ttl = redis_client.ttl(key)
        if remaining_ttl < ttl * jitter_pct:
            # Randomly select one requestor to refresh
            if random.random() < jitter_pct:
                # This requestor refreshes in background
                background_refresh(key)
        return data

    # Cache miss - load and store
    data = load_from_database(key)
    redis_client.set(key, json.dumps(data), ex=ttl)
    return data
```

### Solution 3: Background Refresh

```python
import asyncio

class BackgroundRefreshCache:
    def __init__(self):
        self.cache = {}
        self._refresh_tasks = set()

    async def get(self, key: str) -> dict:
        if key not in self.cache:
            return await self._load_and_cache(key)

        entry = self.cache[key]
        if entry.is_stale():
            # Refresh in background, return stale data
            task = asyncio.create_task(self._load_and_cache(key))
            self._refresh_tasks.add(task)
            task.add_done_callback(self._refresh_tasks.discard)

        return entry.data

    async def _load_and_cache(self, key: str) -> dict:
        data = await load_from_database(key)
        self.cache[key] = CacheEntry(data, ttl=300)
        return data
```

## Multi-Level Caching

### L1 (In-Process) + L2 (Redis) + L3 (Database)

```python
from cachetools import TTLCache

class MultiLevelCache:
    def __init__(self):
        # L1: In-process memory cache (fastest, smallest)
        self.l1 = TTLCache(maxsize=1000, ttl=30)

        # L2: Redis cache (fast, medium size)
        self.l2 = redis.Redis(host='redis-primary', port=6379)

        # L3: Database (slow, full data)
        self.l3 = Database()

    def get_account(self, account_id: str) -> dict:
        # L1 check
        l1_key = f'account:{account_id}'
        if l1_key in self.l1:
            return self.l1[l1_key]

        # L2 check
        l2_key = f'account:{account_id}'
        cached = self.l2.get(l2_key)
        if cached:
            data = json.loads(cached)
            self.l1[l1_key] = data  # Promote to L1
            return data

        # L3: Database
        data = self.l3.query_account(account_id)

        # Write-through to L2
        self.l2.set(l2_key, json.dumps(data), ex=300)

        # Promote to L1
        self.l1[l1_key] = data

        return data
```

## GenAI-Specific Caching

### Embedding Cache

```python
# Cache document embeddings to avoid recomputation
class EmbeddingCache:
    def __init__(self, redis_client: redis.Redis):
        self.redis = redis_client
        self.model_version = 'v2'

    def get_embedding(self, text: str, model: str = 'default') -> list[float] | None:
        """Check cache for pre-computed embedding."""
        # Hash the text for cache key
        import hashlib
        text_hash = hashlib.sha256(text.encode()).hexdigest()
        cache_key = f'embedding:{self.model_version}:{model}:{text_hash}'

        cached = self.redis.get(cache_key)
        if cached:
            return json.loads(cached)
        return None

    def set_embedding(self, text: str, embedding: list[float], model: str = 'default'):
        """Cache computed embedding."""
        import hashlib
        text_hash = hashlib.sha256(text.encode()).hexdigest()
        cache_key = f'embedding:{self.model_version}:{model}:{text_hash}'

        # Store with long TTL (embeddings don't expire quickly)
        self.redis.set(
            cache_key,
            json.dumps(embedding),
            ex=86400 * 30,  # 30 days
        )

    def invalidate_model(self, model: str):
        """Invalidate all embeddings for a model version."""
        pattern = f'embedding:*:{model}:*'
        for key in self.redis.scan_iter(match=pattern):
            self.redis.delete(key)
```

### LLM Response Cache

```python
# Cache deterministic LLM responses
class LLMResponseCache:
    def __init__(self, redis_client: redis.Redis):
        self.redis = redis_client

    def _cache_key(self, prompt: str, model: str, params: dict) -> str:
        import hashlib
        payload = f'{model}:{prompt}:{json.dumps(params, sort_keys=True)}'
        return f'llm:{hashlib.sha256(payload.encode()).hexdigest()}'

    def get(self, prompt: str, model: str, params: dict) -> str | None:
        key = self._cache_key(prompt, model, params)
        return self.redis.get(key)

    def set(self, prompt: str, model: str, params: dict, response: str):
        """Only cache when temperature=0 (deterministic)."""
        if params.get('temperature', 0) > 0:
            return  # Don't cache non-deterministic responses

        key = self._cache_key(prompt, model, params)
        self.redis.set(key, response, ex=3600)
```

## Redis Cluster and High Availability

### Cluster Topology

```
# Production Redis setup
# 3 master nodes, 3 replica nodes (6 total)

# Master nodes (handle writes)
redis-master-0:6379  -> slots 0-5460
redis-master-1:6379  -> slots 5461-10922
redis-master-2:6379  -> slots 10923-16383

# Replica nodes (handle reads, failover)
redis-replica-0:6379 -> replica of master-0
redis-replica-1:6379 -> replica of master-1
redis-replica-2:6379 -> replica of master-2

# Automatic failover if master fails
# Sentinel monitors health and promotes replica
```

### Connection Pool Configuration

```python
import redis

# Production connection pool
pool = redis.ConnectionPool(
    host='redis-sentinel',
    port=26379,
    max_connections=50,
    socket_connect_timeout=5,
    socket_timeout=5,
    retry_on_timeout=True,
    health_check_interval=30,
)

redis_client = redis.Redis(connection_pool=pool)
```

## Security Considerations

### Redis Security

```python
# TLS encryption for data in transit
redis_client = redis.Redis(
    host='redis-primary',
    port=6380,  # TLS port
    ssl=True,
    ssl_ca_certs='/path/to/ca-cert.pem',
)

# Authentication
redis_client = redis.Redis(
    host='redis-primary',
    password=os.environ['REDIS_PASSWORD'],
)

# Never cache sensitive data
SENSITIVE_PATTERNS = [
    'password',
    'ssn',
    'card_number',
    'cvv',
    'pin',
    'secret',
]

def is_cacheable(data: dict) -> bool:
    for key in data:
        if any(pattern in key.lower() for pattern in SENSITIVE_PATTERNS):
            return False
    return True
```

## Performance Tuning

### Redis Configuration

```
# redis.conf for production
maxmemory 4gb
maxmemory-policy allkeys-lru  # Evict least recently used
save 900 1    # Snapshot every 15 min if 1 key changed
save 300 10   # Snapshot every 5 min if 10 keys changed
save 60 10000 # Snapshot every minute if 10k keys changed

# Network
tcp-backlog 511
timeout 300
tcp-keepalive 60

# Persistence
appendonly yes
appendfsync everysec  # Balance between performance and durability
```

### Monitoring

```python
# Cache hit ratio tracking
class CacheMetrics:
    def __init__(self):
        self.hits = 0
        self.misses = 0

    def record_hit(self):
        self.hits += 1

    def record_miss(self):
        self.misses += 1

    def hit_ratio(self) -> float:
        total = self.hits + self.misses
        return self.hits / total if total > 0 else 0

# Alert thresholds
# Hit ratio < 80% -> warning
# Hit ratio < 60% -> critical
# Memory usage > 80% -> warning
# Memory usage > 95% -> critical
```

## Common Mistakes

1. **Caching sensitive data**: PII, card numbers, passwords in cache
2. **No TTL**: Cache entries never expire, stale data served indefinitely
3. **Cache stampede**: No protection against thundering herd on cache miss
4. **Write-back caching**: Cache updated before database, data loss on crash
5. **Oversized values**: Storing large JSON blobs (> 1MB) in Redis
6. **Ignoring memory limits**: Redis OOM when maxmemory reached
7. **Single Redis instance**: No failover, single point of failure

## Interview Questions

1. How do you prevent cache stampede for an endpoint receiving 10k requests/second?
2. Design a caching strategy for exchange rates that update every 60 seconds.
3. What happens when Redis runs out of memory with `allkeys-lru` policy?
4. How do you ensure cache consistency when multiple services update the same data?
5. When would you use write-through vs cache-aside pattern?

## Cross-References

- [[api-design/README.md]] - API response caching strategies
- [[resilience-patterns/README.md]] - Fallback to cache when services are down
- [[performance/README.md]] - Caching for performance optimization
- `../python/async-python.md` - Async Redis operations
- `../go/concurrency.md` - Goroutine-safe cache implementations
