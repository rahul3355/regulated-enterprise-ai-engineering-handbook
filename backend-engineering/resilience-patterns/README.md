# Resilience Patterns for Banking GenAI Platforms

## Why Resilience Patterns

In a banking GenAI platform, failures are inevitable: network timeouts, database outages, downstream service failures, model inference errors. Without resilience patterns, a single failure cascades across the system, taking down all services. Resilience patterns ensure graceful degradation, automatic recovery, and containment of failures.

## Circuit Breaker Pattern

### How It Works

```
Circuit States:

CLOSED (Normal)
- Requests flow through normally
- Tracks failure count/rate
- Threshold reached -> OPEN

OPEN (Failing)
- All requests fail immediately (no downstream call)
- After timeout period -> HALF-OPEN

HALF-OPEN (Testing)
- Allow limited requests through
- Success -> CLOSED (recovered)
- Failure -> OPEN (still broken)
```

### Implementation

```python
import time
from enum import Enum
from threading import Lock
from dataclasses import dataclass

class CircuitState(Enum):
    CLOSED = 'closed'
    OPEN = 'open'
    HALF_OPEN = 'half_open'

@dataclass
class CircuitBreakerConfig:
    failure_threshold: int = 5       # Failures before opening
    success_threshold: int = 3        # Successes before closing from half-open
    timeout_seconds: int = 60         # Time before trying half-open
    failure_rate_threshold: float = 0.5  # 50% failure rate triggers open

class CircuitBreaker:
    def __init__(self, name: str, config: CircuitBreakerConfig):
        self.name = name
        self.config = config
        self.state = CircuitState.CLOSED
        self.failure_count = 0
        self.success_count = 0
        self.last_failure_time = None
        self._lock = Lock()

    def call(self, func, *args, **kwargs):
        with self._lock:
            if self.state == CircuitState.OPEN:
                if self._should_transition_to_half_open():
                    self.state = CircuitState.HALF_OPEN
                    self.success_count = 0
                else:
                    raise CircuitOpenError(
                        f'Circuit {self.name} is OPEN. '
                        f'Retry after {self.config.timeout_seconds}s'
                    )

        try:
            result = func(*args, **kwargs)

            with self._lock:
                self._on_success()

            return result

        except Exception as e:
            with self._lock:
                self._on_failure()

            raise

    def _on_success(self):
        if self.state == CircuitState.HALF_OPEN:
            self.success_count += 1
            if self.success_count >= self.config.success_threshold:
                self.state = CircuitState.CLOSED
                self.failure_count = 0
                self.success_count = 0
        else:
            self.failure_count = 0

    def _on_failure(self):
        self.failure_count += 1
        self.last_failure_time = time.time()

        if self.state == CircuitState.HALF_OPEN:
            self.state = CircuitState.OPEN
        elif self.failure_count >= self.config.failure_threshold:
            self.state = CircuitState.OPEN

    def _should_transition_to_half_open(self) -> bool:
        if self.last_failure_time is None:
            return False
        return (time.time() - self.last_failure_time) >= self.config.timeout_seconds


class CircuitOpenError(Exception):
    """Raised when circuit breaker is open and request is rejected."""
    pass
```

### Banking Usage Example

```python
# Circuit breaker for external payment gateway
payment_gateway_circuit = CircuitBreaker(
    name='payment-gateway',
    config=CircuitBreakerConfig(
        failure_threshold=5,
        timeout_seconds=30,
        success_threshold=2,
    ),
)

def initiate_external_payment(payment_data: dict) -> dict:
    """Call external payment gateway with circuit breaker."""
    try:
        return payment_gateway_circuit.call(
            requests.post,
            'https://gateway.external-bank.com/api/payments',
            json=payment_data,
            timeout=10,
        ).json()

    except CircuitOpenError:
        # Gateway is down - queue payment for later processing
        logger.warning('Payment gateway circuit open, queuing payment')
        PaymentQueue.enqueue(payment_data)
        return {'status': 'queued', 'reason': 'gateway_unavailable'}

    except requests.exceptions.Timeout:
        # Timeout counts as failure in circuit breaker
        raise PaymentGatewayTimeoutError('External gateway timed out')
```

