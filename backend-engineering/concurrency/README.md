# Concurrency Models for Banking GenAI Platforms

## Why Concurrency Matters

Banking systems handle thousands of simultaneous transactions. GenAI platforms process concurrent inference requests across GPU resources. Without proper concurrency control, data corruption, race conditions, and resource contention cause financial losses and service degradation.

## Concurrency Challenges

### Race Conditions

```python
# RACE CONDITION: Lost update problem
# Two threads read same balance, both update, one update lost

def transfer_unsafe(from_account: str, to_account: str, amount: Decimal):
    # Thread A reads balance: 10000
    from_balance = db.get_balance(from_account)  # 10000
    # Thread B reads balance: 10000 (same value!)
    to_balance = db.get_balance(to_account)      # 5000

    # Thread A calculates: 10000 - 500 = 9500
    # Thread B calculates: 5000 + 500 = 5500

    # Thread A writes: 9500
    db.set_balance(from_account, from_balance - amount)  # 9500
    # Thread B writes: 5500 (overwrites correct value)
    db.set_balance(to_account, to_balance + amount)      # 5500

    # Result: from_account is correct (9500), to_account is correct (5500)
    # BUT if a third thread also transferred, updates could be lost
```

### Deadlocks

```python
# DEADLOCK: Two transactions lock resources in different order

# Transaction A: Lock account1, then account2
# Transaction B: Lock account2, then account1

# Timeline:
# T1: Transaction A locks account1
# T2: Transaction B locks account2
# T3: Transaction A waits for account2 (held by B)
# T4: Transaction B waits for account1 (held by A)
# -> DEADLOCK: Neither can proceed

# Solution: Always lock in consistent order (e.g., by account_id)
def transfer_safe(from_account: str, to_account: str, amount: Decimal):
    # Always lock lower ID first
    first, second = sorted([from_account, to_account])

    with db.lock_account(first):
        with db.lock_account(second):
            # Perform transfer
            execute_transfer(from_account, to_account, amount)
```

## Python Concurrency

### Threading (I/O Bound)

```python
import threading
from concurrent.futures import ThreadPoolExecutor

# Thread-safe counter
class ThreadSafeCounter:
    def __init__(self):
        self._value = 0
        self._lock = threading.Lock()

    def increment(self):
        with self._lock:
            self._value += 1

    @property
    def value(self):
        with self._lock:
            return self._value

# Thread pool for I/O operations
def fetch_account_balances(account_ids: list[str]) -> dict:
    """Fetch balances concurrently using thread pool."""
    results = {}
    lock = threading.Lock()

    def fetch_one(account_id: str):
        balance = db.query_balance(account_id)
        with lock:
            results[account_id] = balance

    with ThreadPoolExecutor(max_workers=10) as executor:
        executor.map(fetch_one, account_ids)

    return results
```

### Asyncio (I/O Bound, High Concurrency)

```python
import asyncio
import aiohttp

async def fetch_account_balance(session: aiohttp.ClientSession, account_id: str) -> dict:
    """Fetch single account balance asynchronously."""
    async with session.get(f'/v1/accounts/{account_id}') as response:
        return await response.json()

async def fetch_all_balances(account_ids: list[str]) -> list[dict]:
    """Fetch all balances concurrently."""
    async with aiohttp.ClientSession() as session:
        tasks = [
            fetch_account_balance(session, acc_id)
            for acc_id in account_ids
        ]
        return await asyncio.gather(*tasks, return_exceptions=True)

# Semaphore for concurrency limit
async def process_payments_with_limit(payments: list[dict], max_concurrent: int = 10):
    """Process payments with concurrency limit."""
    semaphore = asyncio.Semaphore(max_concurrent)

    async def process_one(payment: dict):
        async with semaphore:
            return await execute_payment(payment)

    tasks = [process_one(p) for p in payments]
    return await asyncio.gather(*tasks, return_exceptions=True)
```

### Multiprocessing (CPU Bound)

```python
from multiprocessing import Process, Queue, Lock
import numpy as np

def compute_risk_scores(documents: list[str], result_queue: Queue):
    """CPU-intensive risk scoring using multiprocessing."""
    for doc in documents:
        # Heavy computation
        score = calculate_complex_risk_score(doc)
        result_queue.put({'document': doc, 'score': score})

# Launch multiple processes
if __name__ == '__main__':
    queue = Queue()
    processes = []

    for i in range(4):  # 4 processes
        docs = document_batches[i]
        p = Process(target=compute_risk_scores, args=(docs, queue))
        p.start()
        processes.append(p)

    # Collect results
    results = []
    for _ in range(len(documents)):
        results.append(queue.get())

    # Wait for all processes
    for p in processes:
        p.join()
```

## Go Concurrency

### Goroutines and Channels

