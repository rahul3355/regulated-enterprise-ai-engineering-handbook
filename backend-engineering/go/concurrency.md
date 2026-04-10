# Go Concurrency Patterns

## Goroutines

```go
// Lightweight threads managed by Go runtime
// 2KB initial stack (vs 1-2MB for OS threads)
// Scheduled multiplexed onto OS threads

// Fire and forget (dangerous - goroutine leak if context not cancelled)
go sendNotification(paymentID)

// With context (proper cancellation)
go func() {
    ctx, cancel := context.WithTimeout(context.Background(), 30*time.Second)
    defer cancel()
    sendNotification(ctx, paymentID)
}()
```

## Channels

```go
// Unbuffered channel: sender blocks until receiver is ready
ch := make(chan int)  // Synchronous communication

// Buffered channel: sender blocks only when buffer is full
ch := make(chan int, 100)  // Asynchronous, holds 100 items

// Channel directions
type Producer chan<- int  // Send-only
type Consumer <-chan int  // Receive-only

// Usage: worker pool results
func processPayments(ctx context.Context, payments []Payment) ([]Result, error) {
    results := make(chan Result, len(payments))
    errCh := make(chan error, 1)
    var wg sync.WaitGroup

    for _, p := range payments {
        wg.Add(1)
        go func(payment Payment) {
            defer wg.Done()
            result, err := processSingle(ctx, payment)
            if err != nil {
                select {
                case errCh <- err:
                default:
                }
                return
            }
            results <- result
        }(p)
    }

    // Close results when all workers done
    go func() {
        wg.Wait()
        close(results)
    }()

    // Collect results
    var allResults []Result
    for result := range results {
        allResults = append(allResults, result)
    }

    // Check for errors
    select {
    case err := <-errCh:
        return allResults, err
    default:
        return allResults, nil
    }
}
```

## Select Statement

```go
// Multiplex multiple channel operations
func worker(ctx context.Context, jobs <-chan Job, results chan<- Result) {
    for {
        select {
        case <-ctx.Done():
            // Context cancelled - exit
            return

        case job, ok := <-jobs:
            if !ok {
                // Jobs channel closed - exit
                return
            }
            result := processJob(job)
            results <- result

        case <-time.After(30 * time.Second):
            // Timeout - log and continue
            log.Println("Worker idle for 30 seconds")
        }
    }
}
```

## Sync Primitives

### Mutex

```go
import "sync"

// Thread-safe counter
type SafeCounter struct {
    mu    sync.Mutex
    value int
}

func (c *SafeCounter) Increment() {
    c.mu.Lock()
    defer c.mu.Unlock()
    c.value++
}

func (c *SafeCounter) Value() int {
    c.mu.Lock()
    defer c.mu.Unlock()
    return c.value
}

// RWMutex: multiple readers OR single writer
type AccountBalanceCache struct {
    mu      sync.RWMutex
    balances map[string]decimal.Decimal
}

func (c *AccountBalanceCache) Get(accountID string) (decimal.Decimal, bool) {
    c.mu.RLock()         // Multiple readers allowed
    defer c.mu.RUnlock()
    balance, ok := c.balances[accountID]
    return balance, ok
}

func (c *AccountBalanceCache) Set(accountID string, balance decimal.Decimal) {
    c.mu.Lock()          // Exclusive write
    defer c.mu.Unlock()
    c.balances[accountID] = balance
}
```

### WaitGroup

```go
// Wait for multiple goroutines to complete
func fetchAccountBalances(accountIDs []string) (map[string]decimal.Decimal, error) {
    var (
        mu      sync.Mutex
        wg      sync.WaitGroup
        results = make(map[string]decimal.Decimal)
        errCh   = make(chan error, len(accountIDs))
    )

    for _, accountID := range accountIDs {
        wg.Add(1)
        go func(id string) {
            defer wg.Done()

            balance, err := db.GetBalance(id)
            if err != nil {
                errCh <- fmt.Errorf("account %s: %w", id, err)
                return
            }

            mu.Lock()
            results[id] = balance
            mu.Unlock()
        }(accountID)
    }

    wg.Wait()
    close(errCh)

    // Check for errors
    var errs []error
    for err := range errCh {
        errs = append(errs, err)
    }
    if len(errs) > 0 {
        return results, fmt.Errorf("fetching balances: %w", errors.Join(errs...))
    }

    return results, nil
}
```

### Once