### Monitoring

```python
# Expose circuit breaker metrics
class CircuitBreakerMetrics:
    def __init__(self, circuit: CircuitBreaker):
        self.circuit = circuit

    def get_metrics(self) -> dict:
        return {
            'circuit_name': self.circuit.name,
            'state': self.circuit.state.value,
            'failure_count': self.circuit.failure_count,
            'success_count': self.circuit.success_count,
            'last_failure_time': self.circuit.last_failure_time,
        }

# Prometheus metrics
from prometheus_client import Gauge

circuit_state = Gauge('circuit_breaker_state', 'Current circuit state', ['circuit_name'])
circuit_failures = Gauge('circuit_breaker_failures', 'Failure count', ['circuit_name'])

def update_metrics(circuit: CircuitBreaker):
    state_map = {'closed': 0, 'open': 1, 'half_open': 2}
    circuit_state.labels(circuit.name).set(state_map[circuit.state.value])
    circuit_failures.labels(circuit.name).set(circuit.failure_count)
```

## Retry Pattern

### Exponential Backoff with Jitter

```python
import random
import time
from functools import wraps

def retry_with_backoff(
    max_retries: int = 3,
    base_delay: float = 1.0,
    max_delay: float = 60.0,
    exponential_base: float = 2.0,
    jitter: bool = True,
    retryable_exceptions: tuple = (Exception,),
):
    """Decorator for retry with exponential backoff."""
    def decorator(func):
        @wraps(func)
        def wrapper(*args, **kwargs):
            last_exception = None

            for attempt in range(max_retries + 1):
                try:
                    return func(*args, **kwargs)

                except retryable_exceptions as e:
                    last_exception = e

                    if attempt == max_retries:
                        raise

                    # Calculate delay
                    delay = min(
                        base_delay * (exponential_base ** attempt),
                        max_delay,
                    )

                    # Add jitter to prevent thundering herd
                    if jitter:
                        delay = random.uniform(delay * 0.5, delay * 1.5)

                    logger.warning(
                        f'Retry {attempt + 1}/{max_retries} for {func.__name__}',
                        extra={'error': str(e), 'delay': delay},
                    )

                    time.sleep(delay)

            raise last_exception

        return wrapper
    return decorator


# Usage
@retry_with_backoff(
    max_retries=3,
    base_delay=1.0,
    max_delay=30.0,
    retryable_exceptions=(requests.exceptions.RequestException,),
)
def call_ai_service(prompt: str) -> str:
    response = requests.post(
        'http://ai-service:8080/v1/chat',
        json={'prompt': prompt},
        timeout=60,
    )
    response.raise_for_status()
    return response.json()['response']
```

### Retry Decision Matrix

```python
def should_retry(status_code: int, error: Exception) -> bool:
    """Determine if error is retryable."""
    # Retryable: server errors, timeouts
    # Not retryable: client errors (4xx)

    if isinstance(error, requests.exceptions.Timeout):
        return True  # Timeout is retryable

    if isinstance(error, requests.exceptions.ConnectionError):
        return True  # Connection error is retryable

    if hasattr(error, 'response'):
        status = error.response.status_code
        if status >= 500:
            return True  # Server error is retryable
        if status == 429:
            return True  # Rate limited, retry after delay
        if status == 408:
            return True  # Request timeout
        if status >= 400:
            return False  # Client error is NOT retryable

    return False  # Unknown error, don't retry
```

## Bulkhead Pattern

### Isolate Resources