```go
// Goroutine for concurrent account processing
func FetchAccountBalances(accountIDs []string) (map[string]decimal.Decimal, error) {
    results := make(map[string]decimal.Decimal)
    var mu sync.Mutex
    var wg sync.WaitGroup
    errChan := make(chan error, len(accountIDs))

    for _, accountID := range accountIDs {
        wg.Add(1)
        go func(id string) {
            defer wg.Done()

            balance, err := db.GetBalance(id)
            if err != nil {
                errChan <- fmt.Errorf("account %s: %w", id, err)
                return
            }

            mu.Lock()
            results[id] = balance
            mu.Unlock()
        }(accountID)
    }

    wg.Wait()
    close(errChan)

    // Check for errors
    for err := range errChan {
        if err != nil {
            return results, err
        }
    }

    return results, nil
}
```

### Worker Pool Pattern

```go
// Worker pool for payment processing
type PaymentWorker struct {
    workerCount int
    jobQueue    chan Payment
    resultQueue chan PaymentResult
}

func NewPaymentWorker(workerCount int) *PaymentWorker {
    return &PaymentWorker{
        workerCount: workerCount,
        jobQueue:    make(chan Payment, 1000),
        resultQueue: make(chan PaymentResult, 1000),
    }
}

func (w *PaymentWorker) Start(ctx context.Context) {
    for i := 0; i < w.workerCount; i++ {
        go w.worker(ctx, i)
    }
}

func (w *PaymentWorker) worker(ctx context.Context, id int) {
    for {
        select {
        case <-ctx.Done():
            return
        case payment := <-w.jobQueue:
            result := processPayment(payment)
            w.resultQueue <- result
        }
    }
}

func (w *PaymentWorker) Submit(payment Payment) error {
    select {
    case w.jobQueue <- payment:
        return nil
    case <-time.After(5 * time.Second):
        return errors.New("job queue full, payment rejected")
    }
}
```

### Context for Cancellation

```go
// Context for timeout and cancellation
func ProcessPaymentWithTimeout(ctx context.Context, payment Payment) error {
    // Create context with timeout
    ctx, cancel := context.WithTimeout(ctx, 30*time.Second)
    defer cancel()

    // Propagate context to all downstream calls
    if err := validatePayment(ctx, payment); err != nil {
        return err
    }

    if err := debitAccount(ctx, payment.FromAccount, payment.Amount); err != nil {
        return err
    }

    if err := creditAccount(ctx, payment.ToAccount, payment.Amount); err != nil {
        // Compensate: credit back to source
        creditAccount(ctx, payment.FromAccount, payment.Amount)
        return err
    }

    return nil
}
```

## Java Concurrency

### ExecutorService

```java
// Thread pool for concurrent payment processing
public class PaymentProcessor {
    private final ExecutorService executor = new ThreadPoolExecutor(
        10,                          // core pool size
        50,                          // maximum pool size
        60L, TimeUnit.SECONDS,       // keep-alive time
        new LinkedBlockingQueue<>(1000),  // work queue
        new ThreadFactoryBuilder().setNameFormat("payment-%d").build(),
        new ThreadPoolExecutor.CallerRunsPolicy()  // rejection policy
    );

    public CompletableFuture<PaymentResult> processPayment(Payment payment) {
        return CompletableFuture.supplyAsync(() -> {
            // Process payment in thread pool
            return executePayment(payment);
        }, executor);
    }

    public List<PaymentResult> processBatch(List<Payment> payments) {
        List<CompletableFuture<PaymentResult>> futures = payments.stream()
            .map(this::processPayment)
            .collect(Collectors.toList());

        return futures.stream()
            .map(CompletableFuture::join)
            .collect(Collectors.toList());
    }

    @PreDestroy
    public void shutdown() {
        executor.shutdown();
        try {
            if (!executor.awaitTermination(60, TimeUnit.SECONDS)) {
                executor.shutdownNow();
            }
        } catch (InterruptedException e) {
            executor.shutdownNow();
        }
    }
}
```

### Concurrent Collections

```java
// Thread-safe cache for account balances
public class AccountBalanceCache {
    private final ConcurrentMap<String, BalanceEntry> cache = new ConcurrentHashMap<>();

    public BigDecimal getBalance(String accountId) {
        BalanceEntry entry = cache.get(accountId);
        if (entry != null && !entry.isExpired()) {
            return entry.balance;
        }
        return null;
    }

    public void updateBalance(String accountId, BigDecimal balance) {
        cache.put(accountId, new BalanceEntry(balance, Instant.now()));
    }

    // Atomic update
    public boolean compareAndSetBalance(String accountId, BigDecimal expected, BigDecimal update) {
        return cache.compute(accountId, (key, entry) -> {
            if (entry != null && entry.balance.equals(expected)) {
                return new BalanceEntry(update, Instant.now());
            }
            return entry;
        }) != null;
    }
}
```

## TypeScript Concurrency

### Async/Await