```go
// Execute initialization exactly once
var (
    dbInstance *sql.DB
    dbOnce     sync.Once
)

func GetDatabase() (*sql.DB, error) {
    var err error
    dbOnce.Do(func() {
        dbInstance, err = sql.Open("pgx", os.Getenv("DATABASE_URL"))
        if err == nil {
            err = dbInstance.Ping()
        }
    })
    return dbInstance, err
}
```

## Context

```go
// Context carries deadlines, cancellation signals, and request-scoped values

func ProcessPayment(ctx context.Context, payment Payment) error {
    // Check if context already cancelled
    select {
    case <-ctx.Done():
        return ctx.Err()
    default:
    }

    // Create child context with timeout
    ctx, cancel := context.WithTimeout(ctx, 30*time.Second)
    defer cancel()

    // Propagate context to database query
    row := db.QueryRowContext(ctx,
        "SELECT balance FROM accounts WHERE id = $1",
        payment.FromAccount,
    )

    // Propagate context to HTTP call
    req, _ := http.NewRequestWithContext(ctx, "POST", gatewayURL, body)
    resp, err := client.Do(req)

    // Propagate context to goroutine
    go func() {
        ctx, cancel := context.WithTimeout(context.Background(), 10*time.Second)
        defer cancel()
        sendNotification(ctx, payment)
    }()

    return nil
}
```

## Worker Pool Pattern

```go
type PaymentWorker struct {
    workerCount int
    jobQueue    chan Payment
    resultQueue chan PaymentResult
    ctx         context.Context
    cancel      context.CancelFunc
}

func NewPaymentWorker(workerCount int) *PaymentWorker {
    ctx, cancel := context.WithCancel(context.Background())
    return &PaymentWorker{
        workerCount: workerCount,
        jobQueue:    make(chan Payment, 1000),
        resultQueue: make(chan PaymentResult, 1000),
        ctx:         ctx,
        cancel:      cancel,
    }
}

func (w *PaymentWorker) Start() {
    for i := 0; i < w.workerCount; i++ {
        go w.worker(i)
    }
}

func (w *PaymentWorker) worker(id int) {
    for {
        select {
        case <-w.ctx.Done():
            log.Printf("Worker %d shutting down", id)
            return

        case payment, ok := <-w.jobQueue:
            if !ok {
                return
            }
            result := processPayment(payment)
            w.resultQueue <- result
        }
    }
}

func (w *PaymentWorker) Stop() {
    w.cancel()
}

func (w *PaymentWorker) Submit(payment Payment) error {
    select {
    case w.jobQueue <- payment:
        return nil
    case <-time.After(5 * time.Second):
        return fmt.Errorf("job queue full")
    }
}
```

## Rate Limiting

```go
// Token bucket rate limiter
type RateLimiter struct {
    tokens   chan struct{}
    interval time.Duration
}

func NewRateLimiter(maxTokens int, interval time.Duration) *RateLimiter {
    rl := &RateLimiter{
        tokens:   make(chan struct{}, maxTokens),
        interval: interval,
    }

    // Fill tokens at regular interval
    go func() {
        ticker := time.NewTicker(interval)
        for range ticker.C {
            select {
            case rl.tokens <- struct{}{}:
            default:
                // Bucket full
            }
        }
    }()

    return rl
}

func (rl *RateLimiter) Allow() bool {
    select {
    case <-rl.tokens:
        return true
    default:
        return false
    }
}

// Usage in middleware
limiter := NewRateLimiter(100, time.Second)

func rateLimitMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        if !limiter.Allow() {
            http.Error(w, "rate limited", http.StatusTooManyRequests)
            return
        }
        next.ServeHTTP(w, r)
    })
}
```

## Common Mistakes

1. **Goroutine leaks**: Not using context cancellation, goroutines run forever
2. **Unbuffered channel deadlock**: Sending to channel with no receiver
3. **Race conditions**: Accessing shared data without mutex
4. **Closing closed channel**: `panic: close of closed channel`
5. **Sending on closed channel**: `panic: send on closed channel`
6. **Ignoring context cancellation**: Not checking `ctx.Done()` in loops
7. **Mutex copy**: Passing struct containing mutex by value

## Interview Questions

1. How do you process 10,000 payments concurrently with a limit of 50 workers?
2. Design a worker pool that processes jobs from a Kafka topic.
3. How do you detect and prevent goroutine leaks?
4. What is the difference between buffered and unbuffered channels?
5. How do you implement a rate limiter using Go primitives?

## Cross-References

- `./service-structure.md` - Goroutines in HTTP handlers
- `./error-handling.md` - Error handling in concurrent code
- `./testing.md` - Testing concurrent code
- `../concurrency/README.md` - Go vs Python vs Java concurrency