```python
from concurrent.futures import ThreadPoolExecutor, TimeoutError

class BulkheadPool:
    """Isolate concurrent requests to a service."""

    def __init__(self, name: str, max_concurrent: int, timeout: float = 30.0):
        self.name = name
        self.executor = ThreadPoolExecutor(max_workers=max_concurrent)
        self.timeout = timeout

    def execute(self, func, *args, **kwargs):
        future = self.executor.submit(func, *args, **kwargs)
        try:
            return future.result(timeout=self.timeout)
        except TimeoutError:
            future.cancel()
            raise BulkheadTimeoutError(
                f'Bulkhead {self.name} timed out after {self.timeout}s'
            )

    @property
    def active_count(self) -> int:
        return len(self.executor._threads)

    def is_at_capacity(self) -> bool:
        return self.active_count >= self.executor._max_workers


# Banking usage: separate pools for different services
payment_pool = BulkheadPool('payments', max_concurrent=50, timeout=10)
ai_pool = BulkheadPool('ai-inference', max_concurrent=20, timeout=60)
notification_pool = BulkheadPool('notifications', max_concurrent=100, timeout=5)

# Payment requests isolated to 50 concurrent
def process_payment(payment_data: dict):
    return payment_pool.execute(
        self.payment_gateway.submit,
        payment_data,
    )

# AI requests isolated to 20 concurrent
def run_ai_inference(prompt: str):
    return ai_pool.execute(
        self.ai_client.generate,
        prompt,
    )
```

### Failure Isolation

```python
# Without bulkhead: one slow service consumes all threads
# With bulkhead: slow service only affects its own pool

# Scenario: AI service becomes slow (10s response time)

# Without bulkhead:
# - All worker threads blocked waiting for AI
# - Payment processing also blocked
# - System cascades to full outage

# With bulkhead:
# - AI pool: 20 threads blocked (AI only)
# - Payment pool: 50 threads free (payments continue)
# - Notification pool: 100 threads free (notifications continue)
# - Failure contained to AI service
```

## Timeout Pattern

### Timeout Hierarchy

```python
# Timeouts at every layer with decreasing budgets

TIMEOUT_HIERARCHY = {
    'api_gateway': 30,          # Total request timeout
    'service_call': 10,         # Inter-service call timeout
    'database_query': 5,        # Database query timeout
    'cache_lookup': 1,          # Cache lookup timeout
    'dns_resolution': 0.5,      # DNS resolution timeout
}

# API endpoint timeout budget:
# - DNS: 0.5s
# - Cache: 1.0s
# - Database: 5.0s
# - Service calls: 10.0s (parallel)
# - Total: 30.0s (includes processing time)
```

### Implementation

```python
import signal

class TimeoutError(Exception):
    pass

def timeout_handler(signum, frame):
    raise TimeoutError('Operation timed out')

def with_timeout(func, seconds: float, *args, **kwargs):
    """Execute function with timeout."""
    # Unix-only: use signal.alarm
    signal.signal(signal.SIGALRM, timeout_handler)
    signal.setitimer(signal.ITIMER_REAL, seconds)

    try:
        result = func(*args, **kwargs)
        return result
    finally:
        signal.setitimer(signal.ITIMER_REAL, 0)


# Cross-platform: use threading
from concurrent.futures import ThreadPoolExecutor, TimeoutError as FuturesTimeoutError

def with_timeout_cross_platform(func, seconds: float, *args, **kwargs):
    with ThreadPoolExecutor(max_workers=1) as executor:
        future = executor.submit(func, *args, **kwargs)
        try:
            return future.result(timeout=seconds)
        except FuturesTimeoutError:
            future.cancel()
            raise TimeoutError(f'Operation timed out after {seconds}s')
```

## Fallback Pattern

### Graceful Degradation