```typescript
// Concurrent account fetching with TypeScript
async function fetchAccountBalances(accountIds: string[]): Promise<Map<string, number>> {
  const results = new Map<string, number>();

  // Fetch concurrently with Promise.all
  const fetchPromises = accountIds.map(async (accountId) => {
    const balance = await db.getAccountBalance(accountId);
    results.set(accountId, balance);
  });

  await Promise.allSettled(fetchPromises);
  return results;
}

// Concurrency limit with p-limit
import pLimit from 'p-limit';

async function processPaymentsWithLimit(payments: Payment[], concurrency: number): Promise<PaymentResult[]> {
  const limit = pLimit(concurrency);

  const tasks = payments.map((payment) =>
    limit(() => executePayment(payment))
  );

  return Promise.all(tasks);
}
```

### Worker Threads for CPU-Bound Tasks

```typescript
import { Worker, isMainThread, parentPort } from 'worker_threads';

if (isMainThread) {
  // Main thread: distribute work to workers
  async function computeRiskScores(documents: string[]): Promise<number[]> {
    const workers: Worker[] = [];
    const results: number[] = [];

    // Create workers
    const chunkSize = Math.ceil(documents.length / 4);
    for (let i = 0; i < 4; i++) {
      const chunk = documents.slice(i * chunkSize, (i + 1) * chunkSize);
      const worker = new Worker(__filename, { workerData: chunk });

      worker.on('message', (score) => results.push(score));
      workers.push(worker);
    }

    // Wait for all workers
    await Promise.all(workers.map((w) => new Promise((resolve) => w.on('exit', resolve))));
    return results;
  }
} else {
  // Worker thread: compute on chunk
  const { workerData: documents } = require('worker_threads');
  for (const doc of documents) {
    const score = calculateRiskScore(doc);
    parentPort!.postMessage(score);
  }
}
```

## Banking-Specific Concurrency Patterns

### Optimistic Locking

```python
# Optimistic locking for account updates
class Account:
    def __init__(self, id: str, balance: Decimal, version: int):
        self.id = id
        self.balance = balance
        self.version = version

    def update_balance(self, amount: Decimal, expected_version: int) -> 'Account':
        """Update balance with optimistic locking."""
        if self.version != expected_version:
            raise ConcurrentModificationError(
                f'Account {self.id} was modified by another transaction. '
                f'Expected version {expected_version}, got {self.version}'
            )
        self.balance += amount
        self.version += 1
        return self

# Database query
def update_account_balance(account_id: str, amount: Decimal, version: int) -> bool:
    result = db.execute(
        """
        UPDATE accounts
        SET balance = balance + :amount, version = version + 1
        WHERE id = :id AND version = :version
        """,
        {'id': account_id, 'amount': amount, 'version': version},
    )
    return result.rowcount > 0  # True if row was updated
```

### Pessimistic Locking

```python
# Pessimistic locking for critical operations
def transfer_with_pessimistic_lock(from_id: str, to_id: str, amount: Decimal):
    """Lock accounts during transfer to prevent concurrent modifications."""
    with db.transaction():
        # Lock accounts in consistent order (prevent deadlocks)
        first_id, second_id = sorted([from_id, to_id])

        first_account = db.query(
            "SELECT * FROM accounts WHERE id = :id FOR UPDATE",
            {'id': first_id},
        )

        second_account = db.query(
            "SELECT * FROM accounts WHERE id = :id FOR UPDATE",
            {'id': second_id},
        )

        # Execute transfer while locked
        if first_id == from_id:
            first_account.balance -= amount
            second_account.balance += amount
        else:
            second_account.balance -= amount
            first_account.balance += amount

        db.save(first_account)
        db.save(second_account)
```

## Common Mistakes

1. **Shared mutable state without synchronization**: Race conditions
2. **Lock ordering inconsistency**: Deadlocks
3. **Holding locks during I/O**: Lock contention, reduced throughput
4. **Too many goroutines/threads**: Memory exhaustion, scheduler overhead
5. **Not handling context cancellation**: Goroutine leaks
6. **Using async for CPU-bound work**: Blocks event loop
7. **Fire-and-forget without error handling**: Silent failures

## Interview Questions

1. How do you prevent race conditions when two threads update the same account?
2. Design a thread-safe payment processing system that handles 10,000 payments/second.
3. What is the difference between optimistic and pessimistic locking? When to use each?
4. How do you detect and prevent deadlocks in a banking system?
5. Explain why you would use asyncio vs multiprocessing for GenAI inference.

## Cross-References

- [[distributed-systems/README.md]] - Distributed locks and consensus
- [[resilience-patterns/README.md]] - Bulkhead pattern for concurrency isolation
- [[performance/README.md]] - Concurrency for throughput optimization
- `../go/concurrency.md` - Go-specific concurrency patterns
- `../python/async-python.md` - Python asyncio patterns