```python
class FallbackChain:
    """Try primary, fall back to alternatives."""

    def __init__(self, *strategies):
        self.strategies = strategies

    def execute(self, *args, **kwargs):
        last_error = None

        for strategy in self.strategies:
            try:
                result = strategy(*args, **kwargs)
                return {'source': strategy.__name__, 'result': result}
            except Exception as e:
                last_error = e
                logger.warning(f'{strategy.__name__} failed: {e}')

        raise FallbackExhaustedError(
            f'All fallback strategies failed. Last error: {last_error}'
        )


# Banking example: exchange rate lookup
def get_exchange_rate_live(currency_pair: str) -> Decimal:
    """Get live exchange rate from API."""
    response = requests.get(
        f'https://fx-api.bank.internal/rates/{currency_pair}',
        timeout=5,
    )
    response.raise_for_status()
    return Decimal(response.json()['rate'])

def get_exchange_rate_cache(currency_pair: str) -> Decimal:
    """Get rate from cache."""
    cached = redis_client.get(f'fx_rate:{currency_pair}')
    if cached:
        return Decimal(cached)
    raise CacheMissError('Rate not in cache')

def get_exchange_rate_static(currency_pair: str) -> Decimal:
    """Get yesterday's closing rate as last resort."""
    rate = static_rates.get(currency_pair)
    if rate:
        return Decimal(rate)
    raise FallbackExhaustedError(f'No rate available for {currency_pair}')


# Usage
rate_resolver = FallbackChain(
    get_exchange_rate_live,
    get_exchange_rate_cache,
    get_exchange_rate_static,
)

result = rate_resolver.execute('USD/EUR')
logger.info(f'Exchange rate from {result["source"]}')
```

## GenAI-Specific Resilience

### Model Fallback Chain

```python
class ModelFallback:
    """Fallback between AI models based on availability and cost."""

    def __init__(self):
        self.strategies = FallbackChain(
            self.try_gpt4,
            self.try_claude,
            self.try_local_model,
        )

    def generate_response(self, prompt: str) -> dict:
        return self.strategies.execute(prompt)

    def try_gpt4(self, prompt: str) -> dict:
        """Primary: GPT-4 (best quality, highest cost)."""
        response = openai.ChatCompletion.create(
            model='gpt-4',
            messages=[{'role': 'user', 'content': prompt}],
            timeout=30,
        )
        return {'model': 'gpt-4', 'response': response.choices[0].message.content}

    def try_claude(self, prompt: str) -> dict:
        """Secondary: Claude (good quality, medium cost)."""
        response = anthropic.Anthropic().messages.create(
            model='claude-3-opus',
            messages=[{'role': 'user', 'content': prompt}],
            timeout=30,
        )
        return {'model': 'claude', 'response': response.content[0].text}

    def try_local_model(self, prompt: str) -> dict:
        """Tertiary: Local model (acceptable quality, no cost)."""
        response = local_model.generate(prompt, timeout=10)
        return {'model': 'local', 'response': response}
```

## Common Mistakes

1. **Retry storms**: Retrying without delay causes cascading overload
2. **No jitter**: All clients retry at same time, thundering herd
3. **Retrying 4xx errors**: Client errors won't succeed on retry
4. **Circuit breaker too sensitive**: Opens on transient errors
5. **Circuit breaker too lenient**: Keeps sending requests to failing service
6. **No bulkhead isolation**: One slow service takes down everything
7. **Timeout too long**: Resources held waiting for unresponsive service
8. **Fallback not tested**: Fallback path never tested, fails when needed

## Interview Questions

1. Design a circuit breaker for a payment gateway with specific thresholds.
2. How do you prevent retry storms when 1000 clients retry after a timeout?
3. When should you NOT retry a failed request?
4. How does bulkhead pattern differ from rate limiting?
5. Design a fallback chain for a document analysis service.

## Cross-References

- [[api-design/README.md]] - Timeout configuration for API calls
- [[microservices/README.md]] - Resilience in inter-service communication
- [[distributed-systems/README.md]] - Handling partial failures
- [[caching/README.md]] - Cache as fallback when primary is unavailable
- `../python/async-python.md` - Async retry implementations
